---
title: "Embedding-based defenses for multi-agent LLMs are failing. Your monitoring stack needs to catch up."
description: "A new paper shows three attacks that slip past embedding-based detectors in LLM multi-agent systems. The fix lives at the logits layer, which most observability tooling still does not capture."
pubDate: 2026-05-11
author: "SentryML Editorial"
tags: ["multi-agent", "llm-observability", "drift", "logprobs", "monitoring", "guardrails"]
category: "deep-dive"
sources:
  - title: "When Embedding-Based Defenses Fail: Rethinking Safety in LLM-Based Multi-Agent Systems"
    url: "https://arxiv.org/abs/2605.01133"
  - title: "PoisonedRAG: Knowledge Corruption Attacks to Retrieval-Augmented Generation of Large Language Models"
    url: "https://arxiv.org/abs/2402.07867"
  - title: "OpenAI Cookbook — Using logprobs"
    url: "https://developers.openai.com/cookbook/examples/using_logprobs"
  - title: "Arize Phoenix — LLM Traces"
    url: "https://arize.com/docs/phoenix/tracing/llm-traces"
  - title: "Benchmarking Poisoning Attacks against Retrieval-Augmented Generation"
    url: "https://arxiv.org/abs/2505.18543"
schema:
  type: "TechArticle"
---

A new paper from researchers at Rice and the University of Virginia argues that the dominant defense pattern for LLM-based multi-agent systems — train an embedding model to flag suspicious messages and prune the agent that sent them — fails against a class of attacks that look statistically benign in the embedding space. The work, [When Embedding-Based Defenses Fail: Rethinking Safety in LLM-Based Multi-Agent Systems](https://arxiv.org/abs/2605.01133), introduces three concrete attacks (Slow Drift, Benign Wrapper, Chaos Seeding) that drive accuracy on MMLU, GSM8K, and BBH down by double digits across LLaMA-3.1-8B, Qwen3-4B, and GPT-4o-mini under standard star, chain, and sparse-graph topologies. The proposed mitigation does not sit at the embedding layer at all. It reads the token logits the model already produced and prunes or down-weights messages by per-token confidence.

For anyone running a multi-agent product in production, the practical question is whether your observability stack actually captures the signal the paper says you need. Most do not.

## The signal: three attacks that hug the benign manifold

The threat model the authors care about is straightforward. You have a multi-agent system where agents exchange natural-language messages over several rounds before producing a group decision. One or more agents are compromised. A defender sits between the agents and looks at the embeddings of each message, hoping to spot anomalous ones and route around the bad actor.

That defense pattern shows up in production tooling. Embedding-similarity flags, RAG retrieval poisoning checks, prompt-injection classifiers built on sentence-transformer features — all of them assume the malicious distribution lives somewhere measurably distant from the benign one. The paper formalizes why that assumption is fragile, then breaks it three different ways.

**Slow Drift** generates messages that stay close to the rolling benign embedding support across rounds. No round-over-round jump is large enough to trip a change-based detector, but the steady pull moves the group's reasoning toward the attacker's target answer. It is the multi-agent analogue of long-horizon [model drift](https://mlmonitoring.report/): nothing alarming on a single observation, plenty of damage in aggregate.

**Benign Wrapper** concatenates a long, legitimate-looking preamble with a short malicious directive. Because mean-pooled embeddings weight every token roughly equally, the wrapper dominates the vector and the payload rides along underneath. This is the same arithmetic that lets adversaries hide injected instructions inside RAG documents — research compiled in the [PoisonedRAG paper](https://arxiv.org/abs/2402.07867) showed five malicious documents in a corpus of millions can yield 90 percent attack success when the retriever scores on cosine similarity.

**Chaos Seeding** is the most interesting of the three because it does not try to imitate benign messages at all. Instead it deliberately fragments the benign cluster. The attacker pushes legitimate agents toward conflicting answers, the benign embedding region widens to cover that disagreement, and the malicious messages no longer look like outliers because there are no outliers left — everything is an outlier. The defender's "benign region" eats itself.

All three attacks share a property: they reduce the separability of embeddings between malicious and benign messages, which is the only signal embedding-based defenses get to use. The authors prove this formally in the paper's analysis section and validate it across the three reasoning benchmarks. Under Chaos Seeding on GSM8K with LLaMA-3.1-8B in a star topology, accuracy collapses from above the no-attack baseline to numbers comparable to single-agent performance.

## Mechanics: why logits survive when embeddings do not

The defense the authors propose reaches one layer deeper. When a model generates a message, it also produces a per-token logit distribution. Convert those to probabilities, compute entropy or take the top-token probability, and you have a per-token confidence signal. Aggregate that across a message and you get something the attacker cannot trivially shape, because to drive token-level confidence up the malicious agent has to actually believe — at the language-model level — what it is saying. Crafting a Benign Wrapper that looks normal in embedding space is cheap. Crafting one whose every token is high-confidence under the receiving agent's own model is much harder, especially when the malicious payload contains specific false claims the model would not produce on its own.

The paper offers two operational variants. **Confidence pruning** filters messages whose aggregate confidence falls below a threshold before they enter the chat history. **Confidence down-weighting** keeps the message but attaches confidence metadata, and receiving agents are prompted to discount uncertain content during their reasoning. The pruning variant works better when you have control over the orchestrator and can drop messages outright. The down-weighting variant is the one you reach for when the agents are running in someone else's framework and you can only annotate.

This is where the gap with current observability tooling shows up. The OpenAI Chat Completions API exposes a `logprobs` parameter that returns per-token log probabilities and the top-k alternatives, with practical guidance in the [OpenAI cookbook on confidence and perplexity scoring](https://developers.openai.com/cookbook/examples/using_logprobs). Anthropic does not currently expose token-level logprobs in its production API at the same fidelity, which means a Claude-based agent in a mixed-vendor system gives you less signal to work with. If you are building a Slow Drift detector, the asymmetry matters.

## What your observability stack actually captures

A reasonable mid-2026 LLM stack looks like this: agents instrumented through OpenTelemetry, traces shipped to [Arize Phoenix](https://arize.com/docs/phoenix/tracing/llm-traces) or a comparable OTel-native backend, evals run on a schedule, and a guardrails layer somewhere in the request path doing prompt-injection and PII checks. Phoenix records the typical spans — retriever calls and their scores, embedding text and model, prompt templates, tool calls, latency, token counts. That is excellent for debugging an agent loop that misbehaved on a single trace. It is not what you need to detect Slow Drift across a hundred concurrent sessions.

The specific gaps:

**Per-token logprobs are usually dropped.** Most tracing libraries capture the response text and metadata but not the underlying token distribution, because the payload is large and the value was not obvious. The new paper makes the value obvious. If you are running an OpenAI-backed agent, set `logprobs: true` and `top_logprobs: 5` and persist the response.

**Inter-agent message embeddings are computed but not summarized.** If you are doing embedding-based outlier detection at all, you probably compute the embedding, run it through an anomaly score, and discard the vector. To detect Chaos Seeding you need to track the *spread* of the benign cluster over time, not just per-message distance from a static centroid. That means storing the embedding population per session, not just the score.

**Confidence trends do not have a place in the dashboard.** Drift dashboards in Evidently, WhyLabs, and similar tools were built for tabular feature drift and embedding drift. None of them treat aggregate token confidence as a first-class series. You either bolt it on as a custom metric or add a span attribute and chart it from your trace store.

**Multi-agent traces are still flattened.** Phoenix's recent agent-graph visualization helps, and CrewAI / LangGraph integrations expose the topology. But none of the major platforms surface a "benign embedding region radius over time, by session" view — the thing the Chaos Seeding paper says you should be watching.

## Original analysis: the monitoring layer has been pointed at the wrong primitive

The broader story here is that the LLM observability industry, including the vendors SentryML covers most often, has been monitoring text and embeddings because those are the primitives ML monitoring already knew how to handle. Embeddings come from a tabular-flavored world: you have a vector, you have a baseline distribution, you compute KL divergence or PSI or a contrastive score, you alert. The whole stack from Evidently to [the ML observability ecosystem](https://mlobserve.com) inherited that pattern from the pre-LLM era.

But the security-relevant signal in LLM systems lives one layer down, in the logit distribution the model produces. Logits are noisier, larger, and harder to summarize. They do not fit into a feature-drift dashboard. They require new aggregations — average per-token entropy, fraction of tokens below a confidence threshold, perplexity per turn — and new baselines, because "what does a confident benign message look like in this model" depends on the model, the temperature, the prompt template, and the agent's role.

There is a counter-argument worth taking seriously. The confidence defense in the paper is not unconditional. The authors note that effectiveness diminishes over many communication rounds, which is consistent with what you would expect: any defense that depends on the malicious agent being uncertain breaks once the malicious agent gains enough context to be confidently wrong. An attacker who controls a fine-tuned model can produce high-confidence malicious tokens at will, and the [recent RAG poisoning benchmark](https://arxiv.org/abs/2505.18543) found that broadly capable attacks degrade hard on out-of-distribution datasets, which is another way of saying defenses do too. Confidence pruning is a better first line than embedding clustering. It is not a moat.

The synthesis I would push to anyone running a multi-agent product is this: stop treating embedding-based anomaly detection as the load-bearing layer. Treat it as a coarse filter that catches the lazy attacks. Put a confidence-based check immediately behind it, and put behavioral evals (does this agent's final answer match a ground truth on a sampled subset?) behind that. Three filters, declining in cost-per-check and increasing in fidelity. The paper gives you the middle layer for free.

## Operational takeaway: what to add this week

If you operate an MAS in production, the runbook changes are concrete.

**Capture logprobs at the orchestrator.** For OpenAI-backed agents, enable `logprobs` in the API call and ship the per-token values to your trace store. Storage cost is non-trivial — figure 5-10x the response text size on average — so sample if you must. For Anthropic agents, you do not have first-class logprobs, so log the response and run a separate scoring pass with a small open model (Qwen3-4B is what the paper used) when you need confidence numbers. Track [the state of Anthropic logprobs support](https://github.com/anerli/anthropic-logprobs) if this matters to you.

**Add a per-message confidence metric to your tracing schema.** Mean per-token probability, fraction of tokens below 0.5, and message-level perplexity are the three to start with. Chart them per agent role over time, not just per call. Slow Drift shows up as a slow downward trend on confidence even when no individual message looks anomalous.

**Track embedding-region radius, not just per-message distance.** Compute the convex hull or simply the variance of message embeddings per session. Alert on sudden widening, which is the Chaos Seeding signature. A static centroid is not enough.

**Add a "wrapper ratio" feature for long messages.** Split each message into segments, embed each, and look for messages where one short segment is far from the others while the overall mean-pooled vector looks benign. This is a cheap Benign Wrapper detector and does not require model changes.

**Run an offline replay of the three attacks against your current defense.** The paper's threat models are simple enough to reproduce. Pick a benchmark your agents already evaluate against, inject a malicious agent following each of the three patterns, and measure how much your existing guardrails detect. If the answer is none, you have a roadmap.

The bigger point: the security-research community is now publishing attacks that are specifically designed to evade the things ML observability tools were built to measure. That gap will not close on its own. It closes when monitoring vendors expose logit-level metrics as first-class objects, when tracing libraries stop discarding the most security-relevant field in the API response, and when "is my agent confident in what it is saying" sits next to "is my latency in budget" on the same dashboard.

For sister-site coverage of the offensive side of this work, see the writeups on [prompt injection in agentic systems](https://promptinjection.report) and [adversarial ML technique tracking](https://adversarialml.dev). The intersection — where attack research and observability practice meet — is where the interesting operational decisions are happening this quarter.

## Sources

- [When Embedding-Based Defenses Fail: Rethinking Safety in LLM-Based Multi-Agent Systems](https://arxiv.org/abs/2605.01133) — The Zhang, Zheng, and Chen paper that introduces the Slow Drift, Benign Wrapper, and Chaos Seeding attacks and proposes confidence-based pruning and down-weighting as the replacement defense.
- [PoisonedRAG: Knowledge Corruption Attacks to Retrieval-Augmented Generation of Large Language Models](https://arxiv.org/abs/2402.07867) — Zou et al., USENIX Security 2025. Establishes that five well-crafted poisoned documents in a corpus of millions can yield 90% attack success against retrieval-based pipelines, the precedent for embedding-space evasion that the new paper extends to inter-agent messaging.
- [OpenAI Cookbook — Using logprobs](https://developers.openai.com/cookbook/examples/using_logprobs) — Reference for enabling token-level log probabilities through the Chat Completions API and using them for confidence thresholds, perplexity scoring, and retrieval evaluation. The practical entry point for capturing the signal the new defense depends on.
- [Arize Phoenix — LLM Traces](https://arize.com/docs/phoenix/tracing/llm-traces) — Documentation for the OTel-native tracing primitives that most multi-agent stacks already use. Covers what Phoenix captures by default (retriever scores, embedding text, prompt templates) and where confidence-based metrics would need to be added as custom span attributes.
- [Benchmarking Poisoning Attacks against Retrieval-Augmented Generation](https://arxiv.org/abs/2505.18543) — Comprehensive benchmark of 13 attack methods and 7 defenses across standard and expanded QA datasets. Useful counter-evidence that no current defense generalizes robustly out-of-distribution, which constrains how much you should trust any single mitigation including the one proposed in the new paper.

## See also

- [LLM operations guide](https://llmops.report/)
- [MLOps platform comparisons](https://mlopsplatforms.com/)
