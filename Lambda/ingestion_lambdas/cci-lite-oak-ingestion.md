```python
import os
import json
import boto3
from datetime import datetime, timedelta
import urllib3
from collections import defaultdict

http = urllib3.PoolManager()
s3 = boto3.client("s3")
secrets = boto3.client("secretsmanager")

INPUT_BUCKET = os.environ["INPUT_BUCKET"]
SYNC_LOOKBACK_MINUTES = int(os.environ.get("SYNC_LOOKBACK_MINUTES", "15"))

# =====================================================================
# 🔵 STEP 1 — Multi-tenant discovery from Secrets Manager
# =====================================================================
def discover_tenants():
    prefix = "cci-lite/oak/"

    resp = secrets.list_secrets()
    tenants = []

    for sec in resp.get("SecretList", []):
        name = sec["Name"]
        if name.startswith(prefix):
            tenant_id = name.replace(prefix, "")
            tenants.append(tenant_id)

    return tenants



def load_credentials(tenant_id):
    prefix = f"cci-lite/oak/{tenant_id}"

    resp = secrets.list_secrets()
    for sec in resp.get("SecretList", []):
        if sec["Name"] == prefix:
            raw = secrets.get_secret_value(SecretId=prefix)
            return json.loads(raw["SecretString"])

    raise RuntimeError(f"No secret found for tenant {tenant_id}")


# =====================================================================
# 🔵 STEP 2 — Existing OAK / ClarifyGo logic (UNCHANGED)
# =====================================================================
def get_oak_token(cfg):
    url = cfg["base_url"] + cfg["token_path"]

    fields = {
        "grant_type": "password",
        "username": cfg["username"],
        "password": cfg["password"],
        "client_id": cfg["client_id"],
        "client_secret": cfg["client_secret"]
    }

    headers = {"Accept": "application/json"}

    resp = http.request(
        "POST",
        url,
        fields=fields,
        encode_multipart=False,
        headers=headers,
    )

    if resp.status != 200:
        raise RuntimeError(f"Token request failed: {resp.status} {resp.data}")

    data = json.loads(resp.data.decode("utf-8"))
    if "access_token" not in data:
        raise RuntimeError(f"No access_token in token response: {data}")

    return data["access_token"]


def fetch_recordings(cfg, token, start_iso, end_iso):
    url = cfg["base_url"] + cfg["recordings_path"].format(start=start_iso, end=end_iso)

    headers = {
        "Authorization": f"Bearer {token}",
        "Accept": "application/json",
    }

    resp = http.request("GET", url, headers=headers)
    if resp.status != 200:
        raise RuntimeError(f"Historic recordings failed: {resp.status} {resp.data}")

    data = json.loads(resp.data.decode("utf-8"))
    return data.get("searchResults", []), data.get("totalResults", 0)


def download_audio(cfg, token, call_id):
    url = cfg["base_url"] + cfg["recording_download_path"].format(id=call_id)

    headers = {
        "Authorization": f"Bearer {token}",
        "Accept": "audio/mpeg,*/*",
    }

    resp = http.request("GET", url, headers=headers, redirect=True)

    if resp.status != 200:
        raise RuntimeError(f"Audio download failed: {resp.status} {resp.data}")

    return resp.data


# =====================================================================
# 🔵 STEP 3 — Run ingestion for a single tenant (ClarifyGo logic preserved)
# =====================================================================
def run_ingestion_for_tenant(tenant_id, cfg):
    print(f"[Ingest][{tenant_id}] Starting ClarifyGo → S3 sync")

    token = get_oak_token(cfg)

    now = datetime.utcnow()
    start = now - timedelta(minutes=SYNC_LOOKBACK_MINUTES)
    start_iso = start.isoformat(timespec="seconds") + "Z"
    end_iso = now.isoformat(timespec="seconds") + "Z"

    raw_records, total = fetch_recordings(cfg, token, start_iso, end_iso)
    print(f"[Ingest][{tenant_id}] Fetched {len(raw_records)} recordings ({start_iso} → {end_iso})")

    groups = defaultdict(list)
    for item in raw_records:
        rec = item.get("recording", item)
        gid = rec.get("recordingGroupingId", rec.get("id"))
        groups[gid].append(rec)

    synced = 0
    skipped_existing = 0
    failed = 0

    for gid, rec_list in groups.items():

        # ------------------------
        # MEETING SKIP LOGIC
        # ------------------------
        all_internal = all(
            str(r.get("callingParty", "")).lower().endswith("@conversant.technology")
            and str(r.get("calledParty", "")).lower().endswith("@conversant.technology")
            for r in rec_list
        )

        looks_like_meeting = any(
            "meeting" in str(r.get("calledParty", "")).lower()
            or "meeting" in str(r.get("callingParty", "")).lower()
            for r in rec_list
        )

        if all_internal and looks_like_meeting:
            print(f"[Ingest][{tenant_id}] ⏩ Skipping internal meeting group {gid}")
            continue

        agent_legs = [
            r for r in rec_list
            if any(
                ep.get("number", "").lower().endswith("@conversant.technology")
                for ep in r.get("pbxAccountEndpoints", [])
            )
        ]

        if not agent_legs:
            print(f"[Ingest][{tenant_id}] ⏩ No agent leg for group {gid}, skipping")
            continue

        # ------------------------
        # OPTION B — INITIATOR WINS
        # ------------------------
        initiator_leg = None
        for r in agent_legs:
            caller = str(r.get("callingParty", "")).lower()
            if caller.endswith("@conversant.technology"):
                initiator_leg = r
                break

        def duration(rec):
            try:
                s = rec.get("mediaStartedTime")
                e = rec.get("mediaCompletedTime")
                return (datetime.fromisoformat(e.replace("Z", "")) -
                        datetime.fromisoformat(s.replace("Z", ""))).total_seconds()
            except:
                return 0

        if initiator_leg:
            best_leg = initiator_leg
        else:
            best_leg = max(agent_legs, key=duration)

        call_id = best_leg.get("id")

        audio_key = f"{tenant_id}/audio/{call_id}.mp3"
        metadata_key = f"{tenant_id}/meta/{call_id}.json"

        try:
            s3.head_object(Bucket=INPUT_BUCKET, Key=audio_key)
            print(f"[Ingest][{tenant_id}] Exists → {call_id}, skipping")
            skipped_existing += 1
            continue
        except:
            pass

        # ------------------------
        # Download audio FIRST
        # ------------------------
        try:
            audio_bytes = download_audio(cfg, token, call_id)
            s3.put_object(
                Bucket=INPUT_BUCKET,
                Key=audio_key,
                Body=audio_bytes,
                ServerSideEncryption="aws:kms",
            )
        except Exception as e:
            print(f"[Ingest][{tenant_id}] ⚠ Audio failed for {call_id}: {e}")
            failed += 1
            continue

        meta = {"raw": best_leg}
        s3.put_object(
            Bucket=INPUT_BUCKET,
            Key=metadata_key,
            Body=json.dumps(meta),
            ServerSideEncryption="aws:kms",
        )

        print(f"[Ingest][{tenant_id}] ✅ Synced {call_id}")
        synced += 1

    print(f"[Ingest][{tenant_id}] Done. synced={synced}, skipped={skipped_existing}, failed={failed}")

    return {
        "tenant": tenant_id,
        "synced": synced,
        "skipped_existing": skipped_existing,
        "failed": failed,
        "total_in_window": total,
        "window_start": start_iso,
        "window_end": end_iso,
    }


# =====================================================================
# 🔵 STEP 4 — Multi-tenant Lambda handler
# =====================================================================
def lambda_handler(event, context):
    print("[Ingest] === OAK Multi-Tenant Ingestion Start ===")

    tenants = discover_tenants()
    print(f"[Ingest] Tenants discovered: {tenants}")

    for tenant_id in tenants:
        try:
            creds = load_credentials(tenant_id)
            creds["base_url"] = creds["base_url"].rstrip("/")
            run_ingestion_for_tenant(tenant_id, creds)
        except Exception as e:
            print(f"[Ingest] ERROR processing tenant {tenant_id}: {e}")

    print("[Ingest] === OAK Multi-Tenant Ingestion Complete ===")
```
