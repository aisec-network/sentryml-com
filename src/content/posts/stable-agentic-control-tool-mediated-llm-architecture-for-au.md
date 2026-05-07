---
title: "A Lean 4 stability proof for tool-mediated LLM agents, and what it means for your runbook"
description: "A new arXiv paper certifies controllability and ISS robustness for an LLM-driven SOC agent using Lean 4. The MLOps takeaway is simpler than the math: monitor the action catalog, not the model."
pubDate: 2026-05-06
author: "Priya Anand"
tags: ["agents", "observability", "formal-methods", "llm-monitoring", "mlops"]
category: "monitoring"
sources:
  - title: "Stable Agentic Control: Tool-Mediated LLM Architecture for Autonomous Cyber Defense (arXiv:2605.03034)"
    url: "https://arxiv.org/abs/2605.03034"
  - title: "The Path To Autonomous Cyber Defense (arXiv:2404.10788)"
    url: "https://arxiv.org/abs/2404.10788"
schema:
  type: "TechArticle"
---

A paper landed on arXiv this week that is worth the attention of anyone running an LLM agent in production, even if your agent never goes near a security operations center. The authors put a tool-mediated LLM controller through a Lean 4 proof, with zero `sorry`, certifying controllability, observability from asymmetric sensor data, and Input-to-State Stability (ISS) under intelligent adversarial disturbance. The empirical evaluation is on 282 real enterprise attack graphs ([Prinos et al., 2026](https://arxiv.org/abs/2605.03034)).

The applied result is the part that matters for ML platform teams: a tool-mediated Claude Sonnet 4 controller cut the attacker's expected game value by 59% versus a deterministic greedy baseline, with **zero variance across 40 runs at four temperatures**. A Claude Haiku 4.5 controller in the same harness converged to suboptimal payoffs but stayed catalog-bounded across an additional 40 runs. Quoting the abstract: "architectural stability is not dependent on the controller capability."

That last sentence is the one to pin to the wall.

## The signal: stability lives in the architecture, not the model

The mechanism is straightforward. Instead of letting an LLM emit free-form actions into the environment, the agent calls deterministic tools — Stackelberg best-response, Bayesian observer updates, attack-graph primitives — and selects from a finite action catalog enforced at the tool-output interface. The Lyapunov argument is built around that interface. Any controller from the catalog inherits the certificate. Any adversary from the catalog is bounded by the same machinery. The LLM contributes exploration; the tool boundary contributes guarantees.

If you have ever stared at a noisy production trace from an agent and tried to figure out why it just paged a customer at 2am, the punchline is familiar in shape. The unbounded part of the system (the prompt-conditioned generator) is where novelty and degradation live. The bounded part (validated tools, typed outputs, allow-listed actions) is where you write your invariants. This paper formalizes that intuition with control theory and a machine-checked proof, but the engineering pattern is portable.

## Mechanics: what the certificate actually buys you

Three operational properties fall out of the proof, each of which has a direct monitoring analog you can implement without Lean.

**Catalog membership as an invariant.** Every action the agent emits is tagged with a tool identity and falls inside a finite set. You log it; you alert on misses. The Haiku 4.5 result is the proof of value here: a weaker controller did not break the catalog, it just made worse choices inside it. In MLOps terms, that means a model swap, a quantization, or a context-pressure regression should not change your safety surface at all. If it does, your invariant is wrong, not your model.

**Game-value variance as a drift signal.** Sonnet 4 produced zero variance in payoff across 40 runs and four temperatures. That is a very loud number. Anyone running agents knows that temperature-sweeping a free-form policy normally produces a distribution wide enough to drive a truck through. Zero variance means the policy decisions that matter are happening inside the deterministic tools. Track payoff variance per release; sudden inflation tells you the model is now reaching past the tool boundary, or the catalog has grown.

**Asymmetric observability.** The certificate explicitly handles the case where you do not get clean signal back from the environment, only sensor data with gaps and lag. SOCs live there. So does any production ML system whose ground truth is delayed by hours or days (fraud, churn, recommendation outcomes). The takeaway is the same: observability of the controller does not require observability of the world, as long as you instrument the tool calls themselves. Tool-call traces are the ground-truth analogue.

The paper sits naturally next to ORNL's earlier survey of the autonomous cyber defense problem ([Oesch et al., 2024](https://arxiv.org/abs/2404.10788)), which laid out the operational pressure on SOCs but stopped short of formal guarantees. The new work is a concrete answer to a question that survey raised: how do you put one of these agents in front of an EDR policy without inviting an incident review.

## Operational takeaway: what to add to monitoring this quarter

If you operate an agentic LLM system, three changes are cheap and pay off quickly.

Treat the action catalog as a first-class monitored surface. Whatever your agent framework is — LangGraph, an in-house orchestrator, an MCP-style tool server — emit a metric per call with `(tool_name, schema_hash, model_id)` labels. Alert on any new tuple that has not been seen in the last 30 days. This catches catalog growth (often a quiet PR that loosens a JSON schema) before it catches you.

Split your performance metric from your stability metric. Decision quality (game value, completion rate, downstream KPI) is your performance signal. Catalog-membership rate, schema-validation pass rate, and tool-error rate are your stability signals. Tools like [Arize](https://arize.com/) and [WhyLabs](https://whylabs.ai/) will happily ingest both, but most teams I see conflate them on a single dashboard and lose the architectural-stability claim that this paper just proved is real. Keep them separate. Page on stability regressions. Review performance regressions weekly.

Record the temperature, the model id, and the tool-id sequence on every span. The Sonnet vs Haiku result in the paper is only legible because they held the harness fixed. You want the same forensic property when a customer reports something weird. If you cannot diff a misbehaving trace against a known-good trace at the tool-call level, you do not have observability over your agent — you have observability over your LLM, which is a much weaker claim.

## What this does not do

The certificate is over the controller-environment loop, given the tool catalog. It does not certify the tools themselves. If your Bayesian observer has a bug, the Lyapunov argument inherits the bug. The same is true in production: validating that an agent only ever emits `refund(order_id, amount)` does nothing for you if `refund` itself is broken. The pattern shifts where you invest verification effort, it does not eliminate the need to invest it. Test your tools as if they were a public API, because to your agent, they are.

The other limitation is generality. The action catalogs in the paper are SOC-specific (EDR policy moves, attack-graph countermeasures). Building a catalog tight enough for a Lyapunov proof in a domain like customer support or coding agent territory is a real engineering task. Most teams will get the operational benefit (monitorable invariants, model-swap stability) without ever touching Lean. That is fine. The architectural pattern is the deliverable; the proof is the receipt.

## Sources

- [Stable Agentic Control: Tool-Mediated LLM Architecture for Autonomous Cyber Defense](https://arxiv.org/abs/2605.03034) — Prinos et al., arXiv:2605.03034 (May 2026). The primary source. Includes the Lean 4 certificate, the 282-attack-graph evaluation, and the Sonnet 4 / Haiku 4.5 game-value comparison.
- [The Path To Autonomous Cyber Defense](https://arxiv.org/abs/2404.10788) — Oesch et al., arXiv:2404.10788 (April 2024). ORNL survey framing the SOC operational problem the new paper is responding to.
