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

### 1. Confirm Pylon is reachable + you have a token

The webapp lives at `PYLON_API_URL` (default `http://localhost:3000` for local
dev). The user's API token lives in `PYLON_API_TOKEN` — either an env var, a
file at `~/.pylon/credentials`, or a token from a fresh pair flow (see step
1a).

```bash
# Reachable at all?
curl -sS -o /dev/null -w "%{http_code}\n" "$PYLON_API_URL/api/runners/status" \
  -H "Authorization: Bearer ${PYLON_API_TOKEN:-none}"
```

- `200` with `{"online": true}`  — runner is connected, submit away
- `200` with `{"online": false}` — webapp up, runner not running (step 2)
- `401` — token is missing/invalid; start the pair flow (step 1a)
- Connection refused — webapp not running (step 0)

### 1a. (If you don't have a token) Pair via the browser

Don't ask the user to copy a token from the dashboard. Use the device flow:

```bash
# 1. Start a pairing
resp=$(curl -sS -X POST "$PYLON_API_URL/api/auth/pair/start")
pair_id=$(echo "$resp"   | jq -r .pair_id)
user_code=$(echo "$resp" | jq -r .user_code)
verify_url=$(echo "$resp" | jq -r .verification_url)
```

Then tell the user, verbatim:

> Open **<verify_url>** in your browser and click **Confirm pairing**.
> Code: **<user_code>**

Then poll until they confirm (or 15 min elapses):

```bash
while :; do
  poll=$(curl -sS -w "\n%{http_code}" "$PYLON_API_URL/api/auth/pair/poll?id=$pair_id")
  status_code=$(tail -n1 <<< "$poll")
  body=$(sed '$d' <<< "$poll")
  case "$(echo "$body" | jq -r .status)" in
    confirmed)
      token=$(echo "$body" | jq -r .api_token)
      mkdir -p ~/.pylon && echo "$token" > ~/.pylon/credentials && chmod 600 ~/.pylon/credentials
      export PYLON_API_TOKEN=$token
      break
      ;;
    expired|consumed) echo "pairing failed: $body"; exit 1 ;;
  esac
  sleep 2
done
```

Once `~/.pylon/credentials` exists, prefer reading the token from there on
subsequent runs (`PYLON_API_TOKEN=$(cat ~/.pylon/credentials)`) so the user
only pairs once.

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

The user then signs in at `/sign-in`, lands on `/dashboard`, copies their API
token from the "Connect your local runner" card.

### 2. (If needed) Pair the runner

If `/api/runners/status` returned `{"online": false}`, the user needs to run
the runner:

```bash
cd <repo>/runner
go build -o pylon-runner ./cmd/runner
./pylon-runner --watch --token $PYLON_API_TOKEN --api $PYLON_API_URL
```

They keep that terminal open. The runner sits idle until you push a job.

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
| `401 invalid token` on any endpoint | Token missing / expired / wrong | Run the device flow again (step 1a); save fresh token to `~/.pylon/credentials` |
| Pairing `expired` | User didn't confirm within 15 min | Start a new pair with `POST /api/auth/pair/start` |
| Pairing `consumed` | The token was already polled (someone else read it) | Start a new pair — old `pair_id` can't be replayed |
| Job hangs in "running", no progress | Network drop, runner crash | Dashboard's stale warning fires after 60s. Tell user to cancel + retry |
| "script exited with error: exit status 1" | Python script crashed | The dashboard's red error box has the stderr tail with the traceback — quote it back to the user |
| "no artifact was written to PYLON_ARTIFACT_DIR" | Script forgot to call `save_model(path=ARTIFACT_DIR)` | Fix the script |
| Connection refused on `/api/jobs` | Webapp isn't running | `cd app...