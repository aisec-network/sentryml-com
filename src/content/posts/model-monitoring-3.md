---
title: "Model Monitoring for LLM Inference: The Metrics Your APM Stack Can't See"
description: "Model monitoring for LLM APIs requires a different metric set than traditional ML. Here's the signal hierarchy — TTFT, KV cache hit rate, output length drift, refusal rate — wired up with OpenTelemetry and Prometheus."
pubDate: 2026-05-16
author: "SentryML Editorial"
tags: ["model-monitoring", "llm-observability", "ttft", "drift-detection", "mlops", "vllm"]
category: "monitoring"
sources:
  - title: "Data Drift Detection: Methods and Metrics — Evidently AI"
    url: "https://www.evidentlyai.com/ml-in-production/data-drift"
  - title: "KL Divergence: When to Use Kullback-Leibler Divergence — Arize AI"
    url: "https://arize.com/blog-course/kl-divergence/"
  - title: "Measuring Data Drift with the Population Stability Index — Fiddler AI"
    url: "https://www.fiddler.ai/blog/measuring-data-drift-population-stability-index"
schema:
  type: "TechArticle"
---

Model monitoring for a scikit-learn classifier and model monitoring for an LLM inference endpoint are related disciplines that share almost no tooling overlap at the operational layer. The classic framework — track input distributions, compute PSI against a reference dataset, alert when PSI exceeds 0.2 — still applies. But LLM deployments generate a different failure surface, and that surface is almost entirely invisible to standard APM.

The signals that matter for LLM inference run in three tiers: latency metrics that catch infrastructure failure, distribution metrics that catch input behavior change, and output quality metrics that catch model degradation. Most teams instrument the first tier and skip the other two until a user complaint lands.

## Tier 1 — Latency: TTFT Is the Number That Matters

Latency for LLM inference is not a single number. A request that takes 12 seconds might have generated its first token in 400ms — which feels fast to an interactive user — or it might have waited 11 seconds before producing anything, then finished quickly. Aggregate p99 latency collapses this distinction.

The metric hierarchy:

- **TTFT (time-to-first-token)**: wall-clock time from request receipt to first output token. This is the perceived latency for streaming applications. For vLLM deployments, TTFT spikes are usually a KV cache eviction problem — the cache is full, new requests are queued waiting for memory.
- **Time-per-output-token (TPOT)**: average time between consecutive output tokens after the first. TPOT degradation under load means you're GPU-bound on the decode phase.
- **Throughput (tokens/sec)**: aggregate output tokens per second across all concurrent requests. Use this to detect tensor parallelism imbalance — if one GPU in a TP-2 setup is slower, throughput falls while per-request latency may look fine.
- **Queue depth**: number of requests waiting for a batch slot. The queue depth metric often gives you 30–60 seconds of warning before TTFT degrades visibly.

vLLM exposes these via its `/metrics` Prometheus endpoint out of the box. Wire Grafana against it, set a p99 TTFT alert at 1.5× your baseline, and you'll catch most infrastructure-level problems before users do.

## Tier 2 — Input Distribution Monitoring

Prompt length distribution is the feature drift signal that moves first when your user base or upstream system changes behavior. A longer prompt shifts KV cache pressure, increases TTFT at the tail, and can expose prompt injection patterns you haven't seen in production before (see [aisec.blog](https://aisec.blog) for the attack taxonomy).

The practical metric is PSI on the prompt token count distribution, computed daily against a rolling 7-day reference window. The [Fiddler AI reference on PSI](https://www.fiddler.ai/blog/measuring-data-drift-population-stability-index) gives the standard thresholds:

```
PSI = Σ (Actual_pct(b) - Expected_pct(b)) × ln(Actual_pct(b) / Expected_pct(b))
```

- PSI < 0.1: distributions are similar
- 0.1 ≤ PSI < 0.2: moderate shift, investigate
- PSI ≥ 0.2: significant drift — your model is serving a meaningfully different population than the one you load-tested against

PSI is symmetric and interpretable; it's a cleaned-up form of KL divergence. [Arize's breakdown of KL divergence](https://arize.com/blog-course/kl-divergence/) covers why PSI is preferred for monitoring: KL is asymmetric, so swapping reference and current distributions gives different values — a debugging nightmare when you're comparing against multiple baselines. Use PSI for rollup signals and KL as a drill-down when you need to quantify directional shift.

Pair prompt-length PSI with topic/embedding drift if you have the compute budget. Encode a sample of prompts into a sentence embedding, compute the centroid and spread of the daily distribution, and track cosine distance from your baseline centroid. Embedding drift above a threshold — 0.05 cosine distance is a reasonable starting point — indicates the types of questions being asked have shifted, not just their length.

## Tier 3 — Output Quality Signals

Output quality monitoring is where most teams stop investing because ground truth is expensive. But several proxy signals don't require labels:

**Response length distribution**: track the p50 and p99 of output token counts daily. A model that starts producing shorter responses to the same prompt distribution is often hitting a temperature or sampling change, or exhibiting hallucination-suppression behavior from a silent guardrail activation. A model producing longer responses may be in a runaway generation loop or responding to adversarial prompts. [guardml.io](https://guardml.io) covers guardrail detection patterns that can cause exactly this kind of length shift.

**Refusal rate**: the fraction of requests that produce a refusal or safety-block response. Track it as a rolling hourly rate. Spike detection (> 3σ from the trailing 24-hour mean) catches two things: a new prompt injection campaign hitting your endpoint, or a silent model update from your provider that changed safety thresholds.

**Output entropy**: for classification-style prompts where your application extracts a structured label from the LLM response, track the distribution of extracted labels. PSI on a categorical label distribution — [Evidently's drift detection preset](https://www.evidentlyai.com/ml-in-production/data-drift) handles this natively — will tell you when the model's decision boundary has shifted before any human-labeled eval catches it.

## Wiring It Up

A minimal OpenTelemetry instrumentation for an LLM inference endpoint, capturing the signals described above:

```python
from opentelemetry import trace, metrics
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.exporter.prometheus import PrometheusMetricReader
import time

reader = PrometheusMetricReader()
provider = MeterProvider(metric_readers=[reader])
meter = provider.get_meter("llm.inference")

# Histograms — use explicit bucket boundaries matched to your SLOs
ttft_histogram = meter.create_histogram(
    "llm.ttft_ms",
    description="Time to first token in milliseconds",
    unit="ms",
)
prompt_tokens_histogram = meter.create_histogram(
    "llm.prompt_tokens",
    description="Prompt token count per request",
)
output_tokens_histogram = meter.create_histogram(
    "llm.output_tokens",
    description="Output token count per request",
)
refusal_counter = meter.create_counter(
    "llm.refusals_total",
    description="Count of refusal responses",
)

def record_inference(prompt_tokens: int, output_tokens: int,
                     ttft_ms: float, is_refusal: bool, model: str):
    attrs = {"model": model}
    ttft_histogram.record(ttft_ms, attrs)
    prompt_tokens_histogram.record(prompt_tokens, attrs)
    output_tokens_histogram.record(output_tokens, attrs)
    if is_refusal:
        refusal_counter.add(1, attrs)
```

With this instrumentation running, Prometheus scrapes the `/metrics` endpoint and you get per-model histograms for all four dimensions. Build a Grafana dashboard with:
- TTFT p50/p95/p99 time series with threshold bands
- `rate(llm_refusals_total[1h]) / rate(llm_requests_total[1h])` as refusal rate
- Prompt/output token count p95 with 7-day overlay for visual drift detection

## What You'll See

A clean deployment shows TTFT p99 stable within ±15% of baseline, prompt token p95 moving slowly with day-of-week seasonality, and refusal rate below 0.5% with no spikes. Output length p95 tracks prompt length with a roughly constant ratio.

Bad signals: TTFT p99 climbing monotonically without a traffic increase (KV cache pressure from longer prompts), refusal rate spiking to 5–15% in a 30-minute window (active injection campaign or provider safety update), output token p95 dropping 40% suddenly (new guardrail activation or model change).

## Caveats

Logging full prompt and completion text for drift analysis is expensive. At 1000 QPS with average 500-token prompts, storing full text is roughly 43 GB/hour. Sample at 1–5% for embedding-based drift analysis and store only token counts for the full population. Be aware that low sampling rates miss rare-event detection — refusal spikes from short attack campaigns may not appear in a 1% sample. For refusal rate specifically, compute it from a flag in your application logic, not from sampled logs.

PSI on prompt length assumes your bucketization is stable. If your tokenizer changes — model upgrade, library version bump — recompute your reference distribution before alerting. A tokenizer change that shifts the average token count by 15% will trigger PSI > 0.2 across every feature, drowning real drift signals in false positives.

## Sources

- [Data Drift Detection: Methods and Metrics — Evidently AI](https://www.evidentlyai.com/ml-in-production/data-drift): comprehensive overview of statistical tests (KS, PSI, KL, Wasserstein) and how Evidently's open-source library implements them, including 20+ drift detection presets.
- [KL Divergence: When to Use Kullback-Leibler Divergence — Arize AI](https://arize.com/blog-course/kl-divergence/): practical guidance on when to use KL vs PSI, the asymmetry problem, and cardinality limits for categorical features.
- [Measuring Data Drift with the Population Stability Index — Fiddler AI](https://www.fiddler.ai/blog/measuring-data-drift-population-stability-index): PSI formula, standard industry thresholds (< 0.1 / 0.1–0.2 / > 0.2), and Fiddler's application to production feature monitoring.
