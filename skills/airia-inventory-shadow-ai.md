---
name: airia-inventory-shadow-ai
description: >-
  Pull the AI asset inventory out of Airia — discovered agents, models and MCP servers across
  connected platforms — and export it, using the Discovery and AiAssets operations. Use when you need
  a machine-readable answer to "what AI is running in this organization".
api: Airia Web APIs
base_url: https://api.airia.ai
auth: X-API-Key header
operations:
  - AiAssets_GetFilteredAiAssets
  - AiAssets_GetAssetWithUsage
  - AiAssets_GetUnifiedAgent
  - AiAssets_GetMcpServerDetails
  - AiAssets_GetConsolidatedApplications
  - AiAssets_TriggerSync
  - AiAssets_ExportAllModelsToCsv
  - AiAssets_ExportAgentsToCsv
  - ShadowAi_GetEffectiveShadowAiUnifiedPolicy
  - ShadowAi_GetShadowAiConfigurations
generated: '2026-09-19'
method: generated
source: openapi/airia-openapi.yml + https://airia.ai/docs/discovery/overview
---

# Inventory the AI in an organization

Airia's Discover surface connects to the platforms where AI actually runs — AWS Bedrock, Azure AI
Foundry, Copilot Studio, Vertex/Google Agent Platform, Glean, LangGraph, n8n, Okta, Entra ID,
ServiceNow, Databricks, GitHub/GitLab (via the Airia Code Scanner), Purview, Cloudflare Zero Trust
logs and a browser extension — and consolidates what it finds into one inventory. These operations
read that inventory.

## 1. List the assets

```
GET /v1/AiAssets                       # AiAssets_GetFilteredAiAssets
GET /v1/AiAssets/{assetId}             # AiAssets_GetAssetWithUsage — asset plus where it is used
GET /v1/AiAssets/agent/{assetId}       # AiAssets_GetUnifiedAgent
GET /v1/AiAssets/mcpServer/{mcpServerId}   # AiAssets_GetMcpServerDetails
GET /v1/AiAssets/applications          # AiAssets_GetConsolidatedApplications
```

Standard paging: `PageNumber`, `PageSize`, `SortBy`, `SortDirection`, `filter`, `IncludeTotalCount`.

## 2. Refresh before you read

```
POST /v1/AiAssets/sync                 # AiAssets_TriggerSync
```

This is one of the ten operations that declares **429**. It is also a write with no idempotency key:
a retry starts another sync. Poll the inventory rather than re-firing the sync.

## 3. Export

```
GET /v1/AiAssets/models/export/csv          # every model
GET /v1/AiAssets/models/{assetId}/export/csv
GET /v1/AiAssets/agents/export/csv
```

These return `application/octet-stream`, not JSON.

## 4. Shadow-AI policy

```
GET /v2/ShadowAi/policy           # ShadowAi_GetEffectiveShadowAiUnifiedPolicy — prefer v2
GET /v1/ShadowAi/policies         # browser-extension rules
GET /v1/ShadowAi/configurations   # discovery configurations
```

## 5. What the governance record carries

Assets rolled into governance are typed against the frameworks themselves, not free text:
`euAiAct` (Prohibited / HighRisk / LimitedRisk / MinimalRisk / NotApplicable), `nistAiRmfTier`
(Tier1–Tier5) and `iso42001Level` (Level1–Level5), each with a confidence field, and model records
carry certification types from a fixed vocabulary (SOC2, ISO 27001, ISO 27017, ISO 27018, ISO 27701,
HIPAA). If your own register already speaks those frameworks, it maps across without a translation
layer. See `conformance/airia-conformance.yml`.

## 6. Watch the deprecated ones

`AuditLog_SearchAuditEntries` (`GET /v1/AuditLog/search`) and
`Guardrail_GetAssignedGuardrailsByEntityIds` are marked **deprecated** in the spec with no stated
removal date and no Sunset header. Do not build a long-lived integration on them; see
`lifecycle/airia-lifecycle.yml`.
