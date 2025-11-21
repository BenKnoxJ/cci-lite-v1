```python
import json, boto3, os, urllib.parse
from datetime import datetime

s3 = boto3.client("s3")
comprehend = boto3.client("comprehend", region_name="eu-central-1")
bedrock = boto3.client("bedrock-runtime", region_name="eu-central-1")

RESULTS_BUCKET = os.environ["RESULTS_BUCKET"]
CONFIG_BUCKET  = os.environ["CONFIG_BUCKET"]
INPUT_BUCKET   = os.environ["INPUT_BUCKET"]
KMS_KEY_ARN    = os.environ["KMS_KEY_ARN"]
BEDROCK_MODEL  = os.environ.get("BEDROCK_MODEL", "anthropic.claude-3-haiku-20240307")
DEFAULT_LANG   = os.environ.get("DEFAULT_LANGUAGE", "en-GB")

# -------------------------------------------------------
# SAFE LOAD JSON
# -------------------------------------------------------
def safe_json_load(bucket, key):
    try:
        obj = s3.get_object(Bucket=bucket, Key=key)
        return json.loads(obj["Body"].read())
    except Exception as e:
        print(f"[Analyzer] ⚠ Failed to load {bucket}/{key}: {e}")
        return {}

# -------------------------------------------------------
# STRICT JSON EXTRACTOR
# -------------------------------------------------------
def extract_json_from_text(text):
    text = text.replace("```json", "").replace("```", "")
    start = text.find("{")
    if start == -1:
        return {}
    depth = 0
    for i in range(start, len(text)):
        if text[i] == "{":
            depth += 1
        elif text[i] == "}":
            depth -= 1
            if depth == 0:
                try:
                    return json.loads(text[start:i+1])
                except:
                    return {}
    return {}

# -------------------------------------------------------
# ENTRYPOINT
# -------------------------------------------------------
def lambda_handler(event, context):
    print(f"[Analyzer] Received {len(event.get('Records', []))} messages")

    for record in event.get("Records", []):
        body = json.loads(record["body"])
        for s3rec in body.get("Records", []):
            bucket = s3rec["s3"]["bucket"]["name"]
            key    = urllib.parse.unquote_plus(s3rec["s3"]["object"]["key"])
            process_call(bucket, key)

    return {"status": "ok"}


# -------------------------------------------------------
# MAIN PROCESSOR
# -------------------------------------------------------
def process_call(bucket, key):
    print(f"[Analyzer] Processing enriched transcript: {key}")

    tenant_id = key.split("/")[0]
    call_id   = os.path.splitext(os.path.basename(key))[0]

    enriched = safe_json_load(bucket, key)
    transcript = enriched.get("transcript", [])
    language   = (enriched.get("language") or DEFAULT_LANG).split("-")[0]

    # -------------------------------------------------------
    # META
    # -------------------------------------------------------
    meta_key = f"{tenant_id}/meta/{call_id}.json"
    meta_obj = safe_json_load(INPUT_BUCKET, meta_key)
    raw_meta = meta_obj.get("raw", meta_obj)

    calling_party = raw_meta.get("callingParty")
    called_party  = raw_meta.get("calledParty")

    # -------------------------------------------------------
    # DIRECTION
    # -------------------------------------------------------
    caller_int = str(calling_party or "").endswith("@conversant.technology")
    callee_int = str(called_party  or "").endswith("@conversant.technology")

    if caller_int and not callee_int: direction="outbound"
    elif not caller_int and callee_int: direction="inbound"
    elif caller_int and callee_int: direction="internal"
    else: direction="unknown"

    # -------------------------------------------------------
    # DURATION
    # -------------------------------------------------------
    start_ts = raw_meta.get("mediaStartedTime")
    end_ts   = raw_meta.get("mediaCompletedTime")
    duration_seconds = None

    try:
        if start_ts and end_ts:
            s = datetime.fromisoformat(start_ts.replace("Z",""))
            e = datetime.fromisoformat(end_ts.replace("Z",""))
            duration_seconds = (e-s).total_seconds()
    except:
        duration_seconds = None

    # -------------------------------------------------------
    # SENTIMENT
    # -------------------------------------------------------
    text_blob = " ".join(u.get("text","") for u in transcript)[:4800]
    try:
        if text_blob.strip():
            sent = comprehend.detect_sentiment(Text=text_blob, LanguageCode=language)
            sentiment = {
                "overall": sent["Sentiment"].lower(),
                "positive": round(sent["SentimentScore"]["Positive"],2),
                "neutral":  round(sent["SentimentScore"]["Neutral"],2),
                "negative": round(sent["SentimentScore"]["Negative"],2),
                "confidence": round(max(sent["SentimentScore"].values()),2)
            }
        else:
            sentiment = {"overall":"neutral","positive":0,"neutral":1,"negative":0,"confidence":0}
    except:
        sentiment = {"overall":"neutral","positive":0,"neutral":1,"negative":0,"confidence":0}

    # -------------------------------------------------------
    # LOAD CONFIGS
    # -------------------------------------------------------
    qa_cfg = safe_json_load(CONFIG_BUCKET, f"{tenant_id}/qa-config.json")
    ai_cfg = safe_json_load(CONFIG_BUCKET, f"{tenant_id}/ai-config.json")

    insight_schema = ai_cfg["json_output_schema"]
    qa_rules       = qa_cfg["qa_rules"]
    qa_schema      = qa_cfg["qa_output_schema"]
    qa_guidance    = qa_cfg["prompt_guidance"]
    instructions   = ai_cfg["instructions"]
    business_ctx   = ai_cfg.get("business_context","")

    # -------------------------------------------------------
    # BUILD TRANSCRIPT TEXT
    # -------------------------------------------------------
    transcript_text = "\n".join(
        f"{u.get('speaker','')}: {u.get('text','')}" for u in transcript
    )[:24000]

    # -------------------------------------------------------
    # 🔥 FIX: NO HALLUCINATIONS ON EMPTY TRANSCRIPT
    # -------------------------------------------------------
    if not transcript_text.strip():
        print("[Analyzer] ⚠ Empty transcript detected — skipping Bedrock.")

        qa_eval = normalize_qa({}, qa_cfg)
        ai_insights = normalize_ai({}, insight_schema)

        return write_final_outputs(
            tenant_id, call_id, calling_party, called_party,
            start_ts, end_ts, duration_seconds, direction,
            transcript, sentiment, qa_eval, ai_insights, raw_meta
        )

    # -------------------------------------------------------
    # BUILD PROMPT
    # -------------------------------------------------------
    prompt = f"""
You are Conversant CCI Lite. 
You MUST return ONLY valid JSON matching the exact schema shown below.
Never omit fields. Never add fields. 
If information is missing in the transcript: output null or empty list accordingly.

BUSINESS CONTEXT:
{business_ctx}

AI INSIGHT INSTRUCTIONS:
{json.dumps(instructions, indent=2)}

QA GUIDANCE:
{qa_guidance}

QA RULESET:
{json.dumps(qa_rules, indent=2)}

TRANSCRIPT:
{transcript_text}

You MUST reply with EXACTLY this JSON shape:

{{
  "qa_evaluation": {json.dumps(qa_schema, indent=2)},
  "ai_insights": {json.dumps(insight_schema, indent=2)}
}}
"""

    # -------------------------------------------------------
    # CALL BEDROCK
    # -------------------------------------------------------
    try:
        resp = bedrock.invoke_model(
            modelId=BEDROCK_MODEL,
            body=json.dumps({
                "anthropic_version": "bedrock-2023-05-31",
                "max_tokens": ai_cfg.get("max_tokens",2500),
                "temperature": ai_cfg.get("temperature",0.2),
                "messages":[{"role":"user","content":[{"type":"text","text":prompt}]}]
            })
        )
        raw_text = json.loads(resp["body"].read())["content"][0]["text"]
        bedrock_data = extract_json_from_text(raw_text)
    except Exception as e:
        print(f"[Analyzer] ⚠ Bedrock error: {e}")
        bedrock_data = {}

    # -------------------------------------------------------
    # NORMALISE OUTPUTS
    # -------------------------------------------------------
    qa_eval     = normalize_qa(bedrock_data.get("qa_evaluation"), qa_cfg)
    ai_insights = normalize_ai(bedrock_data.get("ai_insights"), insight_schema)

    # -------------------------------------------------------
    # AUTO-INJECT QA RECOMMENDATIONS
    # -------------------------------------------------------
    auto_rec = []
    for r in qa_eval["rule_results"]:
        if not r["pass"] and r.get("coach_note"):
            auto_rec.append(f"Rule '{r['rule_name']}': {r['coach_note']}")

    llm_rec = ai_insights.get("recommendations") or []
    if isinstance(llm_rec, str):
        llm_rec = [llm_rec]

    ai_insights["recommendations"] = list({*(auto_rec), *(llm_rec)}) or None

    # -------------------------------------------------------
    # WRITE OUTPUTS
    # -------------------------------------------------------
    return write_final_outputs(
        tenant_id, call_id, calling_party, called_party,
        start_ts, end_ts, duration_seconds, direction,
        transcript, sentiment, qa_eval, ai_insights, raw_meta
    )


# -------------------------------------------------------
# FINAL OUTPUT WRITER (shared)
# -------------------------------------------------------
def write_final_outputs(tenant_id, call_id, calling_party, called_party, start_ts, end_ts,
                        duration_seconds, direction, transcript, sentiment,
                        qa_eval, ai_insights, raw_meta):

    processing_ts = datetime.utcnow().isoformat()+"Z"

    final_v1 = {
        "schema_version":"cci-lite-v1.0",
        "call":{
            "tenant_id":tenant_id,
            "call_id":call_id,
            "language":DEFAULT_LANG,
            "direction":direction,
            "from_number":calling_party,
            "to_number":called_party,
            "start_timestamp":start_ts,
            "end_timestamp":end_ts,
            "duration_seconds":duration_seconds,
            "audio_uri":f"s3://{INPUT_BUCKET}/{tenant_id}/audio/{call_id}.mp3",
            "transcript_uri":f"s3://{RESULTS_BUCKET}/{tenant_id}/enriched/{call_id}.json",
            "metadata_uri":f"s3://{INPUT_BUCKET}/{tenant_id}/meta/{call_id}.json",
            "processing_timestamp":processing_ts
        },
        "sentiment":sentiment,
        "qa_evaluation":qa_eval,
        "ai_insights":ai_insights,
        "raw":{"metadata":raw_meta}
    }

    flat_calls = [{
        **final_v1["call"],
        **sentiment,
        **ai_insights,
        "overall_score":qa_eval["overall_score"],
        "grade":qa_eval["grade"]
    }]

    flat_qa = [{
        "tenant_id":tenant_id,
        "call_id":call_id,
        **r,
        "overall_score":qa_eval["overall_score"],
        "grade":qa_eval["grade"]
    } for r in qa_eval["rule_results"]]

    outputs = [
        (f"{tenant_id}/final/v1.0/{call_id}.json", final_v1),
        (f"{tenant_id}/final/v2.0-flat/calls/{call_id}.jsonl", flat_calls),
        (f"{tenant_id}/final/v2.0-flat/qa/{call_id}.jsonl", flat_qa),
    ]

    for out_key, obj in outputs:
        if isinstance(obj, list):
            body = "\n".join(json.dumps(o, ensure_ascii=False) for o in obj) + "\n"
        else:
            body = json.dumps(obj, ensure_ascii=False) + "\n"

        s3.put_object(
            Bucket=RESULTS_BUCKET,
            Key=out_key,
            Body=body.encode("utf-8"),
            ServerSideEncryption="aws:kms",
            SSEKMSKeyId=KMS_KEY_ARN
        )

    print(f"[Analyzer] ✔ Completed call (clean) {call_id}")


# -------------------------------------------------------
# NORMALISE QA
# -------------------------------------------------------
def normalize_qa(qa_eval, qa_cfg):
    if not isinstance(qa_eval, dict):
        qa_eval = {}

    rules_cfg = qa_cfg.get("qa_rules", [])
    rule_results = qa_eval.get("rule_results")

    if not isinstance(rule_results, list) or len(rule_results)==0:
        rule_results = [{
            "rule_name":r["rule_name"],
            "category":r.get("category",""),
            "score":0.0,
            "pass":False,
            "weight":r.get("weight",1.0),
            "evidence":None,
            "coach_note":None
        } for r in rules_cfg]

    weights = [x["weight"] for x in rule_results]
    total_w = sum(weights) or 1
    overall = round(sum(x["score"]*x["weight"] for x in rule_results)/total_w,2)

    if overall>=90: grade="A"
    elif overall>=80: grade="B"
    elif overall>=70: grade="C"
    elif overall>=60: grade="D"
    else: grade="E"

    return {
        "rule_results":rule_results,
        "overall_score":overall,
        "grade":grade
    }

# -------------------------------------------------------
# NORMALISE AI
# -------------------------------------------------------
def normalize_ai(ai, schema):
    if not isinstance(ai, dict):
        ai = {}

    out = {}
    for k in schema.keys():
        v = ai.get(k)
        if isinstance(v,str):
            out[k] = [v]
        else:
            out[k] = v if v is not None else None

    return out
```
