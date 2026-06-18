---
name: Earnings Call Sentiment Analysis
description: This skill generates a professional sentiment analysis report from S&P Global earnings call transcript data.
---

## Workflow

1. **Resolve company** → `pronto_mcp_getCompanies` (get `companyId`)
2. **Get sentiment timeline** → `pronto_mcp_getCompanyTimeline` (returns per-document sentiment, polarity counts, and Pronto citation links)
3. **Get chart data** → SQL query against semantic model `PRONTOEC_ASPECTSANDTHEMES`
4. **Render chart** → `data_to_chart` with Vega-Lite spec
5. **Present output** → Chart + phased bullet-point summary with Pronto links

---

## SQL Template (Chart Data)

```sql
SELECT 
    a.calldate AS CALLDATE,
    a.score AS SENTIMENT_SCORE
FROM __prontoec_aspectsandthemes a
WHERE a.companyid = {{COMPANY_ID}}
    AND a.eventcategorygroup = 'Aggregate Weighted Score'
    AND a.eventcategory = 'Aggregate Weighted Score 4_2_1'
    AND a.documentsectionname = 'Total'
    AND a.isaggregate = TRUE
    AND a.calldate >= '{{START_DATE}}'
ORDER BY a.calldate ASC
```

Use semantic model `PRONTOEC_ASPECTSANDTHEMES` when executing.

---

## Chart Specification

```json
{
  "title": "{{COMPANY_NAME}} — Earnings Call Sentiment ({{YEAR_RANGE}})",
  "layer": [
    {
      "mark": {"type": "area", "opacity": 0.15, "color": "#1f77b4"},
      "encoding": {
        "x": {"field": "CALLDATE", "type": "temporal"},
        "y": {"field": "SENTIMENT_SCORE", "type": "quantitative", "scale": {"domain": [0, 1]}}
      }
    },
    {
      "mark": {"type": "line", "strokeWidth": 2.5, "color": "#1f77b4"},
      "encoding": {
        "x": {"field": "CALLDATE", "type": "temporal", "title": "Earnings Call Date", "axis": {"format": "%b %Y"}},
        "y": {"field": "SENTIMENT_SCORE", "type": "quantitative", "title": "Sentiment Score", "scale": {"domain": [0, 1]}, "axis": {"format": ".0%"}}
      }
    },
    {
      "mark": {"type": "point", "size": 60, "filled": true, "color": "#1f77b4"},
      "encoding": {
        "x": {"field": "CALLDATE", "type": "temporal"},
        "y": {"field": "SENTIMENT_SCORE", "type": "quantitative", "scale": {"domain": [0, 1]}},
        "tooltip": [
          {"field": "CALLDATE", "type": "temporal", "title": "Call Date", "format": "%b %d, %Y"},
          {"field": "SENTIMENT_SCORE", "type": "quantitative", "title": "Sentiment Score", "format": ".1%"}
        ]
      }
    }
  ]
}
```

---

## Output Format

Output **only** the chart and a phased bullet-point summary. No lengthy paragraphs.

### Structure:

1. **Chart** (rendered via `data_to_chart`)
2. **Phased summary** — group the timeline into 2–4 distinct phases based on sentiment shifts:
   - Phase name and date range
   - Average score for the phase
   - Key highs/lows with scores, positive/negative statement counts, and Pronto document links (from `getCompanyTimeline` results)
   - Brief context (1 line max per call)
3. **Key Takeaways** — 3–5 bullets:
   - Current score vs. period average
   - All-time high and low (with dates)
   - Recovery speed if applicable
   - Dominant driver of current sentiment

### Citation Links

Always include Pronto document links from `getCompanyTimeline` results as markdown links, e.g.:
```
[Q2 2024 Earnings Call (Jul 30, 2024)](https://spglobal.prontonlp.com/#/ref/$DOCID_T000003204698)
```

---

## Variant: Section Breakdown (Management vs. Analyst)

```sql
SELECT 
    a.calldate AS CALLDATE,
    a.documentsectionname AS SECTION,
    a.score AS SENTIMENT_SCORE
FROM __prontoec_aspectsandthemes a
WHERE a.companyid = {{COMPANY_ID}}
    AND a.eventcategorygroup = 'Aggregate Weighted Score'
    AND a.eventcategory = 'Aggregate Weighted Score 4_2_1'
    AND a.documentsectionname IN ('CFO', 'Question')
    AND a.isaggregate = TRUE
    AND a.calldate >= '{{START_DATE}}'
ORDER BY a.calldate ASC
```

Add to chart encoding:
```json
"color": {"field": "SECTION", "type": "nominal", "title": "Speaker Section"}
```

---

## Variant: Peer Comparison

```sql
SELECT 
    a.calldate AS CALLDATE,
    c.companyname AS COMPANY,
    a.score AS SENTIMENT_SCORE
FROM __prontoec_aspectsandthemes a
INNER JOIN __ciqcompany c ON a.companyid = c.companyid
WHERE a.companyid IN ({{ID_1}}, {{ID_2}}, {{ID_3}})
    AND a.eventcategorygroup = 'Aggregate Weighted Score'
    AND a.eventcategory = 'Aggregate Weighted Score 4_2_1'
    AND a.documentsectionname = 'Total'
    AND a.isaggregate = TRUE
    AND a.calldate >= '{{START_DATE}}'
ORDER BY a.calldate ASC
```

Add to chart encoding:
```json
"color": {"field": "COMPANY", "type": "nominal", "title": "Company"}
```

---

## Notes

- Scores are 0–1 scale from the ATC LLM model
- `eventcategory = 'Aggregate Weighted Score 4_2_1'` is the headline sentiment metric (weighted by importance, all polarities)
- `documentsectionname = 'Total'` = full-call aggregate; `'CFO'` / `'Question'` for section-level
- `isaggregate = TRUE` filters to pre-aggregated records — no need for AVG/GROUP BY
- Default time range: 5 years unless user specifies otherwise
