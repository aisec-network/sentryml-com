---
title: "Watermarking Should Be Treated as a Monitoring Primitive"
description: "A new paper reframes LLM watermarking from an adversarial evasion problem into a monitoring infrastructure question. If you are deploying watermarked models, you are deploying a monitoring system — whether or not you intended to."
pubDate: 2026-05-14
author: "SentryML Editorial"
tags: ["watermarking", "monitoring", "provenance", "attribution", "mlops"]
category: "monitoring"
sources:
  - title: "Watermarking Should Be Treated as a Monitoring Primitive (arXiv:2605.13095)"
    url: "https://arxiv.org/abs/2605.13095"
  - title: "A Watermark for Large Language Models (Kirchenbauer et al., arXiv:2301.10226)"
    url: "https://arxiv.org/abs/2301.10226"
schema:
  type: "TechArticle"
---

A paper posted to arXiv on May 14 ([arXiv:2605.13095](https://arxiv.org/abs/2605.13095)) makes a narrow argument that has broad operational implications: watermarking is not just a provenance feature, it is a monitoring primitive. The threat model almost everyone uses to evaluate watermarking schemes — can an adversary remove the watermark from a single sample? — is the wrong question. The right question is what an external observer can infer by aggregating watermark signals across many outputs over time.

If you are running a model in production with a watermarking scheme enabled, this is not a research curiosity. It is a monitoring surface you probably have not fully scoped.

## The signal

The standard watermarking evaluation goes like this: you have a provider, a user, and an adversary. The adversary tries to strip the watermark from individual outputs, or to forge a watermark to cause a false positive. The scheme is considered good if it resists those attacks at the per-sample level.

The paper introduces a different actor: the observer. An observer does not need to attack individual samples. They aggregate watermark signals across many outputs, infer per-entity statistical patterns, and extract attribution information the watermark was never supposed to leak externally.

The authors show that even zero-bit watermarking — the simplest variant, where each output either is or is not watermarked — enables attribution under multi-key settings. If different users or API clients receive outputs tagged with different attribution keys, an observer with detector access can, over time, identify which key produced which class of outputs. This is not a cryptographic break. It is a statistical aggregation problem, and it is the same class of problem that breaks naive differential privacy when you can run enough queries.

The dual-use tension the paper names is real: the properties that make a watermark useful for internal monitoring (persistent, key-dependent statistical structure, aggregatable across outputs) are exactly the properties that make it useful for external surveillance. You cannot have one without the other, unless you specifically design for distribution-preserving or undetectable schemes — and the paper is honest that those constraints are nontrivial.

## Mechanics

The key framing is the observer-based threat model. An observer has three things:

1. Access to outputs from the system (they are a user, or can query the API)
2. Knowledge that a watermarking scheme is in use (not secret at any scale)
3. Detector access — either because the detection API is public, or because they have trained their own detector against known watermarked outputs

Given those three, the observer runs a statistical aggregation. For [KGW-style watermarks](https://arxiv.org/abs/2301.10226) (the Kirchenbauer-Geiping-Wen green/red token list scheme, still the most widely analyzed), each output biases token selection using a key-dependent hash. Individual outputs are low-signal; aggregated outputs reveal the key's fingerprint. The observer does not extract the key itself. They learn which outputs share a key, which is often enough to infer entity-level information — which API client, which user tier, which model variant.

Multi-bit watermarks, which embed structured messages in outputs rather than a binary present/absent signal, compound the problem. The message structure carries more entropy, which means there is more key-dependent statistical structure for an observer to exploit. More bits in the watermark means more bits of information leaking per output.

The mitigation paths the paper identifies are distribution-preserving schemes (the watermarked distribution is statistically indistinguishable from the unwatermarked one, which makes aggregation attacks harder) and undetectable schemes (where even knowing the scheme exists does not help an observer distinguish watermarked from unwatermarked outputs). Both are significantly harder to build than standard watermarks, and the paper does not claim they are solved problems.

## Operational takeaway

If you are deploying watermarking — for output provenance, regulatory compliance, abuse detection, or copyright attribution — three things belong in your runbook.

**Scope your monitoring surface explicitly.** A watermarking deployment is a monitoring deployment. Your internal telemetry pipeline that ingests watermark detection signals has the same access an external observer is trying to build. The difference is you should be using yours deliberately. Map exactly what entity-level information is inferable from your key structure and detection logs. If the answer is "a lot more than we meant," that is a design decision, not a side effect.

**Treat the detector as infrastructure, not a debug tool.** Most teams stand up a watermark detector to verify that outputs are being watermarked, run it on a few samples, and move on. The paper's argument implies the detector should be a first-class metric emitter. Run it continuously across a sample of production outputs. Track the detection rate as a time series. A detection rate that drops (watermark stripping at scale) or shifts by key (unexpected key distribution) is a signal worth alerting on. Wire this into Prometheus the same way you wire inference latency:

```python
from prometheus_client import Gauge, Counter
import hashlib

wm_detection_rate = Gauge(
    "watermark_detection_rate",
    "Fraction of sampled outputs that pass watermark detection",
    ["model_id", "attribution_key_hash"],
)
wm_strip_events = Counter(
    "watermark_strip_events_total",
    "Outputs that fail detection above expected baseline",
    ["model_id"],
)

def record_detection(model_id: str, key: str, detected: bool, baseline_rate: float = 0.95):
    key_hash = hashlib.sha256(key.encode()).hexdigest()[:8]
    wm_detection_rate.labels(model_id=model_id, attribution_key_hash=key_hash).set(
        float(detected)
    )
    if not detected:
        wm_strip_events.labels(model_id=model_id).inc()
```

The `attribution_key_hash` label gives you per-key time series without putting raw key material in your metrics cardinality. Aggregate detection rate by key over a rolling 1-hour window; a sustained drop below your baseline is the alert threshold.

**Audit your key distribution schema.** The observer-based attack gets easier when attribution keys map cleanly to entity types. If API tier A always uses key K1, and API tier B always uses key K2, an external observer can infer tier from watermark statistics without ever breaking your cryptography. Consider key rotation schedules and mixing strategies that make per-entity key fingerprints harder to pin down, while still preserving your internal attribution ability. MLflow's run tracking is a reasonable place to log key rotation events as experiment metadata, so you can correlate detection rate changes with key lifecycle events post-hoc.

**Know what your scheme leaks before your adversary does.** The practical question is not "can someone strip our watermark" (yes, eventually, at some cost). It is "what does aggregated watermark signal tell an observer about our user base, model variants, or key structure?" Answer that question with a red-team exercise before you answer it from a researcher's paper.

## Where this leaves the monitoring team

The paper's conclusion is that watermarking cannot be evaluated in isolation from monitoring. The same statistical structure that makes a watermark detectable internally makes it observable externally. This is not an argument against watermarking. It is an argument that watermarking is monitoring infrastructure and should be designed and operated like it.

The teams that will get burned are the ones that deploy a watermarking scheme as a compliance checkbox, never wire up the detector as a metric, and discover the dual-use properties when an external observer beats them to the analysis. The monitoring team's job is to ensure that does not happen.

## Sources

- [Watermarking Should Be Treated as a Monitoring Primitive](https://arxiv.org/abs/2605.13095) — arXiv:2605.13095, the paper motivating this analysis. Introduces the observer-based threat model and demonstrates statistical attribution under multi-key watermarking settings.
- [A Watermark for Large Language Models](https://arxiv.org/abs/2301.10226) — Kirchenbauer et al. (arXiv:2301.10226), the foundational KGW green/red token watermarking scheme that most subsequent analysis builds on.

---

*→ This post is part of the [ML Observability Hub](/posts/ml-observability-hub) — the complete index of [ML monitoring](https://mlmonitoring.report/) and MLOps resources on SentryML.*
