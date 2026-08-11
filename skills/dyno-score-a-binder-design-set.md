---
name: dyno-score-a-binder-design-set
description: Upload a set of candidate binder structures to Dyno Phi and run the filter pipeline (inverse folding to folding to complex folding to score), then retrieve the scored results. Use when asked to evaluate, filter, rank or score protein binder designs against a target.
api: Dyno Phi — Protein Design API
base_url: https://api.dyno-agents.app
spec: openapi/dyno-phi-openapi.yml
generated: '2026-08-10'
method: generated
source: openapi/dyno-phi-openapi.yml
operations:
  - create_ingest_session_v1_phi_ingest_sessions__post
  - get_upload_urls_v1_phi_ingest_sessions__session_id__upload_urls_post
  - finalize_ingest_session_v1_phi_ingest_sessions__session_id__finalize_post
  - get_ingest_session_v1_phi_ingest_sessions__session_id__get
  - submit_job_v1_phi_jobs__post
  - get_job_status_v1_phi_jobs__job_id__status_get
  - stream_job_logs_v1_phi_jobs__job_id__logs_stream_get
  - get_job_scores_v1_phi_jobs__job_id__scores_get
  - get_my_quota_v1_phi_auth_me_quota_get
  - cancel_job_v1_phi_jobs__job_id__delete
---

# Score a binder design set with Dyno Phi

Every call goes to `https://api.dyno-agents.app` and every call needs the
`x-api-key` header. A static-key caller must also send `X-Organization-ID`.
Create a key at <https://design.dynotx.com/dashboard/settings>.

If you have a shell available, the provider's own CLI (`pip install dyno-phi`)
does all of this in two commands — see `skills/dyno-phi.md`. Use this skill when
you must call the REST API directly.

## 0. Check quota before you start

`GET /v1/phi/auth/me/quota` (`get_my_quota_v1_phi_auth_me_quota_get`) returns
`max_total_jobs`, `max_concurrent_jobs`, `current_total_jobs`,
`current_concurrent_jobs` and `reset_at`. A value of `-1` means unlimited.

This matters: there are **no rate-limit response headers** on this API, so the
quota endpoint is the only way to see how close you are to the cap. Exceeding it
returns **HTTP 429** on job submission with no `Retry-After`.

## 1. Ingest the structures

Structures are PDB or mmCIF files. Use an ingest session for more than one file.

1. `POST /v1/phi/ingest_sessions/` — `create_ingest_session_v1_phi_ingest_sessions__post`
2. `POST /v1/phi/ingest_sessions/{session_id}/upload_urls` — `get_upload_urls_v1_phi_ingest_sessions__session_id__upload_urls_post` returns signed URLs
3. `PUT` each file to its signed URL with `Content-Type: application/octet-stream`
4. `POST /v1/phi/ingest_sessions/{session_id}/finalize` — `finalize_ingest_session_v1_phi_ingest_sessions__session_id__finalize_post`
5. Poll `GET /v1/phi/ingest_sessions/{session_id}` — `get_ingest_session_v1_phi_ingest_sessions__session_id__get` until status is `READY` (or `FAILED`)

Finalizing produces a **Dataset**; keep its `dataset_id`.

Retry a failed signed-URL `PUT` on 429/500/502/503/504 — three attempts,
exponential backoff base 2, which is what the provider's own client does.

For a single file, `POST /v1/phi/files/upload` (multipart) or
`POST /v1/phi/files/upload-url` are simpler.

## 2. Submit the filter pipeline

`POST /v1/phi/jobs/` — `submit_job_v1_phi_jobs__post`, body `JobSubmitRequest`:

```json
{
  "job_type": "filter_pipeline",
  "dataset_id": "<dataset_id>",
  "params": {
    "plddt_threshold": 0.80,
    "ptm_threshold": 0.55,
    "iptm_threshold": 0.50,
    "ipae_threshold": 10.85,
    "rmsd_threshold": 3.5,
    "msa_tool": "single_sequence"
  },
  "priority": 0
}
```

Those five thresholds are the provider's `default` preset. The `relaxed` preset
is `ptm 0.45`, `ipae 12.4`, `rmsd 4.5`. Use `msa_tool: "single_sequence"` for de
novo binders with no natural homologs, `"mmseqs2"` when homologs help.

`job_type` is a closed enum: `esmfold`, `proteinmpnn`, `alphafold`,
`rfdiffusion`, `ligandmpnn`, `chai1`, `boltz`, `align_structures`, `tm_score`,
`af2rank`, `rso`, `bindcraft`, `rf3`, `rfdiffusion3`, `boltzgen`, `esm2`,
`openfold3`, `research`, `design_pipeline`, `filter_pipeline`. Run a single
stage by naming it instead of `filter_pipeline`.

`dataset_id` is mutually exclusive with inline `fasta_str` / `pdb_content` in
`params`.

You may pass your own `run_id`. Do **not** treat it as an idempotency key — the
API publishes no idempotent-replay semantics, so a retried submit may create a
second job and consume quota twice.

## 3. Poll to completion

`GET /v1/phi/jobs/{job_id}/status` — `get_job_status_v1_phi_jobs__job_id__status_get`.

- Non-terminal: `pending`, `running`
- Terminal: `completed`, `failed`, `cancelled`

Poll every **5 seconds**; the provider's own client gives up after **7200
seconds**. `progress` carries `current_step`, `percent_complete` and
`eta_seconds`.

**A failed job is not an HTTP error.** Submission returns 200 and the failure
appears as `status: "failed"` with an `error` string. Check the terminal status,
not the HTTP code.

For live detail, `GET /v1/phi/jobs/{job_id}/logs/stream`
(`stream_job_logs_v1_phi_jobs__job_id__logs_stream_get`) is a `text/event-stream`.

Abandon a run with `DELETE /v1/phi/jobs/{job_id}` — `cancel_job_v1_phi_jobs__job_id__delete`.

## 4. Retrieve the scores

`GET /v1/phi/jobs/{job_id}/scores` — `get_job_scores_v1_phi_jobs__job_id__scores_get`
returns `ScoresDownloadResponse {job_id, artifact_id, download_url, filename,
expires_in}`. Fetch `download_url` before `expires_in` elapses.

The payload is a **CSV**, not typed JSON. Expect these columns, per the
provider's documented metric table:

| Metric | Source | Good threshold |
|---|---|---|
| `plddt` | ESMFold | >= 0.80 |
| `ptm` | AlphaFold2 | >= 0.55 |
| `af2_iptm` | AlphaFold2 multimer | >= 0.50 |
| `af2_ipae` | AlphaFold2 multimer | lower is better |
| `rmsd` | binder vs design backbone | <= 3.5 A |

`GET /v1/phi/datasets/{dataset_id}/scores` gives the same thing for the whole
dataset.

## Error handling

The envelope is `{"detail": ...}` — **not** RFC 9457 problem+json.

| Status | Meaning | Do |
|---|---|---|
| 401 | `Missing API key. Provide an x-api-key header.` | Send `x-api-key`; check `X-Organization-ID` |
| 422 | Validation error; `detail` is an array of `{loc, msg, type}` | Fix the field named in `loc` |
| 429 | Job quota exceeded | Check `/auth/me/quota`, cancel a running job, or wait for `reset_at` |

Only 200 and 422 are declared in the OpenAPI. 401 and 429 are real and
undeclared — handle them anyway.

## Cost and safety

There is **no test mode and no dry run**. Every job is real GPU compute against
production and counts against quota. To rehearse, call
`GET /v1/phi/tutorial` — it pre-creates a tutorial dataset of five PD-L1 binder
structures in your org and hands back signed download URLs.
