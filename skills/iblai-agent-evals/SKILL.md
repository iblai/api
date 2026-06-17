---
name: iblai-agent-evals
description: Measure and improve agent quality via the platform API — evaluation datasets, dataset items (JSON, CSV upload, or from chat traces), experiment runs, LLM-as-Judge and human-annotation scoring, score configs, and CSV export. Use to test an agent against a dataset and grade the results.
---

# iblai-agent-evals

Measure and improve an agent's quality from the API: build evaluation
datasets, run experiments that send each question to the agent, then grade the
results with LLM-as-Judge and/or human scores and export to CSV. Use to test an
agent against a dataset and grade the results.

## Auth & conventions

- **Base URL:** `https://api.iblai.app`
- **Header:** `Authorization: Api-Token $IBLAI_API_KEY` on every request.
  (The dev docs phrase this as `Authorization: Token <key>` — it is the same
  platform key; use **Api-Token**.)
- **Path vars:** `{org}` = `$IBLAI_ORG`, `{username}` = `$IBLAI_USERNAME`.
- **Host root:** `…/dm/api/ai-agent/orgs/{org}/users/{username}/evaluations/`.
  Below, `…/evals` = that root.
- Not connected yet? Run **`/iblai-login`** first to populate `IBLAI_ORG`,
  `IBLAI_USERNAME`, and `IBLAI_API_KEY`.

## Reads

### Datasets

- **GET** `https://api.iblai.app/dm/api/ai-agent/orgs/{org}/users/{username}/evaluations/datasets/` — list.
- **GET** `…/evaluations/datasets/{dataset_name}/` — get one.

### Dataset items

- **GET** `…/datasets/{dataset_name}/items/` — list items.

### Experiments (runs)

- **GET** `…/datasets/{dataset_name}/runs/` — list runs.
- **GET** `…/datasets/{dataset_name}/runs/{run_name}/` — run details.
- **GET** `…/datasets/{dataset_name}/runs/{run_name}/export/` — export results as CSV.

## Writes

### Datasets

- **POST** `…/evaluations/datasets/` — create.

### Dataset items

- **POST** `…/datasets/{dataset_name}/items/` — add items (direct JSON, or link
  from existing chat traces — the system extracts input + agent response from
  each trace).
- **POST** `…/datasets/{dataset_name}/items/upload/` — upload CSV.
- **PUT** `…/datasets/{dataset_name}/items/{item_id}/` — update an item.
- **DELETE** `…/datasets/{dataset_name}/items/{item_id}/` — delete an item. Destructive — confirm with the user first.

### Experiments (runs)

- **POST** `…/datasets/{dataset_name}/runs/` — start an experiment (async background task; sends each question to the agent, records responses).
- **DELETE** `…/datasets/{dataset_name}/runs/{run_name}/` — delete a run. Destructive — confirm with the user first.

### Grading

- **POST** `…/datasets/{dataset_name}/runs/{run_name}/evaluate/` — **LLM-as-Judge**: grade a whole run against custom criteria (score 0–1).
- **GET** `…/evaluations/scores/` ; **POST** `…/evaluations/scores/` ; **DELETE** `…/evaluations/scores/{score_id}/` — human/numeric/boolean/categorical scores. Deletion is destructive — confirm with the user first.
- **GET** `…/evaluations/score-configs/` ; **POST** `…/evaluations/score-configs/` — reusable scoring rubrics.

## Example

Start an experiment run against a dataset (runs async in the background):

```bash
curl -X POST \
  "https://api.iblai.app/dm/api/ai-agent/orgs/$IBLAI_ORG/users/$IBLAI_USERNAME/evaluations/datasets/support-qa/runs/" \
  -H "Authorization: Api-Token $IBLAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"run_name": "baseline-2026-06", "mentor": "d17dc729-60fd-4363-81a0-f67d9318b03e"}'
```

## Notes

- Typical pipeline: **create dataset → add items → start run (async) → grade
  (LLM-as-Judge and/or human scores) → export CSV**.
- Experiment runs are async (background tasks) — POST returns before grading is
  possible; poll the run-details endpoint until the responses are recorded
  before calling `evaluate/` or `export/`.
- LLM-as-Judge scores a whole run against criteria you supply (0–1); per-item
  human scores go through `…/evaluations/scores/` and can reuse a rubric from
  `…/evaluations/score-configs/`.
- Dataset items can come from direct JSON, a CSV upload, or existing chat traces
  (the system pulls input + agent response out of each trace).
