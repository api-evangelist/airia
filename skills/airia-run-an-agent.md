---
name: airia-run-an-agent
description: >-
  Run an Airia agent (a "pipeline") from an application and read its result, including the streaming
  and multipart variants and how to stop a run. Use when you need to call an Airia agent
  programmatically with an API key.
api: Airia Web APIs
base_url: https://api.airia.ai
auth: X-API-Key header
operations:
  - PipelineExecution_RunPipelineV23
  - PipelineExecution_RunPipelineMultipartV23
  - PipelineExecution_GetPipelineExecutionResult2
  - PipelineExecution_ResumeStream
  - PipelineExecution_StopAgentStream
  - PipelineExecution_StopExecution
  - PipelinesConfig_GetPipelines2
  - PipelinesConfig_GetAgentSummary2
generated: '2026-09-19'
method: generated
source: openapi/airia-openapi.yml + https://airia.ai/docs/build/interface-options/api-deployment
---

# Run an Airia agent

Airia calls an agent a **pipeline** in its contract. The console shows an Agent; the API executes a
pipeline. Everything below uses the real operationIds from `openapi/airia-openapi.yml`.

## 1. Get a key

Create it in the console under **Settings → Developer → API Keys** (`Generate Key`). Two kinds:

- leave **Roles** empty → a *personal access token* that carries your own permissions and dies with
  your account;
- select roles → a *service account* key that keeps those roles regardless of what happens to you.

Use a service account for anything that outlives one person, and scope it to a single project unless
it genuinely needs all of them. The key value is shown **once**; a lost key is replaced, not
recovered. Send it as `X-API-Key` on every request.

## 2. Find the agent

```
GET /v1/PipelinesConfig            # PipelinesConfig_GetPipelines2 — paged list
GET /v1/PipelinesConfig/summary    # PipelinesConfig_GetAgentSummary2
GET /v1/PipelinesConfig/{id}       # PipelinesConfig_GetPipeline2
```

List operations take `PageNumber`, `PageSize`, `SortBy`, `SortDirection`, `filter` and
`IncludeTotalCount`. The total count is returned **only** when you ask for it.

## 3. Execute it

```
POST /v2/PipelineExecution/{pipelineId}                    # PipelineExecution_RunPipelineV23
POST /v2/PipelineExecution/Multipart/{pipelineIdentifier}  # file upload variant
POST /v1/PipelineExecution/batch                           # PipelineExecution_RunBatchPipelineExecution
GET  /v1/PipelineExecution/{executionId}                   # fetch the result later
```

Prefer the `/v2/` execution paths; the `/v1/` ones are the earlier generation of the same calls.

Send `x-correlation-id` on every call. Every one of the 1,299 operations accepts it, responses echo
it, and it is the id to quote to support. The Python SDK (`pip install airia`) generates one for you.

## 4. Streaming, and stopping

```
GET  /v2/PipelineExecution/ResumeStream/{executionId}   # reattach to a stream
POST /v2/PipelineExecution/StopStream                   # stop streaming
POST /v2/PipelineExecution/StopExecution                # stop the run
```

A dropped client on a streaming run surfaces as **499 Client Closed Request** — declared on three
operations. Reattach with `ResumeStream` rather than starting a second execution.

## 5. There is no idempotency key — this matters

The contract publishes **no `Idempotency-Key` header** anywhere across 619 mutating operations.
Retrying a failed execution **runs the agent again**, with whatever side effects its tools have.

Before you retry:

1. Fetch the execution with `PipelineExecution_GetPipelineExecutionResult2` using the executionId you
   already have.
2. Only re-POST if you never received one.
3. Keep your own `request → executionId` map keyed on your own idempotency key. Airia will not
   de-duplicate for you, and `x-correlation-id` is a tracing id, not a replay guard.

## 6. Errors

Every 4xx/5xx returns the ASP.NET Core `ProblemDetails` shape as `application/json` (not
`application/problem+json`): `type`, `title`, `status`, `detail`, `instance`, plus extensions.
Validation failures add an `errors` map keyed by field.

| Status | What it means here |
|---|---|
| 401 | No or invalid `X-API-Key` |
| 403 | Key is valid, its roles lack the permission — permissions resolve live on every request |
| 404 | Wrong id, or outside the key's project scope |
| 429 | Agent-execution rate exceeded: 3/s Professional, 4/s Teams, 5/s Enterprise (default) |
| 402 | A budget or licence limit blocked the call |
| 499 / 502 / 504 | Client disconnect, or an upstream model provider failed |

There are **no rate-limit response headers**, documented or observed, so back off on 429 by policy
rather than by reading a remaining budget.

See `errors/airia-problem-types.yml`, `rate-limits/airia-rate-limits.yml` and
`conventions/airia-conventions.yml`.
