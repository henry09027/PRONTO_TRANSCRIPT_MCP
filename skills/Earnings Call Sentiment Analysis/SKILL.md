---
name: Earnings Call Sentiment Analysis
description: This skill generates a professional sentiment analysis report from S&P Global earnings call transcript data.
---

## Data Source

- **Table**: `QRSLLM_POC_DB.NOWCASTING_SANDBOX.PRONTOEC_ASPECTSANDTHEMES`
- **Company metadata**: `SPGLOBALXPRESSCLOUD_SPGMIQRS.XPRESSFEED.CIQCOMPANY`
- **Join key**: `COMPANYID`
- **Warehouse**: `QRS_LLM_POC_WH_M_1`

---

## Core SQL Template

```sql
SELECT 
    a.calldate AS CALL_DATE,
    AVG(a.score) AS AVG_SENTIMENT_SCORE
FROM QRSLLM_POC_DB.NOWCASTING_SANDBOX.PRONTOEC_ASPECTSANDTHEMES a
WHERE a.companyid = {{COMPANY_ID}}
    AND a.eventcategorygroup = 'Aggregate Weighted Score'
    AND a.documentsectionname = 'Total'
    AND a.calldate >= '{{START_DATE}}'
GROUP BY a.calldate
ORDER BY a.calldate ASC
```
---

## Chart Specification (Vega-Lite)

```JSON
{
  "title": "{{COMPANY_NAME}} – Earnings Call Sentiment ({{YEAR_RANGE}})",
  "mark": { "type": "line", "point": true, "strokeWidth": 2 },
  "encoding": {
    "x": {
      "field": "CALL_DATE",
      "type": "temporal",
      "axis": { "title": "Earnings Call Date", "format": "%b %Y" }
    },
    "y": {
      "field": "AVG_SENTIMENT_SCORE",
      "type": "quantitative",
      "axis": { "title": "Sentiment Score" },
      "scale": { "domain": [0.2, 1.0] }
    },
    "tooltip": [
      { "field": "CALL_DATE", "type": "temporal", "title": "Call Date", "format": "%b %d, %Y" },
      { "field": "AVG_SENTIMENT_SCORE", "type": "quantitative", "title": "Score", "format": ".2f" }
    ]
  }
}
```

---

## Use Cases
### Company-Specific
### Single company trend:

```SQL
-- Replace {{COMPANY_ID}} with target company
-- Example: Samsung Electronics = 91868, Apple = 24937
WHERE a.companyid = {{COMPANY_ID}}
    AND a.eventcategorygroup = 'Aggregate Weighted Score'
    AND a.documentsectionname = 'Total'
```

### Management vs. Analyst sentiment (section breakdown):

```SQL
SELECT 
    a.calldate AS CALL_DATE,
    a.documentsectionname AS SECTION,
    AVG(a.score) AS AVG_SENTIMENT_SCORE
FROM QRSLLM_POC_DB.NOWCASTING_SANDBOX.PRONTOEC_ASPECTSANDTHEMES a
WHERE a.companyid = {{COMPANY_ID}}
    AND a.eventcategorygroup = 'Aggregate Weighted Score'
    AND a.documentsectionname IN ('CFO', 'Question')
    AND a.calldate >= '{{START_DATE}}'
GROUP BY a.calldate, a.documentsectionname
ORDER BY a.calldate ASC
```

### Chart addition for section breakdown:

```JSON
"color": { "field": "SECTION", "type": "nominal", "title": "Speaker Section" }
```

---

## Industry/Sector-Specific
### Sector average sentiment:

```SQL
SELECT 
    DATE_TRUNC('quarter', a.calldate) AS QUARTER,
    AVG(a.score) AS AVG_SECTOR_SENTIMENT
FROM QRSLLM_POC_DB.NOWCASTING_SANDBOX.PRONTOEC_ASPECTSANDTHEMES a
INNER JOIN SPGLOBALXPRESSCLOUD_SPGMIQRS.XPRESSFEED.CIQCOMPANY c
    ON a.companyid = c.companyid
WHERE c.simpleindustryid = {{INDUSTRY_ID}}
    AND a.eventcategorygroup = 'Aggregate Weighted Score'
    AND a.documentsectionname = 'Total'
    AND a.calldate >= '{{START_DATE}}'
GROUP BY DATE_TRUNC('quarter', a.calldate)
ORDER BY QUARTER ASC
```
### Peer group overlay:

```SQL
SELECT 
    a.calldate AS CALL_DATE,
    c.companyname AS COMPANY,
    AVG(a.score) AS AVG_SENTIMENT_SCORE
FROM QRSLLM_POC_DB.NOWCASTING_SANDBOX.PRONTOEC_ASPECTSANDTHEMES a
INNER JOIN SPGLOBALXPRESSCLOUD_SPGMIQRS.XPRESSFEED.CIQCOMPANY c
    ON a.companyid = c.companyid
WHERE a.companyid IN ({{ID_1}}, {{ID_2}}, {{ID_3}})
    AND a.eventcategorygroup = 'Aggregate Weighted Score'
    AND a.documentsectionname = 'Total'
    AND a.calldate >= '{{START_DATE}}'
GROUP BY a.calldate, c.companyname
ORDER BY a.calldate ASC
```
### Chart addition for peer comparison:

```JSON
"color": { "field": "COMPANY", "type": "nominal", "title": "Company" }
```
---
## Country/Region-Specific
### Country-level sentiment tracker:

```SQL
SELECT 
    DATE_TRUNC('quarter', a.calldate) AS QUARTER,
    AVG(a.score) AS AVG_SENTIMENT_SCORE
FROM QRSLLM_POC_DB.NOWCASTING_SANDBOX.PRONTOEC_ASPECTSANDTHEMES a
INNER JOIN SPGLOBALXPRESSCLOUD_SPGMIQRS.XPRESSFEED.CIQCOMPANY c
    ON a.companyid = c.companyid
WHERE c.countryid = {{COUNTRY_ID}}
    AND a.eventcategorygroup = 'Aggregate Weighted Score'
    AND a.documentsectionname = 'Total'
    AND a.calldate >= '{{START_DATE}}'
GROUP BY DATE_TRUNC('quarter', a.calldate)
ORDER BY QUARTER ASC
```
### Cross-country comparison:

```SQL
SELECT 
    DATE_TRUNC('quarter', a.calldate) AS QUARTER,
    c.countryid AS COUNTRY_ID,
    AVG(a.score) AS AVG_SENTIMENT_SCORE
FROM QRSLLM_POC_DB.NOWCASTING_SANDBOX.PRONTOEC_ASPECTSANDTHEMES a
INNER JOIN SPGLOBALXPRESSCLOUD_SPGMIQRS.XPRESSFEED.CIQCOMPANY c
    ON a.companyid = c.companyid
WHERE c.countryid IN ({{COUNTRY_ID_1}}, {{COUNTRY_ID_2}})
    AND a.eventcategorygroup = 'Aggregate Weighted Score'
    AND a.documentsectionname = 'Total'
    AND a.calldate >= '{{START_DATE}}'
GROUP BY DATE_TRUNC('quarter', a.calldate), c.countryid
ORDER BY QUARTER ASC
```
---
## Notes
- Scores are on a 0–1 scale derived from the ATC (Analyst Consensus) LLM model
- documentsectionname = 'Total' captures the full-call aggregate; use 'CFO' or 'Question' for section-level analysis
- eventcategorygroup = 'Aggregate Weighted Score' provides the headline sentiment metric weighted by importance
- Each earnings call may produce multiple rows per signal type; always AVG() and GROUP BY calldate for the chart
- Have only the chart and the bullet point summaries in the output, nothing else. Remain concise and professional
