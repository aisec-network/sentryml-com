---
title: "Embedding-Based Agent Monitoring Has a Blind Spot. Here's What to Watch Instead."
description: "A new paper demonstrates three attack patterns — Slow Drift, Benign Wrapper, Chaos Seeding — that defeat embedding-based detection of malicious agents in LLM multi-agent systems. The fix requires monitoring logit-level confidence, not just output embeddings."
pubDate: 2026-05-05
author: "Priya Anand"
tags: ["agents", "security", "monitoring", "multi-agent", "observability"]
category: "monitoring"
sources:
  - title: "When Embedding-Based Defenses Fail: Rethinking Safety in LLM-Based Multi-Agent Systems (arXiv 2605.01133)"
    url: "https://arxiv.org/abs/2605.01133"
  - title: "AgentDojo: A Dynamic Environment to Evaluate Prompt Injection Attacks and Defenses for LLM Agents (arXiv 2406.13352)"
    url: "https://arxiv.org/abs/2406.13352"
schema:
  type: "TechArticle"
---

A paper out this week from Zhang, Zheng, and Chen shows that the primary defense class used to detect malicious agents in LLM-based multi-agent systems — embedding-based anomaly detection — fails reliably under adversarial conditions. They demonstrate three attacks that bypass it: Slow Drift, Benign Wrapper, and Chaos Seeding. All three work because embeddings compress the wrong features.

## The signal

[The paper](https://arxiv.org/abs/2605.01133) targets multi-agent systems where LLM agents pass messages to each other to build shared context and reach collective decisions. The standard defense model is: compute a dense vector of each agent's messages, compare against the benign distribution, prune agents whose vectors are outliers. Straightforward anomaly detection.

The attack surface is the compression step. Once an attacker knows the defense is "your embedding must land near the benign cluster," generating messages that satisfy that constraint while still injecting harmful framing is a tractable optimization problem. The paper demonstrates this isn't theoretical — they implement it three different ways and validate across models and datasets.

## Mechanics

**Slow Drift** shifts message content incrementally across communication rounds, staying within the benign embedding distribution at each step while cumulatively steering group consensus. The detector sees gradual change consistent with a legitimate agent updating its position based on evidence. It never fires.

**Benign Wrapper** surrounds the adversarial payload with enough neutral boilerplate text to pull the final embedding vector back toward the benign cluster. The semantic neighborhood of the embedding is determined by the full message, not just the problematic fragment. This is essentially a higher-dimensional version of the prompt injection wrapper pattern [AgentDojo documents](https://arxiv.org/abs/2406.13352) at the tool boundary.

**Chaos Seeding** is less surgical. It injects high-variance messages that destabilize group consensus without triggering outlier detection, because the messages are diverse rather than consistently adversarial. The variance looks like a confused agent, not a hostile one.

All three work because text embeddings don't encode token-level certainty. Two messages can have nearly identical embeddings — similar semantic territory, similar phrasing — while one is a high-confidence assertion and the other is a low-confidence fabrication the model produced with flat probability distributions over the output tokens. The embedding collapses that distinction. The underlying logits do not.

The paper's proposed fix is confidence-score pruning: use token-level probability signals (logits or log probabilities) to weight or discard agent messages during aggregation. These signals "can remain informative when embeddings are not distinguishable." In testing, confidence-based pruning outperforms embedding-only approaches across the attack scenarios.

One important caveat from the analysis: confidence signal effectiveness decays over communication rounds as adversarial content gets incorporated into the group context. The authors flag this explicitly — early intervention matters more than late-stage filtering.

## Operational takeaway

Most teams running multi-agent pipelines aren't doing embedding-based agent monitoring or confidence-based monitoring. They're watching latency, error rates, token cost, and maybe output string comparisons at the pipeline level. The paper's threat model may feel remote. But its monitoring architecture is directly applicable as a design pattern, and the decay finding has immediate runbook implications.

**Start logging logprobs per agent turn.** Most OpenAI-compatible endpoints expose token-level log probabilities. If you're calling them, log `logprobs` output alongside the completion. You don't need an anomaly detector wired up on day one — having the signal in your telemetry store is the prerequisite. A sudden shift in average output confidence from a specific agent in your pipeline is detectable from that data without any specialized tooling.

**Track per-agent output variance across rounds, not just point values.** An agent participating in a genuine multi-round debate should shift position as evidence accumulates. An agent producing unnaturally consistent outputs across rounds — low embedding variance over time — is behaving like an optimized message injector, not a genuine reasoning participant. This is visible from the same trace data you're probably already emitting; it just requires a time-series view rather than a per-call view.

**Weight your alerting toward early rounds.** If the confidence signal degrades over communication rounds, your detection window is at the start of a run, not the end. For a 10-round consensus pipeline, the signal-to-noise ratio peaks around round 1-2. Calibrating tighter anomaly thresholds in early rounds — and accepting wider bounds as shared context accumulates — matches the paper's empirical finding. This is a runbook note, not a product feature: document it explicitly so whoever is on-call at 3am knows where to look first.

**Don't use embedding distance as a single hard filter.** If your current stack does clustering-based outlier detection at the trace level — Arize Phoenix's clustering views, LangSmith's grouped trace analysis — treat it as one signal in a set rather than a binary gate. The paper's three attacks are each specifically optimized against single-feature embedding detectors. A combined feature approach (embedding distance + confidence entropy + output variance) is significantly harder to satisfy simultaneously.

The broader pattern here: as multi-agent systems move from experimental to production, the monitoring model needs to shift from endpoint-level metrics to message-graph behavioral analysis. Endpoint metrics tell you a model call succeeded or failed. They don't tell you whether a participant in a multi-agent debate is systematically nudging consensus in a harmful direction.

Confidence signals are a concrete step toward that. The access constraint is real — some hosted inference providers don't expose logprobs, or expose them only at certain tiers — but it's worth knowing whether your current inference provider supports it before assuming you don't have the data.

## Sources

**[When Embedding-Based Defenses Fail: Rethinking Safety in LLM-Based Multi-Agent Systems](https://arxiv.org/abs/2605.01133)** — The source paper (arXiv 2605.01133). Theoretical analysis and empirical validation of three embedding-bypass attacks, plus the confidence-score pruning approach.

**[AgentDojo: A Dynamic Environment to Evaluate Prompt Injection Attacks and Defenses for LLM Agents](https://arxiv.org/abs/2406.13352)** — Debenedetti et al.'s benchmark for evaluating prompt injection at the tool boundary. Related failure mode: adversarial content injected via tool output rather than agent-to-agent messages, but the same gap in behavioral monitoring applies.
