---
name: train-model
description: Prepare a dataset, establish model-training goals, then submit a Python training job to Pylon when the spec is complete. Use for /train-model, dataset-to-CSV preparation, and train/build/fit model requests.
---

# Train Model

Use this skill for `/train-model`. First establish the dataset and optimisation spec. Once the spec is complete, submit the training job to Pylon. You are submitting jobs on the user's behalf; never run training locally yourself.

Pylon is a self-hosted "train anything as an API" platform. A local `pylon-runner --watch` process connects to the Pylon webapp and runs submitted scripts. The runner sets up Python, runs the script, streams logs, and uploads artifacts from `PYLON_ARTIFACT_DIR`.

## When Triggered

1. Ask the user for the dataset source if missing:
   - local file path, upload, database export, API source, or sample rows
   - current format, such as CSV, JSONL, JSON, spreadsheet, SQL export, or plain text
   - target prediction/output column, if known
2. If a dataset is available, inspect its schema and propose an optimised training CSV layout.
3. Format or plan the CSV with:
   - stable column names in snake_case
   - one row per training example
   - explicit input, target, metadata, split, and weight columns where useful
   - normalized labels and categorical values
   - removed duplicates, empty rows, and unusable records
   - escaped newlines and delimiters
   - UTF-8 encoding
4. Ask follow-up questions about optimisation goals before recommending training settings.
5. Once the dataset schema, target, optimisation goal, and training approach are clear, use the Pylon flow below.

## Follow-Up Questions

Ask only the questions needed for the dataset and model type. Prefer concise batches of 3-6 questions.

- What should the model optimise for: accuracy, precision, recall, latency, cost, safety, fairness, ranking quality, or another metric?
- Which errors are most expensive: false positives, false negatives, bad ranking, hallucination, unsafe output, or slow responses?
- What is the target model/task type: classifier, regressor, ranker, recommender, extractor, chatbot, embedding model, or fine-tuned LLM?
- Are there required train, validation, and test split rules?
- Are there columns that must be excluded for privacy, leakage, compliance, or bias reasons?
- Are there class imbalance, rare label, multilingual, or domain-specific edge cases to preserve?
- What deployment constraints matter: maximum latency, memory, cost per prediction, batch size, or runtime environment?

## Pylon Flow

### 1. Confirm Pylon is reachable

The webapp lives at `PYLON_API_URL`, defaulting to `http://localhost:3000`. The API token lives in `PYLON_API_TOKEN`, either as an environment variable or pasted by the user. If no token is available, ask the user for it.

Check reachability:

```bash
curl -sS -o /dev/null -w "%{http_code}\n" "$PYLON_API_URL/api/runners/status" \
  -H "Authorization: Bearer $PYLON_API_TOKEN"
```

- `200` with `{"online": true}`: runner is connected; submit jobs.
- `200` with `{"online": false}`: webapp is up, runner is not running; ask the user to pair the runner.
- `401`: token is wrong; ask the user to copy it again from the dashboard.
- Connection refused: webapp is not running; bootstrap it.

### 0. Bootstrap the webapp if needed

Skip this if Pylon is reachable.

```bash
cd <repo>
docker compose up -d

cd apps/webapp
cp .env.example .env
# Edit .env and set BETTER_AUTH_SECRET=$(openssl rand -base64 32)
pnpm install
pnpm db:migrate
pnpm dev
```

The user signs in at `/sign-in`, opens `/dashboard`, and copies the API token from the "Connect your local runner" card.

### 2. Pair the runner if needed

If `/api/runners/status` reports `{"online": false}`, ask the user to run:

```bash
cd <repo>/runner
go build -o pylon-runner ./cmd/runner
./pylon-runner --watch --token "$PYLON_API_TOKEN" --api "$PYLON_API_URL"
```

They must keep that terminal open.

### 3. Compose the training script

Follow the Pylon SDK contract:

- Write the model to `os.environ["PYLON_ARTIFACT_DIR"]`; only that directory is uploaded.
- Load data from URLs, not repo-local paths. If the source is local, upload it first or embed small data as base64.
- Print informative stage logs; stdout and stderr stream to the dashboard.
- Return from `main()` instead of calling `sys.exit(0)` early.
- Do not use `__file__` or relative repo paths.
- Call `h2o.shutdown(prompt=False)` when using H2O.
- Use chronological splits for time-series data.

Minimal script shape:

```python
"""Train a model from prepared CSV data."""
import os
import h2o
from h2o.estimators import H2OGradientBoostingEstimator
import pandas as pd

ARTIFACT_DIR = os.environ["PYLON_ARTIFACT_DIR"]
DATA_URL = "https://example.com/training-data.csv"

def main():
    print(f"Loading {DATA_URL}")
    df = pd.read_csv(DATA_URL)
    print(f"Loaded {len(df):,} rows")

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

### 4. Compose metadata

Metadata is captured at job creation and packaged as `pylon_metadata.json`.

```json
{
  "prompt": "<what the user asked for>",
  "dataSource": {
    "type": "url",
    "value": "<training data URL>"
  },
  "inputSchema": [
    { "name": "feature_1", "type": "double" }
  ],
  "outputSchema": [
    { "name": "prediction", "type": "double" }
  ]
}
```

Use MLflow schema types: `double`, `float`, `long`, `integer`, `boolean`, `string`, `datetime`. Names must match what the script uses and what inference consumers will send.

### 5. Submit the job

```bash
curl -X POST "$PYLON_API_URL/api/jobs" \
  -H "Authorization: Bearer $PYLON_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d @- <<'EOF'
{
  "name": "<descriptive-name>",
  "script": "<entire Python script as one string>",
  "dependencies": ["h2o==3.46.0.11", "pandas", "numpy"],
  "metadata": {}
}
EOF
```

Expected response:

```json
{
  "id": "abc123def456",
  "manifestUrl": "http://localhost:3000/api/jobs/abc123def456/manifest",
  "dashboardUrl": "http://localhost:3000/dashboard/jobs/abc123def456",
  "runnerOnline": true
}
```

If `runnerOnline` is `false`, the job is created but no runner will pick it up until the user starts one.

### 6. Hand off to the dashboard

End with the `dashboardUrl` from the response:

> Training started - watch live progress at <dashboardUrl>

Do not poll job status unless the user explicitly asks.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `runnerOnline: false` after POST | Runner not started or token mismatch | Ask the user to run `pylon-runner --watch --token ...` |
| `401 invalid token` | Token regenerated in dashboard | Ask for a new token |
| Job hangs in running with no progress | Network drop or runner crash | Tell the user to cancel and retry from the dashboard |
| Script exited with error | Python script crashed | Use the dashboard stderr tail to fix the script |
| No artifact was written | Script did not save to `PYLON_ARTIFACT_DIR` | Fix artifact saving |
| Connection refused | Webapp is not running | Start `apps/webapp` with `pnpm dev` |

## Repo References

Use these only when working inside a Pylon repo:

| Path | What |
|---|---|
| `apps/webapp/src/app/api/jobs/route.ts` | POST handler |
| `apps/webapp/src/app/api/jobs/[id]/manifest/route.ts` | Runner manifest |
| `apps/webapp/src/app/api/runners/listen/route.ts` | Runner SSE endpoint |
| `apps/webapp/src/lib/db/schema.ts` | Job metadata schema |
| `runner/cmd/runner/main.go` | Runner CLI and watch loop |

## End-of-Turn Checklist

Before finishing after a submission, confirm:

- Submitted to `POST /api/jobs`.
- Used `Authorization: Bearer $PYLON_API_TOKEN`.
- Included `metadata.prompt`, `metadata.dataSource`, `metadata.inputSchema`, and `metadata.outputSchema`.
- Gave the user the `dashboardUrl`.
- Did not poll progress unless asked.
- Did not run the script locally.

## Output Before Submission

Return:

1. The requested dataset input or missing dataset details.
3. The proposed optimised CSV schema.
4. Any cleaning or transformation steps to apply.
5. Follow-up questions about optimisation goals.
6. The Pylon submission plan once the spec is complete.
