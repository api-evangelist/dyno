---
name: dyno-run-a-published-protocol
description: Discover, fork and execute a Dyno Phi Protocol — a shareable, versioned workflow template — and collect the resulting artifacts. Use when asked to reuse an existing protein-design recipe on Dyno Phi, plan a multi-step design workflow, or run a workflow rather than a single job.
api: Dyno Phi — Protein Design API
base_url: https://api.dyno-agents.app
spec: openapi/dyno-phi-openapi.yml
generated: '2026-08-10'
method: generated
source: openapi/dyno-phi-openapi.yml
operations:
  - list_protocols_v1_phi_protocols_get
  - recommend_protocols_v1_phi_protocols_recommend_post
  - get_protocol_v1_phi_protocols__protocol_id__get
  - fork_protocol_v1_phi_protocols__protocol_id__fork_post
  - validate_protocol_template_v1_phi_protocols_validate_post
  - plan_and_execute_with_protocol_v1_phi_workflows_plan_execute_post
  - plan_workflow_v1_phi_workflows_plan_post
  - create_workflow_v1_phi_workflows__post
  - execute_workflow_v1_phi_workflows__workflow_id__execute_post
  - get_workflow_run_status_v1_phi_workflows__workflow_id__runs__run_id__status_get
  - get_workflow_results_v1_phi_runs__run_id__results_get
  - get_run_assets_v1_phi_runs__run_id__assets_get
  - get_protocol_executions_v1_phi_protocols__protocol_id__executions_get
  - star_protocol_v1_phi_protocols__protocol_id__star_post
---

# Run a published Protocol on Dyno Phi

A **Protocol** is Dyno Phi's reusable recipe: a versioned, forkable, starrable
template derived from a Workflow. A **Workflow** is the immutable plan itself —
a DAG of `NodeSpec {id, op, params, retry_policy, map_config}` joined by
`EdgeSpec {src, dst, condition}`. Executing either produces a **Run**, and a Run
owns **Artifacts** and **Assets**.

All calls go to `https://api.dyno-agents.app` with an `x-api-key` header, plus
`X-Organization-ID` for static-key callers.

## 1. Find a protocol

- `GET /v1/phi/protocols` — `list_protocols_v1_phi_protocols_get`. Filters:
  `visibility`, `tags`, `starred`. Paginates with `limit` (default 100) and
  `offset` (default 0) — note this collection uses limit/offset while
  `/v1/phi/jobs/` and `/v1/phi/datasets/` use `page`/`page_size`.
- `POST /v1/phi/protocols/recommend` —
  `recommend_protocols_v1_phi_protocols_recommend_post` when you can describe
  the intent rather than name the protocol.
- `GET /v1/phi/protocols/{protocol_id}` —
  `get_protocol_v1_phi_protocols__protocol_id__get` returns
  `ProtocolResponse {id, name, description, intent_signature,
  protocol_template, source_workflow_id, parent_protocol_id, version,
  visibility, tags, is_starred, created_at, updated_at}`.

Read `intent_signature` before running anything: it declares what the protocol
expects as input.

## 2. Fork before you modify

Never edit someone else's protocol in place. `POST /v1/phi/protocols/{protocol_id}/fork`
(`fork_protocol_v1_phi_protocols__protocol_id__fork_post`) creates your own copy
with `parent_protocol_id` set to the original, preserving lineage.

Then `PUT /v1/phi/protocols/{protocol_id}` to change it, and
`POST /v1/phi/protocols/validate`
(`validate_protocol_template_v1_phi_protocols_validate_post`) to check the
template before you spend compute on it. Validate first — it is the only
free step in this flow.

## 3. Execute

Two routes:

**Straight through** — `POST /v1/phi/workflows/plan/execute`
(`plan_and_execute_with_protocol_v1_phi_workflows_plan_execute_post`) plans and
runs against a protocol in one call.

**Explicit** — plan, inspect, then run:

1. `POST /v1/phi/workflows/plan` — `plan_workflow_v1_phi_workflows_plan_post`
   (there is also an unauthenticated-looking `/plan/public` variant,
   `plan_workflow_public_v1_phi_workflows_plan_public_post`)
2. `POST /v1/phi/workflows/` — `create_workflow_v1_phi_workflows__post`
3. `POST /v1/phi/workflows/{workflow_id}/execute` —
   `execute_workflow_v1_phi_workflows__workflow_id__execute_post`

Prefer the explicit route when the workflow is expensive: the plan tells you
which `op` nodes will run, and each node is a biomodal job that consumes quota.

Publish a stable version with
`POST /v1/phi/workflows/{workflow_id}/publish`; list history with
`GET /v1/phi/workflows/{workflow_id}/versions`.

## 4. Track the run

`GET /v1/phi/workflows/{workflow_id}/runs/{run_id}/status` —
`get_workflow_run_status_v1_phi_workflows__workflow_id__runs__run_id__status_get`.

Same status vocabulary as jobs: `pending`, `running` are non-terminal;
`completed`, `failed`, `cancelled` are terminal. Poll every 5 seconds. A failed
run surfaces as a terminal status, not an HTTP error.

## 5. Collect the output

- `GET /v1/phi/runs/{run_id}/results` — `get_workflow_results_v1_phi_runs__run_id__results_get`
- `GET /v1/phi/runs/{run_id}/assets` — `get_run_assets_v1_phi_runs__run_id__assets_get`
- `GET /v1/phi/artifacts/{artifact_id}/download` —
  `get_artifact_download_url_v1_phi_artifacts__artifact_id__download_get`
  returns a signed URL; fetch it before it expires

`ArtifactResponse` carries `{artifact_id, run_id, artifact_type, filename,
storage_path, size_bytes, mime_type, created_at, metadata}`.

## 6. Close the loop

- `GET /v1/phi/protocols/{protocol_id}/executions` —
  `get_protocol_executions_v1_phi_protocols__protocol_id__executions_get` shows
  the protocol's run history
- `POST /v1/phi/protocols/{protocol_id}/star` marks one worth keeping;
  `DELETE` on the same path unstars it

## Cautions

- **No idempotency key.** Re-issuing an execute call may start a second run and
  bill quota twice.
- **Only 200 and 422 are declared** in the OpenAPI. 401 (`Missing API key.
  Provide an x-api-key header.`) and 429 (job quota exceeded) are real.
- **No status page and no SLA.** `GET /health` is the only availability signal.
- Two paths exist for run and artifact access — the versioned `/v1/phi/...` set
  and an older untagged `/runs/...`, `/artifacts/...` set. Prefer the versioned
  ones.
