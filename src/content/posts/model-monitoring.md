---
title: "Model Monitoring in Production: What to Track and When to Act"
description: "A practical guide to model monitoring for ML engineers: drift types, the metrics that actually matter, handling the no-ground-truth problem, and which tools to reach for first."
pubDate: 2026-05-11
author: "SentryML Editorial"
tags: ["model-monitoring", "data-drift", "mlops", "observability", "concept-drift"]
category: "monitoring"
sources:
  - title: "Model monitoring for ML in production: a comprehensive guide — Evidently AI"
    url: "https://www.evidentlyai.com/ml-in-production/model-monitoring"
  - title: "A Guide to Monitoring Machine Learning Models in Production — NVIDIA Technical Blog"
    url: "https://developer.nvidia.com/blog/a-guide-to-monitoring-machine-learning-models-in-production/"
  - title: "Model monitoring in production — Azure Machine Learning docs"
    url: "https://learn.microsoft.com/en-us/azure/machine-learning/concept-model-monitoring?view=azureml-api-2"
  - title: "Machine learning model monitoring: Best practices — Datadog"
    url: "https://www.datadoghq.com/blog/ml-model-monitoring-in-production-best-practices/"
schema:
  type: "TechArticle"
---

Model monitoring is the ongoing practice of tracking how a deployed machine learning model behaves against real-world data — and acting on what you find before degradation compounds into a business problem. Unlike traditional software, ML models can fail silently. A classification model predicting churn or fraud can look perfectly healthy from an infrastructure standpoint while producing increasingly wrong answers, because the relationship between features and labels has drifted since training.

This post covers the mechanics of what breaks, the metrics worth tracking at each layer, and the practical decisions — tooling, alert thresholds, retraining triggers — that belong in your runbook.

## Why Models Degrade Without Obvious Errors

The [Azure ML documentation](https://learn.microsoft.com/en-us/azure/machine-learning/concept-model-monitoring?view=azureml-api-2) puts it plainly: ML system behavior is learned from data, not encoded in rules. That means the same code base, running correctly, can produce increasingly bad predictions as the world changes around it.

There are two root causes that account for most production degradation:

**Data drift** — the statistical distribution of your input features shifts from what the model saw during training. A fraud model trained on pre-pandemic transaction patterns, for example, will encounter a fundamentally different input distribution once consumer behavior normalizes to a new baseline. Feature means shift, variance expands, and categorical feature cardinality grows.

**Concept drift** — the underlying relationship between features and labels changes. Even if your inputs look identical, what those inputs *mean* for the outcome evolves. A content recommendation model experiences concept drift when user preferences shift seasonally. Financial risk models experience it after regulatory changes restructure what "high-risk" borrowers actually look like.

Data drift is detectable without labels. Concept drift is harder — you need ground truth to confirm it directly, though prediction drift serves as an early proxy.

## The Four Monitoring Layers

[NVIDIA's production monitoring guide](https://developer.nvidia.com/blog/a-guide-to-monitoring-machine-learning-models-in-production/) organizes ML monitoring into functional and operational layers. In practice, the monitoring pyramid looks like this, from most-ignored to most-visible:

**Layer 1 — Software health.** Latency percentiles, error rates, GPU/CPU utilization, memory pressure. If you're already running Prometheus and Grafana, NVIDIA Triton exports compatible metrics out of the box. This layer rarely catches model quality problems, but it catches infrastructure issues masquerading as model issues.

**Layer 2 — Data quality.** Missing values, type mismatches, out-of-bounds values, schema violations. Azure ML Model Monitor tracks null value rate, data type error rate, and out-of-bounds rate per feature. These are cheap to compute and catch pipeline failures that corrupt inputs upstream of the model.

**Layer 3 — Distribution drift.** Statistical comparison of production inputs against a training or recent-production baseline. Common metrics:
- **Kolmogorov-Smirnov (KS) test** for continuous features
- **Pearson's Chi-Squared test** for categorical features
- **Jensen-Shannon distance** and **Population Stability Index (PSI)** for both

PSI is particularly popular in financial services because it's interpretable: PSI < 0.1 means negligible shift, 0.1–0.25 warrants investigation, > 0.25 signals significant distribution change.

**Layer 4 — Model performance.** Accuracy, F1, MAE, RMSE — whatever metric matches your business objective. This is the most meaningful signal and the hardest to obtain, because it requires ground truth labels that often arrive with delay.

## The No-Ground-Truth Problem

Most production ML systems face a label latency problem. A credit model may not know if a loan defaults for 12–18 months. A fraud model depends on chargebacks that arrive weeks later. [Evidently AI's guide](https://www.evidentlyai.com/ml-in-production/model-monitoring) recommends a pragmatic tiered approach:

1. **When ground truth is available**, use it. Compare predictions against actuals on whatever lag fits your use case. Build a rolling evaluation window.
2. **When ground truth is delayed**, monitor prediction drift as a proxy. If the distribution of model outputs is shifting, something is changing upstream — either the input distribution, the model's decision boundary in practice, or both.
3. **When ground truth is indefinitely unavailable**, weight your monitoring toward data quality and input distribution signals. Feature attribution drift — tracking whether feature importance rankings in production match training — can surface degradation even without labels.

NannyML's Confidence-Based Performance Estimation (CBPE) extends this further for classification: it estimates performance metrics without labels by exploiting the relationship between predicted probabilities and historical accuracy, giving you an approximation of current precision and recall without waiting for actuals.

For ML observability tooling that handles this full spectrum, see [mlobserve.com](https://mlobserve.com) and the comparison coverage at [mlmonitoring.report](https://mlmonitoring.report).

## Tooling Landscape

The mature options in this space:

**Evidently AI** — open-source Python library with 25M+ downloads. Generates interactive HTML reports, runs test suites inside prediction pipelines, and exposes dashboards. Strong for batch monitoring and CI/CD integration. Free tier covers most use cases; hosted version adds scheduling and team features.

**Arize AI** — purpose-built platform with strong support for embedding drift (critical for NLP and vision models), real-time monitoring, and explainability. Paid, with a free tier that's genuinely useful for smaller deployments.

**WhyLabs** — built on whylogs, an open-source logging library. Lightweight to instrument, good Kafka/streaming integration, privacy-preserving statistical sketches.

**Fiddler** — enterprise-focused, stronger on compliance use cases where fairness and bias monitoring matter alongside drift detection.

**Prometheus + Grafana** — sufficient for Layer 1 and parts of Layer 2 if you already have them in your stack. Not designed for Layer 3 and Layer 4 signals; you'll end up building custom exporters.

**Azure ML Model Monitor / SageMaker Model Monitor** — reasonable choices if you're already locked into the respective cloud platform. Minimal extra setup, decent built-in alerts.

## What Goes in Your Runbook

Based on the patterns that cause most production incidents, here's what to configure before your next deployment:

**Baseline everything at deploy time.** Compute and store feature distribution statistics from your validation set before pushing to production. This is your comparison reference. Drift against a stale or incorrect baseline produces false alarms that erode alerting trust.

**Set PSI alerts at two thresholds.** Warning at PSI > 0.1 for your top 10 features by importance. Critical at PSI > 0.25. Route warning alerts to a Slack channel; route critical alerts to pager.

**Track prediction drift independently of input drift.** A model can receive drifted inputs and still produce stable output distributions — or vice versa. Both combinations matter. Sudden compression of prediction probabilities toward a single class often indicates a broken feature pipeline, not gradual drift.

**Define your retraining trigger explicitly.** "When accuracy drops" is not a trigger — it's a post-mortem. Specify: if PSI on feature X exceeds 0.25 for three consecutive monitoring windows, or if prediction drift KS distance exceeds 0.15, open a retraining task automatically. The threshold calibration takes iteration, but having an explicit trigger beats manual inspection at review cadence.

**Monitor feature attribution drift if you have compute budget for it.** Feature importance rankings in production should roughly match training. If a previously low-importance feature becomes dominant in production, that's a signal the model is relying on something that may be data-leakage or a spurious correlation that's newly prominent.

The [Datadog ML monitoring guide](https://www.datadoghq.com/blog/ml-model-monitoring-in-production-best-practices/) recommends involving the data scientists who trained the model in threshold-setting, precisely because the acceptable variance in PSI or KS distance is domain-dependent. A recommendation model tolerates more input drift than a safety-critical medical classifier.

## Sources

- [Model monitoring for ML in production: a comprehensive guide](https://www.evidentlyai.com/ml-in-production/model-monitoring) — Evidently AI's comprehensive breakdown of monitoring components, drift types, and tooling. Updated regularly; primary reference for this post.
- [A Guide to Monitoring Machine Learning Models in Production](https://developer.nvidia.com/blog/a-guide-to-monitoring-machine-learning-models-in-production/) — NVIDIA Technical Blog. Covers the functional/operational monitoring split with specific Prometheus/Triton integration guidance.
- [Model monitoring in production — Azure Machine Learning](https://learn.microsoft.com/en-us/azure/machine-learning/concept-model-monitoring?view=azureml-api-2) — Microsoft's official documentation covering built-in signals, statistical methods (KS, Jensen-Shannon, PSI), lookback windows, and alert integration via Event Grid.
- [Machine learning model monitoring: Best practices](https://www.datadoghq.com/blog/ml-model-monitoring-in-production-best-practices/) — Datadog's engineering blog post on production ML monitoring, including threshold-setting and alert fatigue considerations.
