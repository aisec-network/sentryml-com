---
title: "LLM Testing: A Practitioner's Guide to Evals, Metrics, and Production Monitoring"
description: "LLM testing spans offline evals, CI gate checks, and live production monitoring — three distinct jobs that need different tools. Here's how to cover all three without drowning your team."
pubDate: 2026-05-12
author: "SentryML Editorial"
tags: ["llm", "evaluation", "monitoring", "mlops", "testing", "observability"]
category: "monitoring"
sources:
  - title: "LLM Evaluation 101: Best Practices, Challenges & Proven Techniques — Langfuse"
    url: "https://langfuse.com/blog/2025-03-04-llm-evaluation-101-best-practices-and-challenges"
  - title: "Building an LLM evaluation framework: best practices — Datadog"
    url: "https://www.datadoghq.com/blog/llm-evaluation-framework-best-practices/"
  - title: "30 LLM evaluation benchmarks and how they work — Evidently AI"
    url: "https://www.evidentlyai.com/llm-guide/llm-benchmarks"
  - title: "Hallucination Metric — DeepEval by Confident AI"
    url: "https://deepeval.com/docs/metrics-hallucination"
schema:
  type: "TechArticle"
---

LLM testing is not one thing. Teams that treat it as a single activity — run a benchmark, ship the model — end up with regressions they can't explain and production failures they can't reproduce. The teams that don't get paged at 3am have drawn a line between three distinct jobs: offline evaluation before deployment, automated gate checks in CI, and continuous monitoring in production. Each requires different tooling and different signal sources.

This post covers how to build all three layers, what metrics actually matter at each stage, and which tools MLOps engineers are reaching for in 2025.

## Why traditional software testing doesn't transfer

A unit test either passes or fails. An LLM output exists on a spectrum — partially right, right for the wrong reasons, correct in tone but hallucinated on the facts. The inputs are unbounded. The behavior is probabilistic. You cannot enumerate test cases the way you can for a function that adds two integers.

This creates two problems. First, coverage is structurally incomplete — you will always encounter user inputs at runtime that your test suite never saw. Second, quality is multidimensional. A response can be factually accurate, coherent, on-topic, and still be completely useless because it answered a different question than the user asked. Any evaluation framework that collapses this to a single pass/fail metric is lying to you about model quality.

The practical answer is layered evaluation with explicit tradeoffs at each layer.

## Layer 1: Offline evaluation (pre-deployment)

Offline evals run against a fixed, curated dataset during development and before deployment. The goal is catching regressions and validating that a new model version doesn't silently degrade performance on the use cases you've already characterized.

Building a useful eval dataset requires intentional curation: happy-path queries, known edge cases, and adversarial inputs your team has encountered before. "Golden" datasets — queries paired with human-reviewed reference answers — are the highest-signal source of ground truth. Ground truth labels let you run deterministic comparisons instead of relying entirely on model-as-judge scoring.

For text generation and summarization, [BLEU and ROUGE](https://www.evidentlyai.com/llm-guide/llm-benchmarks) remain useful for overlap-based scoring. For tasks where meaning matters more than word matching — QA, dialog, reasoning — LLM-as-a-judge scoring with a capable secondary model (GPT-4o, Claude Sonnet) gives you a more meaningful signal, though it requires calibration against human judgments. [DeepEval](https://deepeval.com/docs/metrics-hallucination) ships 14+ prebuilt metrics covering faithfulness, hallucination, contextual relevancy, and toxicity, and integrates with pytest so you can gate merges on eval scores.

For RAG-specific applications, [RAGAS](https://langfuse.com/blog/2025-03-04-llm-evaluation-101-best-practices-and-challenges) is the standard choice. It measures faithfulness (fraction of claims supported by retrieved context), answer relevancy, context precision, and context recall as a linked set of metrics. Running RAGAS on a fixed QA dataset before each retrieval pipeline change gives you a regression baseline that is far more informative than eyeballing outputs.

Offline evals should be fast enough to run in CI. If your eval suite takes 45 minutes, engineers will skip it. Keep the core gate suite under 10 minutes; run the full suite overnight.

## Layer 2: CI gate checks

The CI layer is where you enforce minimum quality thresholds before a model or prompt version lands in production. This is operationally close to a deployment health check: the system has to pass before the rollout continues.

The most common gate pattern is a percentage threshold on your primary eval metric. If your QA pipeline has historically scored above 0.80 faithfulness on the golden dataset, a new retrieval model that scores 0.71 should not ship automatically. Set the threshold with headroom — not so tight that normal variation triggers false alerts, not so loose that real degradations slip through.

[LangSmith](https://www.datadoghq.com/blog/llm-evaluation-framework-best-practices/) integrates well into CI for teams already using LangChain. For framework-agnostic setups, DeepEval's pytest integration is the path of least resistance. Both let you assert on eval metrics the same way you'd assert on test coverage thresholds.

For teams operating multiple LLM applications, [LLMOps practices around CI/CD gate design](https://llmops.report) are worth reviewing — particularly the patterns for managing evaluation datasets as versioned artifacts alongside model versions.

## Layer 3: Production monitoring

This is where most teams underinvest. Offline evals tell you how the model performs on inputs you've seen before. Production monitoring tells you what's actually happening with real user traffic — which is almost always different.

The foundational instrumentation step is [tracing](https://langfuse.com/blog/2025-03-04-llm-evaluation-101-best-practices-and-challenges): capturing prompt, response, metadata (latency, token count, model version), and any intermediate steps in your pipeline as structured logs. Tools like Langfuse, [Arize](https://mlobserve.com), and Datadog LLM Observability ingest these traces and compute evaluation scores continuously. Without trace data, you are flying blind.

On top of traces, layer three jobs look like:

**Hallucination rate trending.** Run a faithfulness check against retrieved context on a sample of live traffic (10–20% is a reasonable starting point for cost management). Alert if the rolling average drops more than 5 points from your deployment baseline. A sudden increase in hallucination rate is almost always correlated with a retrieval quality problem or a prompt change that inadvertently weakened the grounding instruction.

**Topic relevancy drift.** Measure whether user conversations stay within the domain your LLM is designed to handle. Drift here is an early signal of prompt injection attempts or viral user behavior (users sharing prompts that cause your assistant to go off-rails). [Datadog's LLM observability framework](https://www.datadoghq.com/blog/llm-evaluation-framework-best-practices/) uses sentiment analysis across sessions to detect frustration patterns that precede support escalations.

**Implicit user feedback signals.** Repeated queries, conversation abandonment, and reformulation are behavioral signals that the model failed without the user explicitly saying so. These are cheaper to collect than explicit ratings and catch a different class of failure.

**Latency and cost.** p95 and p99 token-to-first-byte latency by model version, prompt template, and user cohort. Cost per query broken down by input/output tokens. Neither is quality in the traditional sense, but both degrade user experience and budget in ways that are as operationally important as hallucination rate.

## What to put in your runbook

- **Deploy gate:** Eval suite must pass before any model or prompt version is promoted. Threshold is defined per application, reviewed quarterly.
- **Hallucination alert:** Page on-call if faithfulness score drops below baseline minus 5 points for 30+ minutes.
- **Relevancy alert:** Page on-call if off-topic conversation rate exceeds 15% of session volume.
- **Latency alert:** Page on-call if p95 TTFT exceeds 3s for more than 10 minutes.
- **Dataset refresh:** Add failing production examples to the golden dataset monthly. Evals that never get new examples will stop catching real regressions.

The [ML observability tooling landscape](https://mlobserve.com) for LLMs has matured significantly over the past 18 months. Most teams with serious LLM deployments are running at least one dedicated evaluation framework alongside their general observability stack. The gap between teams that know what their model is doing in production and teams that don't is widening fast.

## Sources

- **[LLM Evaluation 101: Best Practices, Challenges & Proven Techniques — Langfuse](https://langfuse.com/blog/2025-03-04-llm-evaluation-101-best-practices-and-challenges)** — Comprehensive overview of offline and online evaluation methods, trace-based instrumentation, and the LLM-as-a-judge pattern.

- **[Building an LLM evaluation framework: best practices — Datadog](https://www.datadoghq.com/blog/llm-evaluation-framework-best-practices/)** — Covers context evaluation, user experience signals, and the production monitoring architecture Datadog uses internally and exposes via its LLM Observability product.

- **[30 LLM evaluation benchmarks and how they work — Evidently AI](https://www.evidentlyai.com/llm-guide/llm-benchmarks)** — Detailed breakdown of MMLU, HumanEval, SWE-bench, TruthfulQA, SafetyBench, and 25+ other benchmarks with descriptions of what each actually measures.

- **[Hallucination Metric — DeepEval by Confident AI](https://deepeval.com/docs/metrics-hallucination)** — Technical documentation for DeepEval's LLM-as-a-judge hallucination metric, including threshold configuration and integration with the broader eval framework.
