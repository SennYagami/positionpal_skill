---
name: agent-sse
description: Use SSE endpoints for agent analysis to avoid 15s proxy timeouts; includes request format and client handling notes.
---

# Agent SSE Usage

## When to use

Use SSE for agent analysis requests to avoid reverse proxy timeouts (e.g., Railway 15s limit). Prefer SSE whenever the analysis may call external exchanges/markets or take more than a few seconds.

## Endpoints

- `POST /api/analyze/stream` (public, body includes `api_key` and `api_secret`)
- `POST /api/agent/aggregate/analyze/stream` (agent, API key/secret in headers)

## Request format

### 1) Public analysis SSE

- URL: `POST /api/analyze/stream`
- Headers: `Content-Type: application/json`
- Body:
  - `api_key`: string
  - `api_secret`: string
  - `analysis_depth`: `basic` | `detailed` (optional)
  - `portfolio_only`: boolean (optional)

### 2) Agent aggregate SSE

- URL: `POST /api/agent/aggregate/analyze/stream`
- Headers:
  - `Content-Type: application/json`
  - `x-api-key`: string
  - `x-api-secret`: string
- Body:
  - `accountIds`: string[] (optional)
  - `analysis_depth`: `basic` | `detailed` (optional)
  - `portfolio_only`: boolean (optional)

## SSE response shape

The server streams JSON events using `text/event-stream`:

```
data: {"stage":"accounts","progress":5,"message":"Loaded 2 exchange accounts"}
data: {"stage":"fetch","progress":20,"message":"Fetching Binance (Main)"}
data: {"stage":"pricing","progress":50,"message":"Pricing spot assets"}
data: {"stage":"macro","progress":60,"message":"Fetching macro signals"}
data: {"stage":"scoring","progress":70,"message":"Scoring positions"}
data: {"stage":"done","progress":100,"report_id":"...","report":{...}}
```

On error:

```
data: {"stage":"error","error":"..."}
```

## Example curl

SSE with curl (keep the connection open):

```bash
curl -N -X POST "https://<host>/api/agent/aggregate/analyze/stream" \
  -H "Content-Type: application/json" \
  -H "x-api-key: <AGENT_KEY>" \
  -H "x-api-secret: <AGENT_SECRET>" \
  -d '{"analysis_depth":"basic","portfolio_only":true}'
```

## Client handling notes

- Always use `-N` in curl to disable buffering.
- Expect multiple `data:` frames; parse each line as JSON.
- If the stream ends without `stage: "done"`, treat as failure and surface the last `stage` + `message`.
*** End Patch}<> to=functions.apply_patch code
++ ...
