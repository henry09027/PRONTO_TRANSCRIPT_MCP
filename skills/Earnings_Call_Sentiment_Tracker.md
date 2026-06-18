---
Name: earnings-sentiment-tracker
Purpose: Analyze and visualize how earnings call sentiment has evolved over time for a given company, sector, or country — producing a professional time-series chart and concise executive summary.
---

## Table of Contents

- [Inputs](#inputs)
- [Execution Steps](#execution-steps)
- [Output Format](#output-format)
- [Use Cases](#use-cases)
- [Interpretation Guide](#interpretation-guide)
- [Guardrails & Best Practices](#guardrails--best-practices)

---

## Inputs

| Parameter | Required | Type | Description |
|-----------|:--------:|------|-------------|
| `entity` | ✅ | string | Company name/ticker, sector name, or country code |
| `scope` | ✅ | enum | One of: `company`, `sector`, `country` |
| `timeframe` | ❌ | string | Lookback period (default: `5y`) |
| `sentiment_filter` | ❌ | enum | Filter to `positive` or `negative` only |
| `benchmark` | ❌ | string | Compare against a sector or index average |

---

## Execution Steps

### Step 1 — Resolve Entity

**Company scope:**

```text
Tool: pronto_mcp_getCompanies
Params: { companyNameOrTicker: "<user input>" }
Returns: companyId, sector, marketCap, country, ticker
```

If multiple results are returned, present the full list to the user for disambiguation before proceeding.

**Sector scope:**

```text
Tool: pronto_mcp_getFilterOptions
Params: { filterNames: ["sectors"] }
Returns: valid sector enum values
```

Match user input to the closest valid sector string.

**Country scope:**

```text
Tool: pronto_mcp_getFilterOptions
Params: { filterNames: ["countries"] }
Returns: valid country code enum values
```

---

### Step 2 — Retrieve Sentiment Timeline

**Company scope:**

```text
Tool: pronto_mcp_getCompanyTimeline
Params: {
  companiesIds: ["<companyId>"],
  dateRange: { gte: "now-5y/d", lte: "now" }
}
Returns: array of { transcriptId, title, date, sentiment, polarity: { positive, negative }, eventCount }
```

**Sector or Country scope:**

```text
Tool: pronto_mcp_getAnalytics
Params: {
  analyticsType: ["scores"],
  sectors: ["<sector>"],          // for sector scope
  countries: ["<countryCode>"],   // for country scope
  dateRange: { gte: "now-5y/d", lte: "now" }
}
```

Or for time-bucketed trend data:

```text
Tool: pronto_mcp_getTopicOvertime
Params: {
  topicSearchQuery: "<topic or leave empty for broad>",
  sectors: ["<sector>"],
  countries: ["<countryCode>"],
  timeframeInterval: "quarter",
  dateRange: { gte: "now-5y/d", lte: "now" }
}
```

---

### Step 3 — Structure the Data

Organize results into a time-series with derived fields:

| Field | Formula |
|-------|---------|
| Sentiment Score | Raw score from API (0.0–1.0) |
| Net Polarity | `(positive - negative) / (positive + negative)` |
| QoQ Change | `current_score - prior_period_score` |
| Rolling Avg (4Q) | Mean of last 4 quarters for trend smoothing |

---

### Step 4 — Generate Chart

Produce a **line chart** with the following specifications:

| Element | Specification |
|---------|---------------|
| X-axis | Earnings call dates (chronological, labeled by quarter) |
| Y-axis | Sentiment score (0.0 to 1.0) |
| Peak annotation | 🔺 label with date and score |
| Trough annotation | 🔻 label with date and score |
| Background bands | `0.0–0.4` red (bearish), `0.4–0.6` amber (neutral), `0.6–1.0` green (bullish) |
| Benchmark line | Dashed line if comparison entity provided |
| Styling | Clean, minimal, suitable for executive reporting |

---

### Step 5 — Generate Executive Summary

Produce **3–5 bullet points** covering:

1. **Overall trend direction** — improving, declining, cyclical, or stable
2. **Peak sentiment** — date, score, and likely business driver
3. **Trough sentiment** — date, score, and likely business driver
4. **Recent trajectory** — last 2–3 quarters momentum
5. **Notable pattern** — cyclicality, structural shifts, or peer divergence

---

## Output Format

```text
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 EARNINGS CALL SENTIMENT REPORT
   [Entity Name] | [Scope] | [Timeframe]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[TIME SERIES CHART]

━━━ KEY FINDINGS ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

• [Trend direction summary]
• [Peak: date, score, catalyst]
• [Trough: date, score, catalyst]
• [Recent momentum]
• [Forward-looking signal or risk flag]

━━━ DATA TABLE ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

| Period | Date | Score | +Stmt | -Stmt | QoQ Δ |
|--------|------|-------|-------|-------|-------|
| ...    | ...  | ...   | ...   | ...   | ...   |

(Last 8 quarters unless full history requested)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Use Cases

### 🏢 Company-Specific

**Example prompt:**

> "How has Tesla's earnings call sentiment changed over the past 3 years?"

**Workflow:**

1. `pronto_mcp_getCompanies` → resolve `Tesla` → companyId
2. `pronto_mcp_getCompanyTimeline` → 3-year lookback, returns per-call sentiment
3. Chart sentiment per call; annotate delivery milestones / margin events
4. Summarize trend, peak, trough, recent direction

**Value proposition:**

- Track management tone shifts ahead of stock inflection points
- Detect when guidance language turns cautious before numbers deteriorate
- Compare sentiment trajectory vs. stock price for divergence signals

---

### 🏭 Industry/Sector-Specific

**Example prompt:**

> "Compare sentiment across the Semiconductor sector over the last 2 years."

**Workflow:**

1. `pronto_mcp_getFilterOptions` → validate sector name
2. `pronto_mcp_getAnalytics` → `analyticsType: ["scores"]`, sector filter, quarterly
3. Optionally: `pronto_mcp_getSectors` to rank sub-sectors by topic relevance
4. Chart aggregate sector sentiment; overlay sub-sector breakdown

**Value proposition:**

- Identify sector-wide cyclical turning points (e.g., memory downturn bottoming)
- Compare individual company sentiment vs. sector average to find relative winners
- Screen for sectors where sentiment is inflecting before earnings revisions follow

---

### 🌍 Country-Specific

**Example prompt:**

> "What's the sentiment trend for South Korean vs. Japanese companies?"

**Workflow:**

1. `pronto_mcp_getFilterOptions` → validate country codes (`KR`, `JP`)
2. `pronto_mcp_getTopicOvertime` → filter by each country, `timeframeInterval: "quarter"`
3. Produce dual-line chart comparing KR vs. JP sentiment trajectories
4. Summarize relative momentum, divergence points, macro drivers

**Value proposition:**

- Gauge regional business confidence for cross-border allocation decisions
- Identify country-level sentiment divergences that precede relative equity performance
- Correlate with macro factors (FX, trade policy, central bank stance)

---

## Interpretation Guide

| Score Range | Label | Typical Characteristics |
|:-----------:|-------|------------------------|
| 0.80 – 1.00 | Very Bullish | Strong beats, raised guidance, new growth catalysts |
| 0.60 – 0.79 | Positive | Solid execution, constructive outlook |
| 0.40 – 0.59 | Neutral / Mixed | Balancing positives with acknowledged headwinds |
| 0.20 – 0.39 | Bearish | Misses, cost pressures, restructuring language |
| 0.00 – 0.19 | Very Bearish | Crisis-level negativity, material warnings |

---

## Guardrails & Best Practices

| Rule | Rationale |
|------|-----------|
| Require ≥ 4 data points before drawing trend conclusions | Avoids false signals from sparse data |
| Flag any single-quarter move > 0.20 | Likely event-driven (M&A, one-time charge, CEO change) rather than organic |
| Always report statement counts alongside scores | A 0.85 score on 50 statements is less reliable than on 400+ |
| Do not attribute causation to sentiment alone | Sentiment is a signal, not a forecast — pair with fundamentals |
| Validate all filter values via `pronto_mcp_getFilterOptions` before use | Prevents silent empty results from invalid enum values |
| Use `dateRange` with capital `M` for months | Lowercase `m` = minutes in Elasticsearch date math |

---

## Tool Reference

| Tool | Purpose | Key Parameters |
|------|---------|----------------|
| `pronto_mcp_getCompanies` | Resolve names/tickers to IDs | `companyNameOrTicker` |
| `pronto_mcp_getCompanyTimeline` | Per-document sentiment over time | `companiesIds`, `dateRange` |
| `pronto_mcp_getFilterOptions` | Get valid enum values for filters | `filterNames` |
| `pronto_mcp_getAnalytics` | Aggregated breakdowns by dimension | `analyticsType`, `sectors`, `countries` |
| `pronto_mcp_getTopicOvertime` | Time-series mentions + sentiment | `topicSearchQuery`, `timeframeInterval` |
| `pronto_mcp_getSectors` | Rank sectors by topic relevance | `topicSearchQuery` |
| `pronto_mcp_searchSentences` | Retrieve individual quoted sentences | `companiesIds`, `topicSearchQuery` |
| `pronto_mcp_getDocumentSummary` | AI summary of a specific document | `transcriptsIds`, `focus` |
