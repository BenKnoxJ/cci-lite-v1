```python
import json
import boto3
import os
import urllib.parse
import re
from datetime import datetime

REGION = "eu-central-1"

s3 = boto3.client("s3")
transcribe = boto3.client("transcribe", region_name=REGION)

RESULTS_BUCKET = os.environ["RESULTS_BUCKET"]
KMS_KEY_ARN    = os.environ["KMS_KEY_ARN"]
LANG_CODE      = os.environ.get("LANG_CODE", "en-GB")


def lambda_handler(event, context):
    try:
        rec = event["Records"][0]
        bucket = rec["s3"]["bucket"]["name"]
        key = urllib.parse.unquote_plus(rec["s3"]["object"]["key"])

        print(f"[Init] Received: {bucket}/{key}")

        # Expect exact structure:
        # <tenant_id>/audio/<UUID>.mp3
        parts = key.split("/")
        if len(parts) != 3 or parts[1] != "audio":
            print(f"[Init] Skipping non-audio key: {key}")
            return {"status": "ignored"}

        tenant_id = parts[0]
        filename  = parts[-1]
        call_id, ext = os.path.splitext(filename)
        ext = ext.lstrip(".").lower()

        if ext not in ("mp3", "wav", "mp4", "m4a", "ogg"):
            raise ValueError(f"[Init] Unsupported audio format: {ext}")

        # Safe call_id
        safe_call_id = re.sub(r"[^0-9A-Za-z._-]", "_", call_id)

        # Unique job name
        timestamp = datetime.utcnow().strftime("%Y%m%d%H%M%S")
        job_name = f"{tenant_id}-{safe_call_id}-{timestamp}"

        # Prevent duplicate jobs
        existing = transcribe.list_transcription_jobs(
            JobNameContains=f"{tenant_id}-{safe_call_id}"
        )

        for j in existing.get("TranscriptionJobSummaries", []):
            if j["TranscriptionJobStatus"] in ("QUEUED", "IN_PROGRESS"):
                print(f"[Init] Skipping duplicate job: {j['TranscriptionJobName']}")
                return {
                    "status": "duplicate_skipped",
                    "job_name": j["TranscriptionJobName"]
                }

        media_uri = f"s3://{bucket}/{key}"

        # ❗ FIXED: MUST BE A FLAT FILE, NOT A FOLDER
        output_key = f"{tenant_id}/raw/{safe_call_id}.json"

        # Sanity check audio size
        head = s3.head_object(Bucket=bucket, Key=key)
        size = head["ContentLength"]
        if size < 5000:
            raise ValueError(f"[Init] Audio file too small: {size} bytes")

        print(f"[Init] Starting Transcribe job: {job_name}")
        print(f"[Init] Media URI: {media_uri}")

        transcribe.start_transcription_job(
            TranscriptionJobName=job_name,
            Media={"MediaFileUri": media_uri},
            MediaFormat=ext,
            LanguageCode=LANG_CODE,
            OutputBucketName=RESULTS_BUCKET,
            OutputKey=output_key,                   # <── FIX APPLIED
            OutputEncryptionKMSKeyId=KMS_KEY_ARN,
            Settings={
                "ShowSpeakerLabels": True,
                "MaxSpeakerLabels": 2,
                "ChannelIdentification": False
            },
            Tags=[
                {"Key": "tenant_id", "Value": tenant_id},
                {"Key": "call_id", "Value": safe_call_id},
                {"Key": "origin_bucket", "Value": bucket},
            ]
        )

        print(f"[Init] SUCCESS: Transcribe job created")
        return {
            "status": "started",
            "tenant_id": tenant_id,
            "call_id": safe_call_id,
            "job_name": job_name
        }

    except Exception as e:
        print(f"[Init] ERROR: {e}")
        raise
```
