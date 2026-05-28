---
name: train-model
description: Placeholder workflow for /train-model requests. Use when a user asks to prepare a dataset, format training data as CSV, or plan optimisation goals before model training.
---

# Train Model

This is a placeholder skill for `/train-model`. It prepares the conversation and dataset plan for a future model-training implementation. It does not start training jobs.

## When Triggered

1. Ask the user for the dataset source:
   - local file path, upload, database export, API source, or sample rows
   - current format, such as CSV, JSONL, JSON, spreadsheet, SQL export, or plain text
   - target prediction/output column, if known
2. State that this skill is currently a placeholder and no training job will be started.
3. If a dataset is available, inspect its schema and propose an optimised training CSV layout.
4. Format or plan the CSV with:
   - stable column names in snake_case
   - one row per training example
   - explicit input, target, metadata, split, and weight columns where useful
   - normalized labels and categorical values
   - removed duplicates, empty rows, and unusable records
   - escaped newlines and delimiters
   - UTF-8 encoding
5. Ask follow-up questions about optimisation goals before recommending training settings.

## Follow-Up Questions

Ask only the questions needed for the dataset and model type. Prefer concise batches of 3-6 questions.

- What should the model optimise for: accuracy, precision, recall, latency, cost, safety, fairness, ranking quality, or another metric?
- Which errors are most expensive: false positives, false negatives, bad ranking, hallucination, unsafe output, or slow responses?
- What is the target model/task type: classifier, regressor, ranker, recommender, extractor, chatbot, embedding model, or fine-tuned LLM?
- Are there required train, validation, and test split rules?
- Are there columns that must be excluded for privacy, leakage, compliance, or bias reasons?
- Are there class imbalance, rare label, multilingual, or domain-specific edge cases to preserve?
- What deployment constraints matter: maximum latency, memory, cost per prediction, batch size, or runtime environment?

## Output

Return:

1. A short status note that this is a placeholder workflow.
2. The requested dataset input or missing dataset details.
3. The proposed optimised CSV schema.
4. Any cleaning or transformation steps to apply.
5. Follow-up questions about optimisation goals.
6. The next implementation step for replacing the placeholder with real dataset conversion and training logic.
