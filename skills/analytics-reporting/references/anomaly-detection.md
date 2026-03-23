# Anomaly Detection Methods

## Z-Score Method (Univariate)

```
z-score = (value - mean) / stdev
flag if |z-score| > 2.5 (99% confidence)
```

**Pros:** Simple, fast, interpretable  
**Cons:** Assumes normal distribution, sensitive to outliers

## Moving Average with Seasonality

```
baseline = 30-day or 90-day rolling mean
trend = linear regression slope (7+ days)
seasonality = average(same_day_of_week) / baseline
adjusted_value = observed / seasonality
deviation = (adjusted_value - baseline) / baseline
flag if |deviation| > 0.25 (25% threshold)
```

**Use case:** Traffic reports, revenue, user counts  
**Adjust threshold:** Down to 0.15 (15%) for sensitive metrics

## Isolation Forest (Multivariate)

```python
from sklearn.ensemble import IsolationForest
model = IsolationForest(contamination=0.05)  # expect 5% anomalies
anomalies = model.fit_predict(data_matrix)
```

**Pros:** Handles multiple dimensions, non-linear patterns  
**Cons:** Needs 50+ samples, slower  
**Use case:** Cross-metric anomalies (traffic + conversion + bounce rate)

## Composite Anomaly Score

```
score_z = (|z_score| / 2.5) * 0.3
score_ma = (|deviation_percent| / 0.25) * 0.4
score_if = isolation_forest_anomaly_score * 0.3
composite = score_z + score_ma + score_if
flag if composite > 0.7
```

**Why:** No single method is perfect. Combine for robustness.

## Algorithm Change Detection

**Red flags:**
- Sudden platform update (Google Algo Core Update, GA4 change)
- Traffic pattern shift that reverses historical trend
- Multiple metrics moving in opposite directions
- Consistent variance spike across 5+ days

**How to detect:**
1. Check official Google announcements (Search Status Dashboard)
2. Compare current z-score against 12-month baseline (not 30-day)
3. Look for reversal: if metric was declining, sudden increase is suspicious
4. Flag "likely algorithm update" if variance increases 3x normal

## Seasonality Adjustment

**Example: Weekly pattern in SaaS traffic**
```
Monday: 1.15x average
Tuesday-Thursday: 1.05x average
Friday: 1.00x average
Weekend: 0.70x average
```

**Steps:**
1. Calculate mean for each day-of-week over 12 weeks
2. Divide each by overall mean → seasonality index
3. Divide today's actual by seasonality index → deseasonalized value
4. Compare deseasonalized to baseline

## When to Alert (Confidence Levels)

| Scenario | Threshold | Confidence |
|----------|-----------|------------|
| Revenue drop | >15% | 95%+ |
| Organic traffic drop | >20% | 90%+ |
| CTR anomaly | >10% | 85% |
| New content launch | any spike | Contextual |
| Weekend/holiday | expected | Seasonal |

## Common False Positives

1. **New content published** — Expected traffic spike, not anomaly
2. **Promotional period** — Expected volume increase
3. **One-off event** (outage, viral tweet) — Temporary, not systemic
4. **Data pipeline lag** — GA4 reports 24-48h delayed
5. **Sampling** (GA4 <1k/day) — High variance expected

**Mitigation:** Tag anomalies with context tags ("new_content", "promo", "event", "sampling") before flagging
