---
title: "ML Observability Hub: SentryML's Guide to Production LLM Monitoring"
description: "The central resource index for ML observability and MLOps on SentryML — agent telemetry, drift detection, detection engineering, and production failure patterns, organized for ML platform teams."
pubDate: 2026-05-11
author: "Priya Anand"
tags: ["hub", "mlops", "observability", "agents", "monitoring", "detection-engineering", "drift"]
category: "hub"
draft: true
schema:
  type: "Article"
---

Most [ML monitoring](https://mlmonitoring.report/) failures happen not because teams lacked tooling, but because they were watching the wrong signals. Embedding similarity scores look fine while a malicious agent drifts its behavior across 40 runs. Model latency dashboards are green while logit-level confidence on critical decisions collapses. Trace spans capture every inference call while the action catalog that actually bounds agent behavior goes completely unmonitored.

SentryML covers ML observability from the perspective of engineers who've been handed a production LLM deployment and need to keep it behaving correctly — not from the perspective of vendors who want to sell you a dashboard. That means we cover what signals actually indicate problems, what the open-source and commercial tooling does and doesn't catch, and how to build detection layers that hold up under adversarial conditions.

The coverage clusters around three areas. **Agent observability** is where most of the active research is right now: how to instrument multi-agent systems, how to detect behavioral drift and malicious agent substitution, and how to build the kind of trace spans that make authorization auditable rather than theatrical. **Detection engineering for LLM apps** covers translating MITRE ATLAS techniques into actual SIEM rules, log shapes, and runbooks — the operational layer that turns threat intelligence into actionable alerts. **Production tooling and infra** covers the practical stack decisions: local inference serving, embedding-based monitoring, and what happens to your observability obligations when you move off managed cloud APIs.

This page is a living index. Add it to your reference stack and return to it as the site grows.

---

## Agent Observability

The fastest-moving area in ML monitoring. Autonomous agents introduce authorization, identity, and behavioral change signals that standard LLM monitoring pipelines weren't designed to capture.

**[The Authority Gap Is an Observability Problem: What MLOps Teams Should Borrow](/posts/bridging-the-ai-agent-authority-gap-continuous-observability)**
Security vendors are framing the "AI agent authority gap" as a new product category. ML platform teams already have most of the instrumentation substrate — the missing piece is identity context inside trace spans, and that's a schema design problem, not a tooling purchase.

**[The Authority Gap Is an Observability Problem: What MLOps Teams Should Actually Instrument](/posts/weekly-bridging-the-ai-agent-authority-gap-continuous-observability-2)**
A deeper cut on the same topic: what the OpenTelemetry GenAI semantic conventions give you, where they fall short on delegation auditing, and the specific span fields that matter for agentic governance.

**[The Agent Authority Gap Is an Observability Problem in a Security Costume](/posts/weekly-bridging-the-ai-agent-authority-gap-continuous-observability)**
The schema fight at the center of agent telemetry standardization. Phoenix, LangSmith, and the emerging OTel conventions all make different tradeoffs on where identity context lives in the trace.

**[A Lean 4 stability proof for tool-mediated LLM agents, and what it means for your runbook](/posts/stable-agentic-control-tool-mediated-llm-architecture-for-au)**
A formal verification result with a practical implication: monitor the action catalog, not the model weights. A certified-stable Claude Sonnet 4 controller still produces zero variance across 40 runs at four temperatures — the action boundary is what actually constrains behavior.

---

## Embedding-Based Monitoring: Where It Fails

**[Embedding-Based Agent Monitoring Has a Blind Spot. Here's What to Watch Instead.](/posts/when-embedding-based-defenses-fail-rethinking-safety-in-llm)**
Three adversarial attack patterns — Slow Drift, Benign Wrapper, Chaos Seeding — reliably defeat embedding-based malicious agent detection. The fix requires monitoring logit-level confidence distributions, not output embeddings. Covers what to instrument and why standard cosine-similarity anomaly detection misses these attack classes.

---

## Detection Engineering

Translating threat intelligence into operational monitoring: log shapes, alert heuristics, runbook structure.

**[Detection Engineering for LLM Apps: A MITRE ATLAS-Mapped Runbook for Prompt Injection Alerting](/posts/llm-detection-engineering-mitre-atlas-runbook)**
A complete treatment: how to map LLM application telemetry to MITRE ATLAS techniques, what log shapes to emit for injection and exfiltration events, and how to structure a runbook that doesn't collapse under alert volume.

---

## Local Inference and Observability Ownership

**[Local Coding Assistants Have Crossed the Quality Bar — Now You Own the Observability](/posts/if-you-ve-been-waiting-to-try-local-ai-development-please-tr)**
When you move from cloud-managed APIs to local inference, usage metering, request [logging](https://mlobserve.com/), and safety filtering all become your problem. A practical look at what monitoring obligations shift when you run Qwen3.6-27B locally instead of calling a vendor endpoint.

---

## What this site covers

**[What this site is for](/posts/welcome)**
Full scope statement: drift detection, production failure writeups, tooling reviews, and what we mean by "ML observability" as distinct from general IT monitoring.

---

## Cross-Site Reading

SentryML covers the monitoring and detection layer. For the offensive attacks that your monitoring needs to catch, see [aisec.blog](https://aisec.blog). For the guardrail and content filter layer that sits in front of monitoring, see [GuardML](https://guardml.io).
