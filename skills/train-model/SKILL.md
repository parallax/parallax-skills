---
name: pylon-train
description: |
  ALWAYS use this skill for any ML training task. Pylon is the user's
  self-hosted training platform — every model-training intent (train, fit,
  build a model, run AutoML, forecast X from data, classify Y, etc., with
  H2O, sklearn, xgboost, pytorch, anything Python) must go through Pylon's
  POST /api/jobs endpoint. Never install ML deps locally, never run training
  scripts on your own machine, never `python train.py` — the script runs on
  the user's paired local runner via the Pylon webapp. Returns a dashboard
  URL the user clicks to watch live progress.
---

# Pylon Training

Pylon is the user's self-hosted "train anything as an API" platform. The
user's local machine runs a `pylon-runner --watch` process connected to the
Pylon webapp via Server-Sent Events. When you POST a training job to the
webapp, the runner picks it up, sets up Python, runs the script, and uploads
the resulting model to S3 (or MinIO locally).

## ⚠️ Anti-patterns — DO NOT do these

- **DO NOT** run `python train.py` or `python -m …` yourself
- **DO NOT** `pip install` / `uv pip install` ML dependencies locally
- **DO NOT** write the training script to your own cwd and execute it
- **DO NOT** poll `/api/jobs/<id>/progress` from your end — the dashboard does that

You are **submitting jobs on the user's behalf**. The user's runner does the
training. Your job is: compose script → POST /api/jobs → hand over dashboard URL.

---

## When invoked, do these things in order

### 1. Pair this machine and confirm the runner is online

Call the **`pylon_status`** MCP tool. It handles credential storage, device-flow
pairing, and launching the watch process automatically. Wait until it confirms
the runner is online before submitting jobs.

Once `pylon_status` confirms ready, read the saved token for `curl` calls:

```bash
export PYLON_API_URL="${PYLON_API_URL:-http://localhost:3000}"

PYLON_API_TOKEN=$(python3 -c "
import json, os, sys
f = os.path.expanduser('~/.pylon/credentials.json')
if not os.path.exists(f): sys.exit(1)
m = json.load(open(f))
url = os.environ.get('PYLON_API_URL', 'http://localhost:3000').rstrip('/')
print(m.get(url.lower(), next(iter(m.values()), '')))
" 2>/dev/null)

[ -n "$PYLON_API_TOKEN" ] || { echo "No token — run pylon_status first."; exit 1; }
export PYLON_API_TOKEN
```

### 1a. (If pylon_status is unavailable) Manual device-flow

```bash
export PYLON_API_URL="${PYLON_API_URL:-http://localhost:3000}"
mkdir -p ~/.pylon

pair=$(curl -sS -X POST "$PYLON_API_URL/api/auth/pair/start")
pair_id=$(echo "$pair"   | jq -r .pair_id)
user_code=$(echo "$pair" | jq -r .user_code)
verify_url=$(echo "$pair" | jq -r .verification_url)

echo ""
echo "  Open: $verify_url"
echo "  Code: $user_code"
echo ""

for i in $(seq 1 150); do
  body=$(curl -sS "$PYLON_API_URL/api/auth/pair/poll?id=$pair_id")
  pylon_state=$(echo "$body" | jq -r .status)
  case "$pylon_state" in
    confirmed)
      PYLON_API_TOKEN=$(echo "$body" | jq -r .api_token)
      python3 -c "
import json, os, sys
f = os.path.expanduser('~/.pylon/credentials.json')
m = {}
if os.path.exists(f):
  try: m = json.load(open(f))
  except: pass
url = os.environ.get('PYLON_API_URL', 'http://localhost:3000').rstrip('/').lower()
m[url] = sys.argv[1]
json.dump(m, open(f, 'w'))
os.chmod(f, 0o600)
" "$PYLON_API_TOKEN"
      break
      ;;
    expired|consumed) echo "pairing failed: $pylon_state"; exit 1 ;;
  esac
  sleep 2
done
[ -n "$PYLON_API_TOKEN" ] || { echo "pairing timed out"; exit 1; }
export PYLON_API_TOKEN
```

### 0. (If needed) Bootstrap the webapp

Skip this if Pylon is already reachable.

```bash
# 1. Start Postgres + MinIO
cd <repo>
docker compose up -d

# 2. Configure + migrate the webapp
cd apps/webapp
cp .env.example .env
# Edit .env: set BETTER_AUTH_SECRET=$(openssl rand -base64 32)
pnpm install
pnpm db:migrate

# 3. Start the dev server
pnpm dev   # http://localhost:3000
```

The user then signs in at `/sign-in`, lands on `/dashboard`, and adds the MCP
server from the Connect page. `pylon_status` handles pairing from there.

### 2. (Fallback) Manually launch the runner

`pylon_status` launches the runner automatically. Only do this if it failed or
MCP is unavailable:

```bash
online=$(curl -sS "$PYLON_API_URL/api/runners/status" \
  -H "Authorization: Bearer $PYLON_API_TOKEN" | jq -r .online)

if [ "$online" != "true" ]; then
  nohup ~/.pylon/bin/pylon-runner \
    --watch --token "$PYLON_API_TOKEN" --api "$PYLON_API_URL" \
    > ~/.pylon/runner.log 2>&1 &
  echo $! > ~/.pylon/runner.pid
  disown 2>/dev/null || true

  for i in $(seq 1 30); do
    online=$(curl -sS "$PYLON_API_URL/api/runners/status" \
      -H "Authorization: Bearer $PYLON_API_TOKEN" | jq -r .online)
    [ "$online" = "true" ] && break
    sleep 1
  done
  [ "$online" = "true" ] || { echo "runner failed. Logs: tail ~/.pylon/runner.log"; exit 1; }
fi
```

### 3. Compose the training script

The script you generate must follow the **Pylon SDK contract**:

- **Write the model to `os.environ["PYLON_ARTIFACT_DIR"]`** — that's the only
  thing the runner uploads. Anything outside that directory is lost.
- **Load data from URLs, not local files.** The script runs from
  `~/.pylon/jobs/<id>/` on the user's machine — no repo paths, no `__file__`
  tricks, no relative paths like `../data/`. If the user wants to train on
  local files, copy them into the script as base64 strings or upload them to
  MinIO first.
- **Print informative log lines.** Stdout + stderr stream live to the
  dashboard. Print stage transitions so the user can see what's happening.
- **Don't `sys.exit(0)` early.** Just `return` from `main()`. The runner
  watches the exit code; non-zero = failed.

Minimal template:

```python
"""<one-line description of what this model does>"""
import os
import h2o
from h2o.estimators import H2OGradientBoostingEstimator
import pandas as pd

ARTIFACT_DIR = os.environ["PYLON_ARTIFACT_DIR"]
DATA_URL = "https://example.com/training-data.csv"

def main():
    print(f"Loading {DATA_URL}")
    df = pd.read_csv(DATA_URL)
    print(f"  {len(df):,} rows")

    # ... feature engineering ...

    h2o.init()
    h2o_df = h2o.H2OFrame(df)

    model = H2OGradientBoostingEstimator(ntrees=200, max_depth=6, seed=42)
    model.train(x=[...], y="target", training_frame=h2o_df)

    print(model.model_performance(h2o_df))

    h2o.save_model(model, path=ARTIFACT_DIR, force=True)
    h2o.shutdown(prompt=False)

if __name__ == "__main__":
    main()
```

### 4. Compose the metadata JSON

Metadata is captured at job-create time and travels with the model into the
artifact tarball as `pylon_metadata.json`. The serving layer reads it later
to know what the model accepts + returns.

```json
{
  "prompt": "<what the user asked for, plain English>",
  "dataSource": {
    "type": "url",
    "value": "<the actual training data URL>"
  },
  "inputSchema": [
    { "name": "feature_1", "type": "double" },
    { "name": "feature_2", "type": "integer" }
  ],
  "outputSchema": [
    { "name": "prediction", "type": "double" }
  ]
}
```

**Schema types** (MLflow vocabulary): `double | float | long | integer |
boolean | string | datetime`. Names must match what the script actually
feeds H2O at training time and what consumers will send at inference time.

### 5. Submit the job

```bash
curl -X POST "$PYLON_API_URL/api/jobs" \
  -H "Authorization: Bearer $PYLON_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d @- <<'EOF'
{
  "name": "<descriptive-name>",
  "script": "<the entire Python script as one string>",
  "dependencies": ["h2o==3.46.0.11", "pandas", "numpy"],
  "metadata": { ... }
}
EOF
```

**Response shape:**

```json
{
  "id":           "abc123def456",
  "manifestUrl":  "http://localhost:3000/api/jobs/abc123def456/manifest",
  "dashboardUrl": "http://localhost:3000/dashboard/jobs/abc123def456",
  "runnerOnline": true
}
```

If `runnerOnline` is `false`, the job is created but no runner will pick it
up until the user starts one. Tell the user that.

### 6. Hand the user the dashboard URL

End your turn with the `dashboardUrl` from the response, formatted clearly:

> Training started — watch live progress at <dashboardUrl>

That URL shows the streaming stage stepper, elapsed time, the runner's
stdout/stderr in real time, the final metrics, and a Cancel button.

**Do not poll the job status from the API yourself unless the user explicitly
asks.** The dashboard is the user-facing surface; you'd just be duplicating it.

---

## Useful API endpoints (reference, not always needed)

| Endpoint | Auth | Purpose |
|---|---|---|
| `POST /api/auth/pair/start` | none | Begin device-flow pairing → returns `pair_id`, `user_code`, `verification_url` |
| `GET /api/auth/pair/poll?id=<pair_id>` | none | Poll for pairing status; returns `api_token` once user confirms |
| `POST /api/auth/pair/confirm` | session | User-side: confirm a pairing (called by the /pair page) |
| `GET /api/runners/status` | session | Is the user's runner online? |
| `POST /api/jobs` | session OR Bearer | Create a job |
| `GET /api/jobs/[id]/manifest` | none (job ID = capability) | What the runner fetches |
| `GET /api/jobs/[id]/progress` | none | Current stage + log tail (polled by dashboard) |
| `POST /api/jobs/[id]/cancel` | session | User cancels a job |
| `GET /api/jobs/[id]/script` | none | Serve the script to the runner |

The `progress` GET returns:
```json
{
  "id": "...",
  "name": "...",
  "status": "created | running | uploaded | done | failed | cancelled",
  "progress": { "stage": "training", "message": "…", "ts": "…" },
  "logs":    [{ "stream": "stdout", "line": "…", "ts": "…" }],
  "metrics": { "rmse": 1234.5 },
  "error":   null,
  "createdAt": "…", "completedAt": "…"
}
```

---

## Composing scripts that work first time

A few things that bite people repeatedly:

1. **`__file__` paths fail** — the runner copies the script to a temp dir
   that has nothing else in it. Never use `Path(__file__).parent / "..."`.
2. **`os.environ["PYLON_ARTIFACT_DIR"]` without a fallback** — fine. The
   runner always sets it. If you fall back to `"."` you'll silently write
   into the wrong directory.
3. **Untyped data after `pd.read_csv`** — NESO CSVs have mixed date formats
   across years; `pd.to_datetime(..., format="mixed", dayfirst=True)` is the
   safe parse.
4. **Forgetting `h2o.shutdown(prompt=False)`** — the runner's `cmd.Wait()`
   blocks on the H2O JVM until you do. Without it the job appears to hang
   after "Training complete".
5. **Time-series data with random train/test split** — leakage. Always
   chronological split.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `runnerOnline: false` after POST | Runner not started or token mismatch | Have the user run `pylon-runner --watch --token …` |
| `401 invalid token` on any endpoint | Token missing / expired / wrong | Call `pylon_status` to re-pair; token is saved to `~/.pylon/credentials.json` |
| Pairing `expired` | User didn't confirm within 15 min | Start a new pair with `POST /api/auth/pair/start` |
| Pairing `consumed` | The token was already polled (someone else read it) | Start a new pair — old `pair_id` can't be replayed |
| Job hangs in "running", no progress | Network drop, runner crash | Dashboard's stale warning fires after 60s. Tell user to cancel + retry |
| "script exited with error: exit status 1" | Python script crashed | The dashboard's red error box has the stderr tail with the traceback — quote it back to the user |
| "no artifact was written to PYLON_ARTIFACT_DIR" | Script forgot to call `save_model(path=ARTIFACT_DIR)` | Fix the script |
| Connection refused on `/api/jobs` | Webapp isn't running | `cd app...