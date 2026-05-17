---
title: "Model Monitoring in Production: The Four-Layer Framework That Catches Silent Failures"
description: "Model monitoring covers more than drift detection. Here's the four-layer framework — software health, data quality, model quality, business KPIs — wired up with Evidently, PSI thresholds, and real alert logic."
pubDate: 2026-05-16
author: "SentryML Editorial"
tags: ["model-monitoring", "drift-detection", "mlops", "evidently", "psi"]
category: "monitoring"
sources:
  - title: "Model Monitoring in ML Production — Evidently AI"
    url: "https://www.evidentlyai.com/ml-in-production/model-monitoring"
  - title: "Measuring Data Drift with the Population Stability Index — Fiddler AI"
    url: "https://www.fiddler.ai/blog/measuring-data-drift-population-stability-index"
  - title: "ML Model Monitoring in Production: Best Practices — Datadog"
    url: "https://www.datadoghq.com/blog/ml-model-monitoring-in-production-best-practices/"
schema:
  type: "TechArticle"
---

Model monitoring exists precisely because ML failures are invisible to APM. A model that's quietly degrading looks fine at the infrastructure level — CPU normal, latency within SLA, error rate zero. But the predictions are wrong: a churn model that stopped identifying at-risk customers, a fraud detector missing a new fraud vector, a recommendation engine stuck on stale embeddings. You won't know until a PM files a ticket or revenue takes a measurable hit.

Unlike software monitoring, where a bug surfaces as a 5xx or an exception in the logs, ML failures are statistical. They accumulate silently over days or weeks. Traditional observability tools catch infrastructure problems. Model monitoring catches the other kind.

## The Four Layers (Most Teams Only Watch One)

Model monitoring covers four distinct layers, each tracking a different failure mode:

**Layer 1 — Software health**: Inference latency (p50/p95/p99), throughput (QPS), error rates, GPU/CPU utilization, memory usage. This is table stakes. Prometheus + Grafana handles it; Ray Serve and vLLM expose these metrics natively. Most teams have this layer. The other three are where production incidents actually originate.

**Layer 2 — Data quality**: Are features arriving as expected? Missing values, schema violations, feature range violations, and corrupted inputs all degrade predictions without throwing an exception. A feature that flips from float to null doesn't error — it just makes the model wrong. This layer is responsible for the majority of silent ML failures in production.

**Layer 3 — Model quality**: Accuracy, precision, recall, AU-ROC (for classification); RMSE, MAE (for regression). This is the hardest layer to monitor because ground truth arrives late. A recommendation model may not know whether a prediction converted for 48 hours. The practical workaround: monitor the output distribution as a proxy signal, and build a feedback pipeline to eventually close the label loop.

**Layer 4 — Business KPIs**: Conversion rate, revenue attributed, escalation rate, cost per decision. The model exists to move a business metric. If that metric decouples from model predictions, something changed — either the model or the world it's operating in.

Most post-mortems in ML blame Layer 2. Input data broke in an upstream feature pipeline, propagated silently through the model, and degraded Layer 4 metrics across a week before anyone noticed.

## The Metric That Matters: PSI

For detecting distribution shift in input features, Population Stability Index (PSI) is the production standard. The formula:

```
PSI = Σ (Actual_pct(b) - Expected_pct(b)) × ln(Actual_pct(b) / Expected_pct(b))
```

Where `b` is each bin in the discretized distribution. PSI is mathematically equivalent to a symmetrized KL divergence — it measures how far the current serving distribution has shifted from the training distribution. [Fiddler AI's reference on PSI](https://www.fiddler.ai/blog/measuring-data-drift-population-stability-index) gives the standard industry thresholds:

- PSI < 0.1: distributions are similar, no action needed
- 0.1 ≤ PSI < 0.2: moderate shift, investigate root cause
- PSI ≥ 0.2: significant drift — consider retraining or model rollback

Why PSI over accuracy? Because ground truth is almost never available in real time. You can compute PSI the moment a batch of requests arrives. You can't compute accuracy until the label shows up — which might be hours, days, or never for some use cases. PSI is a leading indicator; accuracy is a lagging one.

PSI pairs naturally with the Kolmogorov-Smirnov test (for continuous features) and Chi-square (for categoricals). Use PSI as the rollup signal, then drill into per-feature KS scores to identify which features are driving the drift.

## Wiring It Up with Evidently

[Evidently](https://www.evidentlyai.com) is the open-source Python library most teams reach for first. It runs as a standalone report generator or inside an Airflow DAG. Here's a minimal daily drift check against a reference dataset:

```python
import pandas as pd
from evidently.report import Report
from evidently.metric_preset import DataDriftPreset, DataQualityPreset

# reference = training distribution snapshot stored in object storage
# current = yesterday's serving logs
reference = pd.read_parquet("s3://my-bucket/reference/features_2025_q4.parquet")
current = pd.read_parquet("s3://my-bucket/serving/features_2026_05_15.parquet")

report = Report(metrics=[
    DataQualityPreset(),
    DataDriftPreset(drift_share=0.3),  # flag if >30% of features drift
])

report.run(reference_data=reference, current_data=current)
report.save_html("drift_report_2026_05_15.html")

# Programmatic alerting: read the result dict
result = report.as_dict()
dataset_drift = result["metrics"][1]["result"]["dataset_drift"]

if dataset_drift:
    send_alert(
        channel="#ml-oncall",
        message=f"Input drift detected: {result['metrics'][1]['result']}"
    )
```

For continuous (not batch) monitoring, Arize and WhyLabs both offer real-time ingestion — you log predictions and features via a Python client at inference time, and the platform computes PSI continuously. WhyLabs uses the open-source `whylogs` library to compute lightweight statistical sketches per request, keeping per-inference overhead under 1ms. Fiddler provides SHAP-based explanations alongside drift scores, which helps attribute drift to specific features.

For teams in regulated environments where sending production data to a SaaS platform is blocked, [guardml.io](https://guardml.io) covers the tradeoffs between on-premises and cloud-hosted ML observability.

## What You'll See on the Dashboard

**Healthy state**: PSI near zero and flat for all monitored features. Output distribution stable week-over-week. Null rates at baseline. Latency p99 within SLA.

**Drift event**: One or more features cross PSI 0.2, typically correlated with an upstream data pipeline change, a product launch that shifted user behavior, or a seasonal pattern the training data didn't cover. The output distribution shifts mean or widens. If ground truth arrives within a reasonable window, accuracy drops 24-72 hours after the drift event began.

**Silent breakage**: Null rate on a key feature goes from 0.1% to 40%. PSI on that feature becomes undefined (all traffic collapses into one bin). Model output distribution narrows abnormally — often to a near-constant modal prediction. This is the scenario that pages you at 3am if you have alerting on schema violations. Without it, a PM discovers it two weeks later.

When you see anomalous drift, cross-reference with [ai-alert.org](https://ai-alert.org) if you're calling third-party model APIs — occasionally what looks like local input drift is actually a silent model update on the vendor side.

## Caveats

**Label delay is the hard problem.** Building a feedback loop to return ground truth to your monitoring system requires real engineering investment. Without it, you're relying on PSI alone, which catches input drift but not concept drift — where the relationship between inputs and the correct output changes without any distributional shift in the features themselves.

**Thresholds are context-sensitive.** The 0.1/0.2 PSI thresholds originated in credit scoring. A fraud model operating against an adversarial distribution may need tighter thresholds. A stable demand forecasting model may tolerate PSI > 0.2 on some features without meaningful accuracy impact. Calibrate thresholds against observed retraining outcomes in your system, not industry defaults from a different domain.

**High-cardinality features break statistical tests.** Device ID, user ID, session token — any feature with millions of unique values produces meaningless PSI and KS scores. Either exclude them from drift monitoring, or hash into fixed-width buckets before computing statistics.

**Alert fatigue is a real cost.** Monitoring every feature at the same threshold generates noise. Start with the five or ten features with the highest feature importance scores (SHAP values from your training runs are a good proxy). Set PSI alerts on those first. Add coverage incrementally as you learn which features actually predict downstream degradation.

## Sources

- **[Model Monitoring in ML Production — Evidently AI](https://www.evidentlyai.com/ml-in-production/model-monitoring)**: The authoritative open-source reference on the four monitoring layers, failure modes, and architecture options (batch vs. real-time). The layer framework in this post draws from their guide.

- **[Measuring Data Drift with the Population Stability Index — Fiddler AI](https://www.fiddler.ai/blog/measuring-data-drift-population-stability-index)**: PSI formula, threshold interpretation, and the connection to KL divergence. Fiddler's platform uses PSI as its primary feature drift signal in production. Vendor source — their thresholds match industry-wide conventions from financial services, where PSI originated.

- **[ML Model Monitoring in Production: Best Practices — Datadog](https://www.datadoghq.com/blog/ml-model-monitoring-in-production-best-practices/)**: Practical runbook guidance from Datadog's ML observability team on alerting strategies, prediction log archiving in object storage, and correlating model metrics with business KPIs. Vendor source.
