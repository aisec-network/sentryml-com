---
title: "Machine Learning Pipeline: Stages, Failure Points, and What to Monitor"
description: "A practitioner's guide to the machine learning pipeline — from data ingestion to production monitoring — covering common failure points, drift types, and the alerts that actually matter."
pubDate: 2026-05-11
author: "SentryML Editorial"
tags: ["mlops", "monitoring", "drift", "pipelines", "data-validation", "ci-cd"]
category: "mlops"
sources:
  - title: "MLOps: Continuous Delivery and Automation Pipelines in Machine Learning — Google Cloud"
    url: "https://docs.cloud.google.com/architecture/mlops-continuous-delivery-and-automation-pipelines-in-machine-learning"
  - title: "MLOps Workflows on Databricks — Databricks Documentation"
    url: "https://docs.databricks.com/aws/en/machine-learning/mlops/mlops-workflow"
  - title: "Monitoring Data and Model Drift in Your MLOps Pipeline — Persistent Systems"
    url: "https://www.persistent.com/blogs/monitoring-data-and-model-drift-in-your-mlops-pipeline/"
schema:
  type: "TechArticle"
---

A machine learning pipeline is not a training script with extra steps. It is the entire automated chain of work — data ingestion, validation, preprocessing, training, model validation, deployment, and monitoring — that lets a model move reliably from a notebook to a production system and stay accurate over time. Most teams underinvest in the pipeline and overinvest in the model, and this is why models degrade quietly for weeks before anyone notices.

Here is what each stage actually does, where it breaks, and what to add to your runbook.

## The Core Stages

**Data ingestion and validation** is the entry point. Raw data arrives from databases, APIs, event streams, or batch exports. The immediate job is not to store it — it's to reject bad data before it propagates downstream. Schema mismatches, unexpected nulls, and silent type coercions all become training bugs if you let them through. A validation layer that checks expected column types, value ranges, and distribution statistics against a stored baseline will catch most problems before training begins. If it fails, the pipeline fails loudly.

**Preprocessing and feature engineering** transforms validated data into the form your model expects. Scalers, encoders, imputers — these are fitted on training data and must be serialized alongside the model. A preprocessing step that recomputes statistics on the full dataset at inference time is a data leakage bug. The artifact that goes into your model registry should be a pipeline object (scikit-learn `Pipeline`, a Spark ML `PipelineModel`, etc.), not a bare model weight file.

**Training and experiment tracking** is where most MLops investment currently sits, and it's the least operationally risky stage once the data is clean. Log hyperparameters, metrics, and the dataset version that produced them. MLflow, Weights & Biases, and Comet are the standard options. The critical outcome is a lineage record: given the model currently serving production traffic, you should be able to answer "what data produced it and what was its validation AUC?" without asking the data scientist who ran the job.

**Model validation** is a gate, not a formality. Compare the new candidate against the champion currently in production on a held-out evaluation set that reflects recent distribution. If the candidate underperforms or fails a bias audit, it should not advance. Databricks calls this the [Champion/Challenger pattern](https://docs.databricks.com/aws/en/machine-learning/mlops/mlops-workflow), and it's the right framing: you are not deploying a model, you are deciding whether to replace the current one.

**Deployment** should package code, not weights. A model served as a REST endpoint via FastAPI or a managed serving layer (SageMaker, Vertex AI, Azure ML) is straightforward. The operational hazard is shadow models — older versions left running, consuming resources, and occasionally receiving traffic from misconfigured routing. Keep the model registry as the single source of truth for what is in production and retire versions explicitly.

**Monitoring** is where most pipelines are underbuilt. A deployed model without monitoring is a time bomb, not a product.

## What to Monitor and When to Alert

[Google's MLOps maturity framework](https://docs.cloud.google.com/architecture/mlops-continuous-delivery-and-automation-pipelines-in-machine-learning) describes three levels: Level 0 is manual — a data scientist hands off a pickle file and an engineer wraps it in Flask. Level 1 automates the training pipeline with data validation and scheduled retraining triggers. Level 2 adds full CI/CD automation across the pipeline. Most teams that have "deployed a model" are at Level 0, which means no monitoring is in place.

Getting to Level 1 requires instrumenting four signals:

**[Data drift](https://mlmonitoring.report/)** — the distribution of input features shifts relative to the training baseline. This is the most common form of pipeline degradation. Income distributions change, sensor calibrations drift, upstream ETL jobs get quietly modified. Use statistical tests: Kolmogorov-Smirnov for continuous features, chi-squared for categoricals. A 95% confidence threshold on a rolling window catches most cases without too many false positives.

**Concept drift** — the relationship between inputs and the target changes. This is harder to detect without ground truth labels and typically shows up weeks later as prediction accuracy drops. Monitor output distributions as a leading indicator: if the model's confidence scores or predicted class proportions shift significantly, that is a signal worth investigating even before labels arrive.

**Prediction drift** — the distribution of model outputs shifts. If a classification model that was producing balanced class probabilities starts skewing 80% toward one class, something has changed. This can be instrumented without ground truth.

**Infrastructure metrics** — latency p99, request error rate, and batch throughput. These do not tell you the model is wrong, but they tell you the model is not serving. Set alerts at the same threshold you would for any production API.

Tools: [Evidently AI](https://www.evidentlyai.com/) and WhyLabs handle statistical drift detection for batch and streaming inference. Arize and Fiddler add ground truth ingestion for concept drift tracking when labels become available with delay. For teams already on Databricks, the built-in data profiling monitors cover the basics without additional tooling.

For the retraining trigger: do not retrain on a fixed schedule. Retrain when a drift threshold is breached or when a performance metric drops below a defined floor on recent labeled data. Schedule-based retraining trains on stale distributions as often as it trains on useful ones.

## The Failure Pattern Nobody Plans For

The most operationally damaging failure is not a crashed service — it's a model that is technically running but producing degraded predictions. No alert fires. No SLA is breached. The model returns 200s for every request. But the F1 score on real-world inputs has dropped 15 points because an upstream feature pipeline silently changed how it handles nulls three weeks ago.

This is why the data validation stage matters more than the model architecture. [Persistent Systems' analysis of production drift incidents](https://www.persistent.com/blogs/monitoring-data-and-model-drift-in-your-mlops-pipeline/) consistently shows that not all drift immediately degrades performance — impact depends on feature importance and drift severity. A covariate that contributes 2% to model output can drift significantly without measurable performance impact. A covariate that drives 40% of the decision boundary cannot.

This is the case for feature importance-weighted alerting: weight your drift alerts by how much each feature actually influences the model. Evidently supports this natively. It dramatically reduces noise while keeping the signal that matters.

For security-conscious ML teams, pipeline inputs are also an attack surface. Adversarial inputs designed to trigger model failures or extract training data look like distribution shift to a naive monitoring layer. See [adversarialml.dev](https://adversarialml.dev) for current research on distinguishing adversarial drift from natural distribution shift, and [mlobserve.com](https://mlobserve.com) for tooling comparisons across observability platforms.

## What Goes in the Runbook

When a drift alert fires:

1. Check the data validation log for the last 24 hours. Is the schema intact? Are null rates within expected ranges?
2. Compare the current input feature distribution to the training baseline using your monitoring tool's statistical test output.
3. If data drift is confirmed, check upstream data pipelines for schema changes or ETL modifications.
4. Assess feature importance: is the drifted feature in the top quartile by SHAP contribution?
5. If yes: trigger a retraining pipeline on recent data. Do not promote until the Champion/Challenger evaluation passes.
6. If no: log the drift, set a review reminder in 7 days, and continue monitoring.

When performance metrics drop without detected drift:

1. Pull a sample of recent inference requests and run manual inspection.
2. Check for concept drift signals — is the predicted class distribution shifting?
3. Review whether ground truth labels are available for recent predictions; if so, compute current accuracy against the production baseline.
4. Consider triggering a retraining pipeline with a broader recent data window.

---

## Sources

- **[MLOps: Continuous Delivery and Automation Pipelines in Machine Learning — Google Cloud](https://docs.cloud.google.com/architecture/mlops-continuous-delivery-and-automation-pipelines-in-machine-learning)**: Google's canonical reference for MLOps maturity levels (0–2), pipeline architecture, and CI/CD automation patterns for ML systems.

- **[MLOps Workflows on Databricks — Databricks Documentation](https://docs.databricks.com/aws/en/machine-learning/mlops/mlops-workflow)**: Databricks' three-stage MLOps model (dev → staging → production), covering Champion/Challenger deployment, Unity Catalog governance, and MLflow-based tracking.

- **[Monitoring Data and Model Drift in Your MLOps Pipeline — Persistent Systems](https://www.persistent.com/blogs/monitoring-data-and-model-drift-in-your-mlops-pipeline/)**: Applied analysis of data drift and concept drift in production pipelines, including statistical test selection and the relationship between feature importance and drift impact on model performance.
