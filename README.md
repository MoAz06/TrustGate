# TrustGate

Runtime authorization for AI agents that checks the data supply chain before a high-impact action executes.

Agent governance tools answer one question: *is this agent allowed to call this tool?* They check identity, match it against a policy, and permit or deny the call. Microsoft's Agent Governance Toolkit, Amazon Bedrock AgentCore Policy, and the MCP specification all do this. It is a necessary control.

There is a second question none of them answer: *should this agent act, given the current state of the data behind the decision?*

These are not the same question. An agent can be fully authorized to call `approve_refund` — right identity, right policy — and still cause damage because the Fivetran sync behind the decision is stale, a new `customer_tier` enum appeared that the input contract never authorized, or the BigQuery row the agent is reading is 42 minutes old when the SLA requires freshness within 15. In each case, the authorization check passes. The damage still happens.

## Why data supply chain trust

AI agents are being deployed into production systems where the data they act on is not static. It comes from syncs, pipelines, and APIs. It has freshness windows. It has schemas that evolve. A governance layer that only checks agent identity and tool permission is built for a world where the data is always correct, always fresh, and always matches the contract the policy was written against.

That world does not exist in production.

TrustGate sits between the agent and the action. Before `approve_refund` executes, it checks live Fivetran evidence, a BigQuery row from the Fivetran-synced table, and a versioned input contract. The policy engine returns `ALLOW`, `APPROVAL_REQUIRED`, or `BLOCK` with a full evidence receipt. The model proposes. The policy engine decides.

The three checks are deterministic, not probabilistic:

- **Freshness** — is the data behind this decision newer than the SLA requires?
- **Schema compliance** — does the data match the shape the policy was written against? A new enum value the contract never authorized means the policy was not written for this case.
- **Connection validity** — is the data source reachable and reporting a successful sync? A schema failure or dropped connection becomes a hard BLOCK, not a silent pass.

This repo is scoped as a hackathon proof. The build log, demo script, and setup notes are public so the flow can be reproduced rather than only watched.

```text
Gemini / Vertex AI function call
-> proposeTrustGateAction
-> Cloud Run TrustGate API
-> live Fivetran REST evidence
-> live BigQuery row evidence from the Fivetran-synced table
-> deterministic policy decision
-> receipt returned to the agent
```

## Architecture

```mermaid
flowchart TD
    subgraph SC["Data supply chain (Fivetran-synced)"]
        direction LR
        GS["Google Sheets"] -->|"Fivetran connection: fulfill_pageant"| BQT[("BigQuery table<br/>trustgate_demo.customers")]
    end

    AGENT["Customer Recovery Agent<br/>Gemini in Agent Builder"]
    TG["TrustGate backend<br/>Cloud Run"]
    FMCP["Official Fivetran MCP server<br/>over stdio"]
    FREST["Fivetran REST API"]
    POLICY{"Deterministic policy engine<br/>risk_score = sum of rule weights, no ML"}
    ALLOW(["ALLOW"])
    APPROVE(["APPROVAL_REQUIRED"])
    BLOCK(["BLOCK"])

    AGENT -->|"calls /mcp (proposeTrustGateAction)"| TG
    TG -->|"Fivetran evidence"| FMCP
    TG -->|"Fivetran evidence (stable path)"| FREST
    TG -->|"reads live row"| BQT
    TG --> POLICY
    POLICY --> ALLOW
    POLICY --> APPROVE
    POLICY --> BLOCK
    POLICY -.->|"auditable receipt"| AGENT
```

The Gemini agent only calls TrustGate's own `/mcp` surface. TrustGate's backend is what calls the official Fivetran MCP server, Fivetran REST, and the Fivetran-synced BigQuery row, then a deterministic policy returns the decision.

## Current Proof

The Cloud Run deployment ran during the hackathon and has since been retired, so the paths below are relative to wherever you run it. The recorded walkthrough shows the whole loop against the live deployment: https://www.youtube.com/watch?v=IeTSkqudJBY

- Gemini agent endpoint: `POST /api/agent/run`
- Action endpoint: `POST /api/actions/propose`
- MCP tool endpoint for Agent Builder `Add tools -> MCP Server`: `POST /mcp` (TrustGate's own MCP surface, not Fivetran's MCP)
- OpenAPI spec for tool imports when available: `/openapi.json`
- Live Fivetran REST evidence observed in receipts: `source=fivetran_rest_live`
- Live Fivetran MCP evidence (official Fivetran MCP server at runtime): `source=fivetran_mcp_live` via `/api/fivetran/mcp-evidence`
- Live BigQuery evidence observed in hosted receipts: `source=bigquery_rest_live`
- Fivetran connection used in the demo: `fulfill_pageant`
- Working Gemini paths: `/api/agent/run` and `scripts/vertex_trustgate_agent_demo.py`
- Build notes with the things that broke: `BUILD_LOG.md`
- Exact video flow: `DEMO_SCRIPT.md`

## What It Does

The demo is a customer recovery agent deciding whether a refund may proceed.

1. Safe data and a normal refund amount return `ALLOW`.
2. A new observed `customer_tier` value outside contract v1 returns `APPROVAL_REQUIRED`.
3. A scoped human approval can allow refunds under `$50` until contract v2 is reviewed.
4. A critical schema/connection failure returns `BLOCK`.

Gemini explains the receipt. Gemini does not decide the policy.

`risk_score` is not a model confidence score. It is a transparent sum of triggered rule weights, shown as `risk_breakdown` in the receipt.

## Dashboard

Opening `localhost:8080` loads an interactive control-room dashboard, not just an API. From the Action Console you set a refund amount and run the agent or the policy. Each decision renders as:

- a colour-coded decision banner (green `ALLOW`, amber `APPROVAL_REQUIRED`, red `BLOCK`) with the one-line reason and the receipt id,
- an evidence trace, Agent -> Fivetran -> BigQuery -> Policy, with the live source at each step,
- evidence cards for the Fivetran REST data, the BigQuery row, the contract diff, and the risk breakdown,
- a `Gemini Agent Run` panel with the tool call and Gemini's explanation,
- a developer-view toggle for the raw receipt JSON.

The `Simulate` buttons inject a new customer tier, a stale sync, or a schema failure so the three decision paths can be shown live. The web app is served by the Cloud Run service. It is built with React loaded as a local vendored bundle (no build step), so the same `public/` is served identically in local dev and on Cloud Run.

## Run Locally

```bash
npm run check
npm test
npm start
```

Open `http://localhost:8080`.

Without Fivetran credentials, the app uses clearly labeled demo evidence. With credentials, it reads Fivetran REST evidence.

Small reliability endpoints:

```bash
curl http://localhost:8080/health
curl http://localhost:8080/api/healthz
curl http://localhost:8080/readyz
```

`/health` and `/api/healthz` are simple liveness checks. `/readyz` reports configuration status by env var name only. It does not return secret values or call external APIs.

## Fivetran Evidence

For the Fivetran track, TrustGate now uses both integration paths listed in the hackathon Fivetran resources: the Fivetran REST API and the official open-source Fivetran MCP server. REST is the stable evidence path, and the MCP server is an additional live partner-evidence path shown in `evidence.fivetran_mcp`.

Create `.env` from `.env.example` and add a scoped Fivetran API key:

```bash
FIVETRAN_API_KEY=...
FIVETRAN_API_SECRET=...
FIVETRAN_CONNECTION_ID=fulfill_pageant
```

TrustGate reads Fivetran:

- `GET /v1/connections/{connectionId}`
- `GET /v1/connections/{connectionId}/schemas`
- `GET /v1/connections/{connectionId}/state` only when the connector supports it

For my Google Sheets connection, `/state` returned `405`, so the server treats that call as optional and keeps the receipt honest.

Test the evidence endpoint:

```bash
curl http://localhost:8080/api/fivetran/evidence
```

The important proof is the real Fivetran field inside the action receipt.

## Fivetran MCP

TrustGate also calls the official Fivetran MCP server (`github.com/fivetran/fivetran-mcp`) at runtime. The Cloud Run container installs that server, and the TrustGate backend spawns it over stdio and calls `list_connections` and `get_connection_details` to gather Fivetran evidence. Each action receipt includes an `evidence.fivetran_mcp` block, and there is a dedicated endpoint:

```bash
curl http://localhost:8080/api/fivetran/mcp-evidence
```

A healthy response shows `source: fivetran_mcp_live`, `details_verified: true`, and `target_connection_present: true` for `fulfill_pageant`.

To be precise about the architecture:

- The Gemini agent (Agent Builder) calls TrustGate's own agent-facing MCP endpoint, `/mcp` (`proposeTrustGateAction`).
- TrustGate's backend then calls the official Fivetran MCP server over stdio for Fivetran evidence.
- Fivetran REST remains the stable evidence path, so a decision still works if the MCP process is unavailable.

So TrustGate uses Fivetran REST and the official Fivetran MCP server at runtime. The MCP integration can be turned off with `FIVETRAN_MCP_DISABLED=1`, in which case TrustGate falls back to REST only.

## BigQuery Evidence

The hosted service queries the Fivetran-synced BigQuery table:

```text
trustgate-hackathon.trustgate_demo.customers
```

The receipt includes `evidence.bigquery.source`. Strong demo proof is:

```text
bigquery_rest_live
```

If the hosted endpoint says `bigquery_rest_live_partial`, BigQuery is readable but the Fivetran table column names do not include a recognizable `customer_tier`. The response includes `available_columns` so I can fix the sheet/schema instead of guessing.

Data freshness is a real signal, not a mock. TrustGate reads the Fivetran-managed `_fivetran_synced` column on the selected BigQuery row and reports `real_freshness_minutes` (minutes since the last sync) against the contract `freshness_sla_minutes`. When the age exceeds the SLA, the policy adds the `stale_sync_supporting_signal` rule. The demo `Simulate stale sync` button sets `freshness_simulated: true` and a stale age so the SLA breach can be shown on camera; the receipt always reports the real sync age alongside the simulated flag, so the simulation is never hidden.

The SLA is configurable with `FRESHNESS_SLA_MINUTES` (default 1440 = 24h) so it can match the connector's real sync cadence. For a clean baseline `ALLOW` during recording and the judging window, set the Fivetran connector to a scheduled sync that runs within the SLA (so the synced data stays recent); otherwise raise `FRESHNESS_SLA_MINUTES` to cover the gap. The `Simulate stale sync` injection always exceeds the SLA regardless of its value.

For local development without Google credentials, TrustGate falls back to clearly labeled `demo_bigquery_contract_query`.

## Gemini Tool Path

I first looked for an OpenAPI import in the Agent Builder / Agent Studio UI I had access to. That UI showed Google Search, URL Context, and MCP Server, but I did not see an OpenAPI import path.

Because the UI exposes an `Add tools -> MCP Server` path, TrustGate now serves its own MCP-compatible tool endpoint so the agent can be built directly in Vertex AI Agent Designer:

```text
POST /mcp
```

This endpoint speaks the MCP streamable-HTTP JSON-RPC protocol and exposes one tool, `proposeTrustGateAction`. In Agent Designer I add it with `Add tools -> MCP Server` and the endpoint URL, and the agent discovers the tool automatically.

To be precise about what this is: `/mcp` is TrustGate's own MCP tool surface for agents. It is not Fivetran's MCP server. Separately, the TrustGate backend spawns the official Fivetran MCP server over stdio and adds that result to receipts as `evidence.fivetran_mcp`.

The app also has a visible Gemini run endpoint:

```text
POST /api/agent/run
```

That endpoint calls Vertex AI Gemini, gives Gemini the `proposeTrustGateAction` function declaration, executes the requested TrustGate action on Cloud Run, and sends the receipt back to Gemini for the final explanation. The dashboard has a `Gemini Agent Run` panel so the video can show the actual function call, the TrustGate receipt, and Gemini's final answer in one place.

I also kept the original Cloud Shell script because it is useful as a separate terminal proof:

```bash
python scripts/vertex_trustgate_agent_demo.py
```

That script defines `proposeTrustGateAction`, lets `gemini-3.5-flash` request the tool call, POSTs the arguments to Cloud Run, and passes the TrustGate receipt back with `Part.from_function_response`.

Docs:

- `docs/vertex-agent-function-calling.md`
- `docs/cloud-run-agent-builder.md`
- `docs/openapi-tool.yaml`

## Deploy

```bash
scripts/deploy-cloud-run.sh
```

```powershell
.\scripts\deploy-cloud-run.ps1
```

The deploy scripts grant the Cloud Run service account BigQuery read/query roles and `roles/aiplatform.user` so the hosted `/api/agent/run` route can call Gemini from Cloud Run metadata auth.

If Cloud Run authentication is enabled later, the calling agent service account needs `roles/run.invoker`.

## Production Hardening Path

I am not calling this production-ready. The current build proves the runtime decision loop for the hackathon. The next engineering work I would do before trusting this in a real workflow:

- Protect mutating endpoints like `/api/actions/propose`, `/api/demo/*`, and `/api/approvals/*` with IAM, IAP, or signed internal calls.
- Move the policy and input contract into versioned config files instead of keeping all demo policy rules in `server.js`.
- Persist decision receipts and approval receipts in immutable storage so a later review can replay the same input, evidence, policy version, and result.
- Add Cloud Logging metrics for `ALLOW`, `APPROVAL_REQUIRED`, `BLOCK`, evidence failures, and Gemini tool-call failures.
- Add CI with syntax checks, policy tests, and secret scanning before every push.
- Replace the public demo reset/inject routes with a separate demo mode flag.

## Product Path

The hackathon version proves one refund authorization loop. The product version would become an action gateway for agents:

1. Authenticated agent identity on every proposed action.
2. Immutable audit receipts with replayable evidence, policy version, and contract version.
3. Versioned input contracts with policy tests before a contract can go live.
4. Enterprise approval workflow with approver identity, expiry, and scoped exceptions.
5. Multi-action support for refunds, discounts, account changes, invoice adjustments, and access changes.

That is the gap between this prototype and a product I would trust in a real business workflow.

## Claims I Am Not Making

- I am not claiming this is production-ready.
- I am not claiming TrustGate detects invisible meaning changes when schema and values are unchanged.
- I am not claiming the Gemini agent calls Fivetran's MCP directly. The agent calls TrustGate's `/mcp` tool; TrustGate's backend is the one that calls the official Fivetran MCP server (alongside Fivetran REST), shown as `fivetran_mcp_live` in the receipt.
- I am not claiming `risk_score` is ML.
- I am not claiming the Agent Builder UI OpenAPI path worked for me unless I record that exact path.
