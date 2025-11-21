```python
import json
import boto3
import os
import urllib.parse
from datetime import datetime

s3 = boto3.client("s3")

RESULTS_BUCKET = os.environ["RESULTS_BUCKET"]
KMS_KEY_ARN = os.environ["KMS_KEY_ARN"]

def lambda_handler(event, context):
    try:
        # -----------------------------
        # Extract S3 event
        # -----------------------------
        record = event["Records"][0]
        bucket = record["s3"]["bucket"]["name"]
        key = urllib.parse.unquote_plus(record["s3"]["object"]["key"])

        # Expect EXACT format:
        # demo-tenant/raw/{call_id}.json
        parts = key.split("/")
        if len(parts) != 3 or parts[1] != "raw":
            print(f"[ResultHandler] Ignoring non-raw key: {key}")
            return {"status": "ignored"}

        tenant_id = parts[0]
        call_id = parts[2].replace(".json", "")

        print(f"[ResultHandler] Processing Transcribe output: {key}")

        # -----------------------------
        # Load raw Transcribe JSON
        # -----------------------------
        raw_obj = s3.get_object(Bucket=bucket, Key=key)
        transcribe_data = json.loads(raw_obj["Body"].read())

        # -----------------------------
        # Build utterances list
        # -----------------------------
        utterances = parse_transcribe_output(transcribe_data)

        # -----------------------------
        # Build lightweight metrics
        # (same as before, no regressions)
        # -----------------------------
        metrics = compute_metrics(utterances)

        # -----------------------------
        # Construct enriched object
        # -----------------------------
        enriched = {
            "tenant_id": tenant_id,
            "call_id": call_id,
            "language": transcribe_data.get("results", {}).get("language_code", "en-US"),
            "transcript": utterances,
            "analysis_features": metrics,
        }

        # -----------------------------
        # Write enriched/{call_id}.json
        # -----------------------------
        out_key = f"{tenant_id}/enriched/{call_id}.json"

        s3.put_object(
            Bucket=RESULTS_BUCKET,
            Key=out_key,
            Body=json.dumps(enriched, ensure_ascii=False).encode("utf-8"),
            ServerSideEncryption="aws:kms",
            SSEKMSKeyId=KMS_KEY_ARN,
        )

        print(f"[ResultHandler] ✅ Wrote enriched file: s3://{RESULTS_BUCKET}/{out_key}")

        return {"status": "success", "enriched_key": out_key}

    except Exception as e:
        print(f"[ResultHandler] ❌ Error: {e}")
        raise


# ----------------------------------------------------------------------
# PARSE TRANSCRIBE UTTERANCES
# ----------------------------------------------------------------------

def parse_transcribe_output(data):
    """Returns list of utterances (speaker, start_time, end_time, text)."""

    results = data.get("results") or {}
    items = results.get("items", [])
    transcripts = results.get("transcripts", [])
    speaker_labels = results.get("speaker_labels") or {}
    segments = speaker_labels.get("segments", [])
    channels = results.get("channel_labels", {}).get("channels", [])

    utterances = []

    # Speaker-labeled mode (ClarifyGo mono)
    if segments:
        for seg in segments:
            start = float(seg.get("start_time", 0))
            end = float(seg.get("end_time", 0))
            spk = seg.get("speaker_label", "unknown").lower()

            words = []
            for it in items:
                if it.get("type") != "pronunciation":
                    continue
                st = float(it.get("start_time", 0))
                et = float(it.get("end_time", 0))
                if st >= start and et <= end:
                    words.append(it["alternatives"][0]["content"])

            text = " ".join(words).strip()
            if not text:
                continue

            speaker = "agent" if "spk_0" in spk else "customer"

            utterances.append({
                "speaker": speaker,
                "start_time": start,
                "end_time": end,
                "text": text,
            })

        return utterances

    # Channel mode fallback
    if channels:
        for ch in channels:
            speaker = "agent" if ch.get("channel_label") in ("0", "ch_0") else "customer"
            for alt in ch.get("alternatives", []):
                t = alt.get("transcript", "").strip()
                if t:
                    utterances.append({
                        "speaker": speaker,
                        "start_time": 0.0,
                        "end_time": 0.0,
                        "text": t,
                    })
        return utterances

    # Single transcript fallback
    for t in transcripts:
        tx = t.get("transcript", "").strip()
        if tx:
            utterances.append({
                "speaker": "unknown",
                "start_time": 0.0,
                "end_time": 0.0,
                "text": tx,
            })

    return utterances


# ----------------------------------------------------------------------
# METRICS (unchanged, safe)
# ----------------------------------------------------------------------

def compute_metrics(utterances):
    if not utterances:
        return {"utterance_count": 0, "word_count": 0, "processing_timestamp": datetime.utcnow().isoformat()}

    word_count = sum(len(u["text"].split()) for u in utterances)

    # Duration: infer from last end_time
    end_times = [u["end_time"] for u in utterances if u.get("end_time", 0) > 0]
    if end_times:
        total_dur = max(end_times)
    else:
        total_dur = len(utterances) * 3.0

    return {
        "utterance_count": len(utterances),
        "word_count": word_count,
        "total_duration": total_dur,
        "processing_timestamp": datetime.utcnow().isoformat() + "Z",
    }
```
