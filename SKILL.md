# Skill: position_safety_monitor

Analyze a Binance account for position risk. Returns a structured safety report covering account summary, macro market conditions, per-position risk scores, and actionable recommendations.

---

## Base URL

```
BASE_URL = https://loyal-celebration-production-e4d8.up.railway.app/
```

All endpoints below are relative to this base. Replace `{BASE_URL}` with the value above throughout this document.

---

## Quick Start

```
POST {BASE_URL}/api/analyze
Content-Type: application/json

{
  "api_key": "<BINANCE_READ_ONLY_API_KEY>",
  "api_secret": "<BINANCE_API_SECRET>",
  "analysis_depth": "basic"
}
```

The response contains both a `report_id` (for later retrieval) and the full `report` inline.

---

## Endpoints

### 1. POST /api/analyze — Run Analysis

Triggers a full analysis. **This is the primary endpoint.**

**Request body**

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `api_key` | string | yes | — | Binance API key (read-only permissions sufficient) |
| `api_secret` | string | yes | — | Binance API secret |
| `analysis_depth` | `"basic"` \| `"detailed"` | no | `"basic"` | `basic` uses local rule-based analysis; `detailed` calls OpenAI for richer text |
| `portfolio_only` | boolean | no | `false` | Skip macro market data (Fear & Greed, BTC trend). Faster, but no market context |

**Response: `AnalyzeResponse`**

```json
{
  "report_id": "uuid-string",
  "report": { ... }
}
```

Save `report_id` if you want to retrieve the report later via `GET /api/report/:id`.

---

### 2. GET /api/report/:id — Fetch Cached Report

Retrieve a previously generated report by its ID.

```
GET {BASE_URL}/api/report/<report_id>
```

Returns the same `report` object wrapped in a `ReportStoreEntry`:

```json
{
  "id": "uuid-string",
  "created_at": "2026-02-25T10:00:00.000Z",
  "report": { ... }
}
```

Returns `404` with `{ "code": "E004", "message": "Report not found" }` if ID is unknown or expired.

---

### 3. POST /api/chat — Conversational Interface

Get a one-sentence natural-language summary of the portfolio risk.

```
POST {BASE_URL}/api/chat
```

```json
{
  "message": "How risky is my portfolio?",
  "api_key": "<KEY>",
  "api_secret": "<SECRET>",
  "analysis_depth": "basic"
}
```

Response:

```json
{
  "role": "assistant",
  "content": "Overall risk score 12 (safe). Top holding is BTC at 38.50%."
}
```

Use `POST /api/analyze` instead when you need structured data.

---

## Report Structure

### Top Level: `PositionSafetyReport`

```typescript
{
  generated_at: string;            // ISO 8601 timestamp
  account_summary: AccountSummary;
  macro_analysis: MacroAnalysis | null;  // null when portfolio_only=true
  positions: PositionAnalysis[];   // sorted by risk_score descending
  overall_risk: RiskAssessment;
  recommendations: string[];       // 1-3 action items
}
```

---

### `AccountSummary`

```typescript
{
  total_assets_usd: number;        // total portfolio value in USD
  spot_value_usd: number;          // value of spot holdings
  futures_value_usd: number;       // notional value of futures positions
  position_count: number;          // total number of assets held
  top_holdings: [
    {
      symbol: string;              // e.g. "BTC"
      value_usd: number;
      allocation: number;          // percentage of total portfolio (0-100)
    }
  ];                               // top 5 by value
}
```

---

### `MacroAnalysis`

`null` when `portfolio_only: true`.

```typescript
{
  fear_greed_index: number;        // 0–100 (source: alternative.me)
  fear_greed_label: "Extreme Fear" | "Fear" | "Neutral" | "Greed" | "Extreme Greed";
  btc_trend: "bullish" | "bearish" | "sideways";  // based on last 2 daily closes
  funding_rate_status: "neutral" | "overheated" | "cold";  // BTC perpetual funding
  market_strength: number;         // 0–100 composite score
}
```

---

### `PositionAnalysis`

One entry per asset held (both spot and futures). Sorted highest risk first.

```typescript
{
  symbol: string;                  // e.g. "ETH", "SOL"
  side: "spot" | "long" | "short";
  quantity: number;
  value_usd: number;
  allocation_percent: number;      // share of total portfolio (0-100)
  risk_score: number;              // 0–100
  risk_level: "low" | "medium" | "high" | "critical";
  indicators: {
    rsi_14: number;                // RSI(14) — >70 overbought, <30 oversold
    oi_change_24h: number;         // open interest change ratio — >0.2 = warning
    volume_ratio: number;          // 24h vol / 20-day avg — >2 = unusual spike
    funding_rate: number;          // perpetual funding rate — >0.0001 = overheated
    volatility_ratio: number;      // ATR(14)/price — >0.05 = high volatility
  };
  analysis: string;                // human-readable risk explanation
}
```

**Risk level thresholds:**

| Score | Level | Meaning |
|---|---|---|
| 0–30 | `low` | Safe to hold |
| 31–60 | `medium` | Monitor closely |
| 61–80 | `high` | Consider reducing |
| 81–100 | `critical` | Urgent action advised |

---

### `RiskAssessment`

```typescript
{
  overall_score: number;           // 0–100 weighted portfolio risk
  level: "safe" | "cautious" | "risky" | "dangerous";
  risk_factors: string[];          // up to 3 highest-risk positions
  summary: string;                 // one-line portfolio summary
}
```

**Overall level thresholds:** safe ≤30 / cautious ≤60 / risky ≤80 / dangerous >80

---

## Error Responses

All errors return `{ "code": string, "message": string }`.

| HTTP | Code | Meaning | Fix |
|---|---|---|---|
| 400 | `E001` | Missing or invalid API key | Check key format and permissions |
| 400 | `E002` | Invalid API secret | Verify secret; ensure key/secret match |
| 404 | `E004` | Report ID not found | Re-run `/api/analyze` |
| 429 | `E005` | Binance rate limit hit | Wait and retry |
| 504 | `E003` | Network timeout | Retry; check connectivity to Binance |

---

## Agent Workflow

**Typical pattern:**

```
1. Call POST {BASE_URL}/api/analyze with credentials
2. Read report.overall_risk.level to assess urgency
3. Check report.positions (sorted by risk_score desc) for top risks
4. Read report.recommendations for suggested actions
5. Optionally read report.macro_analysis for market context
6. Use report_id to retrieve same report later if needed
```

**Example: check if portfolio needs immediate attention**

```python
BASE_URL = "https://positionpalbackend-production.up.railway.app"

response = POST f"{BASE_URL}/api/analyze" { api_key, api_secret, analysis_depth: "basic" }
level = response.report.overall_risk.level

if level in ("risky", "dangerous"):
    # highlight high-risk positions
    critical = [p for p in response.report.positions if p.risk_level == "critical"]
    # surface recommendations
    actions = response.report.recommendations
```

---

## Performance Notes

- **Analysis time scales with portfolio size.** Each held asset requires 3 Binance API calls (klines, funding rate, open interest). Expect ~0.5–1s per asset after initial market data load.
- `portfolio_only: true` skips the Fear & Greed API call and macro BTC analysis, saving 1–2 seconds.
- `analysis_depth: "basic"` avoids OpenAI and is much faster; use `"detailed"` only when richer text analysis is needed.
- Reports are cached in memory. The `report_id` is valid for the lifetime of the process.

---

## Security

- API keys are **never persisted** — used in memory during analysis only, then discarded.
- Only **read-only** Binance permissions are required. Do not provision keys with withdrawal or trading permissions.
- Required Binance key permissions: `Read Info` (spot and futures account data).
