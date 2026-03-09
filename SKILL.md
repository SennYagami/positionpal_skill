---
name: agent-sse
description: Agent API guide for PositionPal to analyze portfolio risk via SSE streaming.
---

# PositionPal Agent API (SSE)

## What this skill is for

This skill documents **agent-only** API usage for PositionPal. It focuses on **SSE (Server‑Sent Events)** so agents can receive progress and results in a single streaming response.

Use this skill when:
- You are building or testing an agent client.
- You need streaming progress updates.
- You want the final report in one streamed response.

## Base URL

All endpoints in this skill use the production backend:

`https://loyal-celebration-production-e4d8.up.railway.app`

## Agent-only endpoints (SSE)

### 1) Aggregate analysis (SSE)

Analyze all connected exchange accounts for the authenticated user. Requires **agent API key + secret** in headers.

- Method: `POST`
- Path: `/api/agent/aggregate/analyze/stream`
- Headers:
  - `Content-Type: application/json`
  - `x-api-key`: `<AGENT_API_KEY>`
  - `x-api-secret`: `<AGENT_API_SECRET>`
- Body (JSON):
  - `accountIds`: `string[]` (optional) — limit analysis to specific accounts
  - `analysis_depth`: `"basic" | "detailed"` (optional)
  - `portfolio_only`: `boolean` (optional) — skip macro market signals

#### Curl example

```bash
curl -N -X POST "https://loyal-celebration-production-e4d8.up.railway.app/api/agent/aggregate/analyze/stream" \
  -H "Content-Type: application/json" \
  -H "x-api-key: <AGENT_API_KEY>" \
  -H "x-api-secret: <AGENT_API_SECRET>" \
  -d '{"analysis_depth":"basic","portfolio_only":true}'
```

## SSE response format

The server returns `text/event-stream` and emits multiple `data:` lines. Each line is JSON:

```
data: {"stage":"accounts","progress":5,"message":"Loaded 2 exchange accounts"}
data: {"stage":"fetch","progress":20,"message":"Fetching Binance (Main)"}
data: {"stage":"pricing","progress":50,"message":"Pricing spot assets"}
data: {"stage":"macro","progress":60,"message":"Fetching macro signals"}
data: {"stage":"scoring","progress":70,"message":"Scoring positions"}
data: {"stage":"finalizing","progress":90,"message":"Building report"}
data: {"stage":"warning","progress":95,"message":"Partial data: some accounts unavailable"}
data: {"stage":"done","progress":100,"report":{...}}
```

Notes on stages:
- `warning` is emitted when partial failures occur (e.g. one exchange API failed but analysis continues)
- `finalizing` precedes `done` during report assembly

On error:

```
data: {"stage":"error","error":"..."}
```

## Report data shape (PositionSafetyReport)

When `stage: "done"` arrives, the payload includes `report` with this structure:

```json
{
  "generated_at": "2026-02-27T04:07:01.727Z",
  "account_summary": {
    "total_assets_usd": 12345.67,
    "spot_value_usd": 8000.12,
    "futures_value_usd": 4345.55,
    "position_count": 12,
    "top_holdings": [
      { "symbol": "BTC", "value_usd": 5200.12, "allocation": 42.1 }
    ]
  },
  "macro_analysis": {
    "fear_greed_index": 62,
    "fear_greed_label": "Greed",
    "btc_trend": "bullish",
    "funding_rate_status": "neutral",
    "market_strength": 68
  },
  "positions": [
    {
      "symbol": "BTC",
      "side": "spot",
      "quantity": 0.12,
      "value_usd": 5200.12,
      "allocation_percent": 42.1,
      "risk_score": 58,
      "risk_level": "medium",
      "indicators": {
        "volume_ratio": 1.4,
        "funding_rate": 0.0001,
        "volatility_ratio": 1.1,
        "price_change_7d": -3.2,
        "price_change_30d": 12.5,
        "atr_pct": 0.042
      },
      "analysis": "Aggregated spot holding across 2 exchange(s).",
      "unrealized_pnl": 120.5
    }
  ],
  "overall_risk": {
    "overall_score": 57,
    "level": "cautious",
    "risk_factors": ["BTC score 58 (42.1%)"],
    "summary": "Aggregated 2 account(s). Largest exchange: binance (Main). Overall score 57/100."
  },
  "recommendations": [
    "Reduce concentration: BTC is 42.1% of portfolio.",
    "Consider lowering leverage / reducing perp exposure."
  ]
}
```

Notes:
- `macro_analysis` can be `null` if `portfolio_only` is `true`.
- `positions` includes both spot and perpetual positions; `side` is `spot | long | short`.
- `indicators` values may be `null` when data is unavailable.

## Client usage notes

- Always use `-N` in curl (disables buffering).
- Parse each `data:` line as a JSON event.
- The final result is sent with `stage: "done"` and includes `report`.
- If the stream ends before `done`, treat it as failure and surface the last event.
