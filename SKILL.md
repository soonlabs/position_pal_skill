# Skill: position_safety_monitor

Analyze a Binance account for position risk. Returns a structured safety report covering account summary, macro market conditions, per-position risk scores, and actionable recommendations.

---

## Base URL

```
BASE_URL = https://loyal-celebration-production-e4d8.up.railway.app
```

All endpoints below are relative to this base.

---

## Quick Start

```bash
curl -s -X POST "${BASE_URL}/api/analyze" \
  -H "Content-Type: application/json" \
  -d '{"api_key":"<KEY>","api_secret":"<SECRET>","analysis_depth":"basic"}' \
  | jq '.report.overall_risk'
```

---

## Endpoints

### 1. POST /api/analyze — Run Full Analysis

Triggers a full synchronous analysis. Blocks until complete.

**Request body**

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `api_key` | string | yes | — | Binance API key (read-only permissions) |
| `api_secret` | string | yes | — | Binance API secret |
| `analysis_depth` | `"basic"` \| `"detailed"` | no | `"basic"` | `basic` = rule-based; `detailed` = OpenAI-enhanced text |
| `portfolio_only` | boolean | no | `false` | Skip macro data (Fear & Greed, BTC trend). Faster. |

**Response: `AnalyzeResponse`**

```json
{
  "report_id": "uuid-string",
  "report": { ... }
}
```

**Presenting the result:** The report has many fields. See **[How to use and present the analyze report](#how-to-use-and-present-the-analyze-report)** for what to show first (overall risk → summary → recommendations) and how to format it for users.

**curl example**

```bash
curl -s -X POST "${BASE_URL}/api/analyze" \
  -H "Content-Type: application/json" \
  -d '{
    "api_key": "<KEY>",
    "api_secret": "<SECRET>",
    "analysis_depth": "basic",
    "portfolio_only": false
  }' | jq '.report.overall_risk'
```

---

### 2. POST /api/analyze/stream — Streaming Analysis (SSE)

Same analysis as `/api/analyze` but streams progress events via **Server-Sent Events** as the analysis runs. Use this when you want real-time progress feedback or want to start rendering results early.

**Request body** — identical to `/api/analyze`

**Response**: `text/event-stream`. Each event is a JSON line prefixed with `data: `.

**Event schema**

| Event type | Fields | Notes |
|---|---|---|
| Progress | `stage`, `progress` (0–100), `message` | Emitted throughout analysis |
| Done | `stage: "done"`, `progress: 100`, `report_id`, `report` | Final event with full report |
| Error | `stage: "error"`, `error` | Emitted on failure |

**Progress stages (in order):**

| `stage` | `progress` | Description |
|---|---|---|
| `balances` | 15 | Account balances fetched |
| `macro` | 25 | Fear & Greed + BTC macro data fetched |
| `indicators` | 25–88 | Per-position indicator fetch (scales with position count) |
| `risk` | 88 | Portfolio risk computed |
| `complete` | 95 | Recommendations generated |
| `done` | 100 | Full report ready |

**curl example** (stream raw events to terminal)

```bash
curl -s -N -X POST "${BASE_URL}/api/analyze/stream" \
  -H "Content-Type: application/json" \
  -d '{"api_key":"<KEY>","api_secret":"<SECRET>","analysis_depth":"basic"}'
```

**JavaScript consumer**

```javascript
const res = await fetch(`${BASE_URL}/api/analyze/stream`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ api_key, api_secret, analysis_depth: 'basic' })
});

const reader = res.body.getReader();
const decoder = new TextDecoder();
let buffer = '';

while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  buffer += decoder.decode(value, { stream: true });
  const lines = buffer.split('\n');
  buffer = lines.pop() ?? '';
  for (const line of lines) {
    if (!line.startsWith('data: ')) continue;
    const event = JSON.parse(line.slice(6));
    if (event.stage === 'done') {
      console.log('Report ready:', event.report.overall_risk);
    } else {
      console.log(`[${event.progress}%] ${event.message}`);
    }
  }
}
```

**Python consumer**

```python
import requests, json

url = f"{BASE_URL}/api/analyze/stream"
payload = {"api_key": KEY, "api_secret": SECRET, "analysis_depth": "basic"}

with requests.post(url, json=payload, stream=True) as r:
    for line in r.iter_lines():
        if not line or not line.startswith(b"data: "):
            continue
        event = json.loads(line[6:])
        if event["stage"] == "done":
            report = event["report"]
            print("Overall risk:", report["overall_risk"]["level"])
            break
        else:
            print(f"[{event['progress']}%] {event['message']}")
```

---

### 3. GET /api/report/:id — Fetch Cached Report

Retrieve a previously generated report by its ID.

```bash
curl -s "${BASE_URL}/api/report/<report_id>" | jq '.report.overall_risk'
```

Returns `ReportStoreEntry`:

```json
{
  "id": "uuid-string",
  "created_at": "2026-02-25T10:00:00.000Z",
  "report": { ... }
}
```

Returns `404` with `{ "code": "E004", "message": "Report not found" }` if ID is unknown or expired.

---

### 4. POST /api/portfolio — Raw Portfolio Snapshot

Fast portfolio snapshot with all holdings, live prices, and unrealized PnL — without risk scoring. Ideal for "what do I hold and what is it worth?" queries.

**curl example**

```bash
curl -s -X POST "${BASE_URL}/api/portfolio" \
  -H "Content-Type: application/json" \
  -d '{"api_key":"<KEY>","api_secret":"<SECRET>"}' \
  | jq '{total: .summary.total_assets_usd, spot_count: (.spot | length), futures_count: (.futures | length)}'
```

**Request body**

| Field | Type | Required | Description |
|---|---|---|---|
| `api_key` | string | yes | Binance API key |
| `api_secret` | string | yes | Binance API secret |

**Response: `PortfolioSnapshot`**

```typescript
{
  generated_at: string;
  summary: {
    total_assets_usd: number;
    spot_value_usd: number;
    futures_value_usd: number;
    asset_count: number;
  };
  spot: SpotHolding[];       // sorted by value desc; assets < $1 excluded
  futures: FuturesHolding[]; // sorted by value desc
}
```

**`SpotHolding`**

```typescript
{
  asset: string;               // e.g. "BTC", "ETH", "USDT"
  quantity: number;
  price_usd: number;
  value_usd: number;
  allocation_percent: number;  // share of total portfolio (0-100)
}
```

**`FuturesHolding`**

```typescript
{
  symbol: string;              // e.g. "BTC", "ETH"
  side: "long" | "short";
  contracts: number;
  entry_price: number;
  mark_price: number;          // current mark price
  value_usd: number;           // contracts × mark_price
  leverage: number;
  allocation_percent: number;
  unrealized_pnl: number;      // positive = profit, negative = loss
  liquidation_price: number;
}
```

**When to use vs `/api/analyze`:**
- Use `/api/portfolio` for fast live balances and P&L (no risk scores, no indicators)
- Use `/api/analyze` when you need risk scores, RSI/OI indicators, and recommendations

**Data consistency (important for agents):**  
`POST /api/portfolio` and the `report.account_summary` inside `POST /api/analyze` are **the same data source**. Both are built from one portfolio snapshot. So:
- `summary.total_assets_usd` (portfolio) **equals** `report.account_summary.total_assets_usd` (analyze).
- `report.account_summary.top_holdings` is the same breakdown as portfolio’s `spot` + `futures` (top 5 by value).
- If you call both endpoints with the same credentials, totals and allocations must match. Prefer **one** of them for “current holdings and total value”: either use `/api/portfolio` for a quick snapshot, or use `report.account_summary` when you already have an analysis report. Do not treat them as two different “sources of truth” or report conflicting numbers.

---

### 5. POST /api/check — Threshold Alert

Run a full analysis and immediately evaluate whether any risk threshold is breached. Designed for automated monitoring — returns a single `triggered: boolean` without requiring the caller to parse the full report.

**curl example**

```bash
curl -s -X POST "${BASE_URL}/api/check" \
  -H "Content-Type: application/json" \
  -d '{
    "api_key": "<KEY>",
    "api_secret": "<SECRET>",
    "alert_if_score_above": 60
  }' | jq '{triggered: .triggered, score: .score, level: .level}'
```

**Request body**

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `api_key` | string | yes | — | Binance API key |
| `api_secret` | string | yes | — | Binance API secret |
| `alert_if_score_above` | number | no | `60` | Trigger if `overall_score` exceeds this value |

**Response: `CheckResponse`**

```typescript
{
  triggered: boolean;       // true if overall_score > alert_if_score_above
  score: number;            // overall_risk.overall_score (0-100)
  level: string;            // "safe" | "cautious" | "risky" | "dangerous"
  top_risks: string[];      // up to 3 highest-risk positions, e.g. ["BTC score 74"]
  report_id: string;        // use to retrieve full report via GET /api/report/:id
}
```

**Python monitoring script**

```python
import requests

def check_portfolio(api_key, api_secret, threshold=60):
    r = requests.post(f"{BASE_URL}/api/check", json={
        "api_key": api_key,
        "api_secret": api_secret,
        "alert_if_score_above": threshold
    })
    result = r.json()
    if result["triggered"]:
        print(f"ALERT: score={result['score']} ({result['level']})")
        print("Top risks:", ", ".join(result["top_risks"]))
    return result
```

---

### 6. POST /api/chat — Conversational Interface

Natural-language Q&A about portfolio risk. Accepts an optional `context` array for multi-turn conversations so follow-up questions retain prior context.

**curl example (single turn)**

```bash
curl -s -X POST "${BASE_URL}/api/chat" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "How risky is my portfolio right now?",
    "api_key": "<KEY>",
    "api_secret": "<SECRET>",
    "analysis_depth": "basic"
  }' | jq '.content'
```

**Request body**

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `api_key` | string | yes | — | Binance API key |
| `api_secret` | string | yes | — | Binance API secret |
| `message` | string | yes | — | User question |
| `analysis_depth` | `"basic"` \| `"detailed"` | no | `"basic"` | `detailed` uses OpenAI for richer answers |
| `context` | `ChatMessage[]` | no | `[]` | Prior conversation turns for multi-turn chat |

**`ChatMessage`**

```typescript
{ role: "user" | "assistant"; content: string; }
```

**Response**

```json
{
  "role": "assistant",
  "content": "Overall risk score 12 (safe). Top holding is BTC at 38.50%."
}
```

**Multi-turn example**

```javascript
const context = [];

async function chat(message) {
  const res = await fetch(`${BASE_URL}/api/chat`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ message, api_key, api_secret, analysis_depth: 'detailed', context })
  });
  const { content } = await res.json();
  context.push({ role: 'user', content: message });
  context.push({ role: 'assistant', content });
  return content;
}

await chat("How risky is my portfolio?");
await chat("Which position should I reduce first?");
await chat("What would happen if BTC drops 20%?");
```

---

## Report Structure

### Top Level: `PositionSafetyReport`

```typescript
{
  generated_at: string;             // ISO 8601 timestamp
  account_summary: AccountSummary;
  macro_analysis: MacroAnalysis | null;  // null when portfolio_only=true
  positions: PositionAnalysis[];    // sorted by risk_score descending
  overall_risk: RiskAssessment;
  recommendations: string[];        // 1–3 action items
}
```

---

### `AccountSummary`

**Same data as `POST /api/portfolio`:** totals and holdings come from the same snapshot. Use this when you already have a report; use `/api/portfolio` when you only need balances.

```typescript
{
  total_assets_usd: number;   // same as portfolio summary.total_assets_usd
  spot_value_usd: number;
  futures_value_usd: number;
  position_count: number;
  top_holdings: {
    symbol: string;           // asset (spot) or symbol (futures), e.g. "BTC", "USDT"
    value_usd: number;
    allocation: number;       // 0-100, share of total_assets_usd
  }[];                        // top 5 by value (spot + futures combined)
}
```

---

### `MacroAnalysis`

`null` when `portfolio_only: true`.

```typescript
{
  fear_greed_index: number;         // 0–100 (source: alternative.me)
  fear_greed_label: "Extreme Fear" | "Fear" | "Neutral" | "Greed" | "Extreme Greed";
  btc_trend: "bullish" | "bearish" | "sideways";
  funding_rate_status: "neutral" | "overheated" | "cold";
  market_strength: number;          // 0–100 composite score
}
```

---

### `PositionAnalysis`

One entry per asset. Sorted highest risk first.

```typescript
{
  symbol: string;
  side: "spot" | "long" | "short";
  quantity: number;
  value_usd: number;
  allocation_percent: number;       // 0-100
  risk_score: number;               // 0–100
  risk_level: "low" | "medium" | "high" | "critical";
  indicators: {
    rsi_14: number;                 // >70 overbought, <30 oversold
    oi_change_24h: number;          // open interest change ratio; >0.2 = warning
    volume_ratio: number;           // 24h vol / 20-day avg; >2 = unusual spike
    funding_rate: number;           // perpetual funding rate; >0.0001 = overheated
    volatility_ratio: number;       // ATR(14)/price; >0.05 = high volatility
  };
  analysis: string;
  unrealized_pnl?: number;          // futures only; positive = profit
}
```

**Risk level thresholds**

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
  overall_score: number;            // 0–100 weighted portfolio risk
  level: "safe" | "cautious" | "risky" | "dangerous";
  risk_factors: string[];           // up to 3 highest-risk positions
  summary: string;
}
```

**Overall level thresholds:** safe ≤30 / cautious ≤60 / risky ≤80 / dangerous >80

---

## How to use and present the analyze report

The analyze response is large. Use the following guidance so users get a clear, scannable summary instead of a data dump.

### 1. What to show, in order

| Priority | Source | What to show |
|----------|--------|----------------|
| **First** | `report.overall_risk` | One line: level + score. Example: *"Portfolio risk: **cautious** (score 42)."* |
| **Second** | `report.account_summary` | One line: total value + top 1–2 holdings. Example: *"Total **$12,450** — BTC 65%, ETH 22%."* |
| **Third** | `report.overall_risk.risk_factors` | Only if non-empty: *"Focus: BTC score 74, ETH score 61."* |
| **Fourth** | `report.recommendations` | Always show: 1–3 short bullets. |
| **Optional** | `report.positions` | Top 3–5 by risk (e.g. `positions.slice(0, 5)`). Do not dump all positions unless the user asks for a full list. |
| **Optional** | `report.macro_analysis` | One line if present: *"Market: Fear & Greed 35 (Fear), BTC trend sideways."* |

### 2. Quick summary template

After calling `/api/analyze`, you can answer “How is my portfolio?” with:

- **Headline:** `overall_risk.level` + `overall_risk.overall_score`.
- **Portfolio:** `account_summary.total_assets_usd` (format as USD) and top 1–2 from `account_summary.top_holdings` (symbol + allocation %).
- **If any risk_factors:** list them in one line.
- **Actions:** `recommendations[]` as short bullets.

Example: *"Risk is **cautious** (42). Portfolio **$12,450** — BTC 65%, ETH 22%. Watch: BTC score 74. Recommendations: Monitor BTC closely; consider reducing if volatility rises."*

### 3. When to show more detail

- **Per-position detail:** Use `positions[i]` only for the **top 3–5 by risk** (array is already sorted by `risk_score`). For each, show: symbol, side, risk_level, value_usd, and optionally `analysis` (one sentence). Skip raw `indicators` unless the user asks for RSI/funding/volatility.
- **Macro:** Show `macro_analysis` in one line (fear_greed_label, btc_trend, funding_rate_status). Omit if `portfolio_only: true` (then `macro_analysis` is null).
- **Full report:** Only dump all positions and all fields when the user explicitly asks for “full details” or “every position.”

### 4. Formatting

- **Money:** `value_usd` → `$1,234.56` (two decimals; optional thousands separator).
- **Percent:** `allocation`, `allocation_percent` → `65.5%` (one or two decimals).
- **Risk:** Prefer the **level** label (`safe` / `cautious` / `risky` / `dangerous`; or for positions `low` / `medium` / `high` / `critical`) over the raw score when talking to the user; show score in parentheses when relevant (e.g. “high (74)”).
- **Lists:** Use bullets or a short table; avoid long paragraphs of numbers.

### 5. What not to do

- Do not return the raw JSON.
- Do not list every position by default — summarize with top holdings and top risks.
- Do not show `report_id`, `generated_at`, or internal fields unless the user asks (e.g. for caching or debugging).
- Do not repeat the same numbers from both `account_summary` and a full position list; pick one (prefer `account_summary` for totals and allocation).

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

## Agent Workflow Patterns

### Pattern 1: Daily Monitoring (Lightweight)

Use `/api/check` for cheap daily monitoring. Only fetch the full report when an alert fires.

```python
import requests, json
from datetime import datetime

BASE_URL = "https://loyal-celebration-production-e4d8.up.railway.app"

def daily_monitor(api_key, api_secret, threshold=65):
    """Run once per day. Only deep-dives when triggered."""
    check = requests.post(f"{BASE_URL}/api/check", json={
        "api_key": api_key,
        "api_secret": api_secret,
        "alert_if_score_above": threshold
    }).json()

    print(f"[{datetime.now():%H:%M}] Score={check['score']} level={check['level']}")

    if not check["triggered"]:
        return  # Portfolio is healthy, nothing to do

    # Threshold breached — fetch full report for details
    print("ALERT:", ", ".join(check["top_risks"]))
    report_data = requests.get(f"{BASE_URL}/api/report/{check['report_id']}").json()
    report = report_data["report"]

    # Surface the highest-risk positions
    critical = [p for p in report["positions"] if p["risk_level"] == "critical"]
    for pos in critical:
        print(f"  CRITICAL: {pos['symbol']} score={pos['risk_score']} "
              f"RSI={pos['indicators']['rsi_14']:.0f}")

    # Show recommendations
    for rec in report["recommendations"]:
        print(f"  Action: {rec}")

    return check["report_id"]
```

---

### Pattern 2: Position Deep-Dive

Use `/api/analyze` with `analysis_depth: "detailed"` for a thorough analysis of a specific situation, then follow up via `/api/chat` for interactive Q&A.

```python
import requests

BASE_URL = "https://loyal-celebration-production-e4d8.up.railway.app"

def deep_dive(api_key, api_secret):
    # Step 1: Full detailed analysis
    result = requests.post(f"{BASE_URL}/api/analyze", json={
        "api_key": api_key,
        "api_secret": api_secret,
        "analysis_depth": "detailed"
    }).json()

    report = result["report"]
    report_id = result["report_id"]

    # Step 2: Print top 3 highest-risk positions
    for pos in report["positions"][:3]:
        print(f"{pos['symbol']} ({pos['side']}) — score {pos['risk_score']} [{pos['risk_level']}]")
        print(f"  {pos['analysis']}")
        if pos.get("unrealized_pnl") is not None:
            pnl = pos["unrealized_pnl"]
            print(f"  PnL: {'+'if pnl >= 0 else ''}{pnl:.2f} USD")

    # Step 3: Multi-turn chat for interactive Q&A
    context = []
    questions = [
        "Which position has the highest liquidation risk?",
        "Should I reduce my leverage given current market conditions?",
        "What is your recommended action for the next 24 hours?"
    ]

    for q in questions:
        resp = requests.post(f"{BASE_URL}/api/chat", json={
            "message": q,
            "api_key": api_key,
            "api_secret": api_secret,
            "analysis_depth": "detailed",
            "context": context
        }).json()
        print(f"Q: {q}")
        print(f"A: {resp['content']}\n")
        context.append({"role": "user", "content": q})
        context.append({"role": "assistant", "content": resp["content"]})

    return report_id
```

---

### Pattern 3: Fast Portfolio Snapshot

When you only need current holdings without risk analysis (e.g., to display balances or compute custom metrics):

```bash
# Get total portfolio value and list all spot holdings
curl -s -X POST "${BASE_URL}/api/portfolio" \
  -H "Content-Type: application/json" \
  -d '{"api_key":"<KEY>","api_secret":"<SECRET>"}' \
  | jq '{
      total_usd: .summary.total_assets_usd,
      spot: [.spot[] | {asset: .asset, value: .value_usd, pct: .allocation_percent}],
      futures_pnl: [.futures[] | {symbol: .symbol, pnl: .unrealized_pnl, liq: .liquidation_price}]
    }'
```

---

### Pattern 4: SSE Streaming with Progress Display

For interactive agents that want to show progress to users in real time:

```python
import requests, json, sys

BASE_URL = "https://loyal-celebration-production-e4d8.up.railway.app"

def analyze_streaming(api_key, api_secret):
    payload = {"api_key": api_key, "api_secret": api_secret, "analysis_depth": "basic"}

    with requests.post(f"{BASE_URL}/api/analyze/stream", json=payload, stream=True) as r:
        for line in r.iter_lines():
            if not line or not line.startswith(b"data: "):
                continue
            event = json.loads(line[6:])

            if event["stage"] == "done":
                report = event["report"]
                print(f"\nDone. Overall risk: {report['overall_risk']['level']} "
                      f"(score {report['overall_risk']['overall_score']})")
                return report

            elif event["stage"] == "error":
                print(f"Error: {event.get('error', 'Unknown error')}")
                return None

            else:
                bar = "#" * (event["progress"] // 5)
                sys.stdout.write(f"\r[{bar:<20}] {event['progress']:3d}%  {event['message']:<40}")
                sys.stdout.flush()
```

---

## Performance Notes

- **Analysis time scales with portfolio size.** Each held asset requires 3 Binance API calls (klines, funding rate, open interest). Expect ~0.5–1s per asset.
- `portfolio_only: true` skips Fear & Greed and BTC macro fetches, saving 1–2 seconds.
- `analysis_depth: "basic"` avoids OpenAI; use `"detailed"` only when richer text is needed.
- `/api/check` is the fastest way to determine if action is required — use it for scheduled monitoring; `/api/analyze` only when an alert fires.
- `/api/analyze/stream` has the same total latency as `/api/analyze` but lets you display progress in real time.
- Reports are cached in memory. `report_id` is valid for the lifetime of the server process.

---

## Security

- API keys are **never persisted** — used in memory during analysis only, then discarded.
- Only **read-only** Binance permissions are required. Never provision keys with withdrawal or trading permissions.
- Required Binance key permissions: `Read Info` (spot and futures account data).
