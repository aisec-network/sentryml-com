---
title: "The Authority Gap Is an Observability Problem: What MLOps Teams Should Borrow"
description: "A new framing of AI agent risk argues that delegation, not identity, is the missing telemetry. ML platform teams already have the substrate to fix it."
pubDate: 2026-05-03
author: "SentryML Editorial"
tags: ["agents", "observability", "mlops", "telemetry", "governance"]
category: "monitoring"
sources:
  - title: "Bridging the AI Agent Authority Gap: Continuous Observability as the Decision Engine (The Hacker News)"
    url: "https://thehackernews.com/2026/04/bridging-ai-agent-authority-gap.html"
  - title: "AI Agent Observability - Evolving Standards and Best Practices (OpenTelemetry)"
    url: "https://opentelemetry.io/blog/2025/ai-agent-observability/"
  - title: "Semantic Conventions for GenAI agent and framework spans (OpenTelemetry)"
    url: "https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans/"
schema:
  type: "TechArticle"
heroImage: https://aisec-imagegen.th3gptoperator.workers.dev/featured/sentryml.com/bridging-the-ai-agent-authority-gap-continuous-observability.png
heroAlt: "The Authority Gap Is an Observability Problem: What MLOps Teams Should Borrow"
---

A piece that ran on The Hacker News last week, [Bridging the AI Agent Authority Gap](https://thehackernews.com/2026/04/bridging-ai-agent-authority-gap.html), is being read as a security argument. The framing is broader than that, and the operational fix lands squarely in MLOps.

The argument: agents are not new actors with independent authority. They are delegated actors. A human, a service account, or another machine identity invokes them, and the agent's authority is whatever the caller passed down. Treat agents as a fresh identity bucket and you miss the [structural problem](https://techsentinel.news/posts/cyber-burnout/), which is that you cannot see the delegation chain in the first place. The proposed fix is "continuous observability as the decision engine" — keep telemetry on who is delegating what to whom, and use that as the policy substrate.

If you run a model in production, this is not a security headline. This is a runbook gap.

## The signal

The Orchid Security post (the vendor behind the framing) calls the unobserved delegation surface "identity dark matter." The same shape exists for inference. A request hits a model gateway. The gateway invokes a tool-using agent. The agent calls a retrieval service, then a finetuned classifier, then a third-party API, then writes to a feature store. By the time the call graph terminates, the originating principal is three hops away, and the only thing your monitoring stack saw was a pile of `POST /v1/chat/completions` spans with no parent context worth anything.

This is the same observability gap that broke distributed tracing in microservices a decade ago, except the "service" is now a probabilistic one and the "call" includes a tool-use loop the model decided on at runtime.

## Mechanics

What's actually missing in most ML stacks isn't drift detection or token-cost dashboards. It's the trace structure that makes the delegation chain queryable.

The OpenTelemetry GenAI working group has been pushing on exactly this since 2024. Their [agent and framework span conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans/) define spans for agent invocations, tool calls, and the framework that orchestrated them, with attributes for the model, the operation name, the agent ID, and the tool name. These are still experimental, but they're stable enough that Datadog, Grafana, and the major framework SDKs (LangGraph, CrewAI, AutoGen) are emitting against them.

The conventions matter for one reason: the parent-child structure of those spans is what carries the delegation chain. If your inference service emits a `gen_ai.agent.invoke_agent` span, and the tool calls inside it propagate W3C trace context to downstream HTTP requests, you can reconstruct "this database write happened because user X asked agent Y, which called tool Z" from a trace query rather than from log forensics after an incident.

Most ML platforms today still emit one of two things: model-server metrics (latency, token counts, error rates) or framework-specific traces that don't propagate context past the SDK boundary. Neither answers the delegation question.

## What to put in the runbook

Concretely, three changes pay for themselves quickly:

**1. Make agent and tool spans first-class in your model gateway.** If you front your models with a gateway (LiteLLM, an internal proxy, or a vendor offering), require that it emits OTel GenAI spans with the agent invocation as the parent, and that it accepts and propagates `traceparent` from upstream. The [evolving conventions guidance](https://opentelemetry.io/blog/2025/ai-agent-observability/) is explicit that the agent span is the unit of analysis, not the LLM call. If your dashboards are still grouping by model name instead of agent ID, you are looking at the wrong axis.

**2. Add a "delegation context" attribute set at the gateway boundary.** At minimum: the originating principal (human user, service account, or upstream agent ID), the tool allowlist that was active for this invocation, and the data sensitivity tier the request was tagged with. None of these are in the OTel spec yet, so you'll be adding them as `genai.delegation.*` custom attributes. Do it anyway. When something misbehaves at 3am, you want to be able to filter traces by "which principal's authority was this agent acting under" without joining four log streams.

**3. Add an alert on tool-call cardinality per agent invocation.** A normal agent run hits two or three tools. A jailbroken or hallucinating one hits forty in a tight loop. This is trivially detectable from spans (`count(gen_ai.tool.call) group by trace_id`) and almost no one has it wired up. It catches both prompt-injection style attacks and the more common failure mode of an agent stuck in a retry storm against a flaky tool.

## Where this leaves the platform team

The Hacker News piece pitches continuous observability as the input to a security decision engine. For an MLOps team, that's a plausible long-term direction, but the immediate value is more boring: you finally have a trace structure that survives the move from "we serve a model" to "we serve a system of agents that call models."

Drift monitoring tooling — Arize, WhyLabs, Evidently, Fiddler — is starting to hang labels off these same spans. The vendors that win the agent-era observability story will be the ones whose ingest pipelines speak OTel GenAI semantic conventions natively, because that's the only way the trace and the model-quality signal end up on the same primary key. If your platform team is still picking a monitoring stack on demo screenshots, the question to ask is "show me an agent invocation with the tool calls and the model-quality attributes on the same span tree." A surprising number of vendors cannot.

The authority gap framing is useful even if you don't care about the security framing. It names the thing that has been quietly broken in our telemetry since the first LangChain prototype hit production: we instrumented the model and forgot to instrument the delegation.

## Sources

- [Bridging the AI Agent Authority Gap: Continuous Observability as the Decision Engine](https://thehackernews.com/2026/04/bridging-ai-agent-authority-gap.html) — The Hacker News piece, authored by Orchid Security, that motivates the delegation framing.
- [AI Agent Observability — Evolving Standards and Best Practices](https://opentelemetry.io/blog/2025/ai-agent-observability/) — OpenTelemetry's overview of where the agent semantic conventions stand and which frameworks are aligning.
- [Semantic Conventions for GenAI agent and framework spans](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans/) — The actual span and attribute spec to instrument against.

## Related across the network

- [AI Assistants Are Rewriting the Threat Model, Not Just the Workflow](https://techsentinel.news/posts/how-ai-assistants-are-moving-the-security-goalposts/) — *techsentinel.news*
- [AI Agents Are Rewriting the Threat Model, and Most Security Teams Aren't Ready](https://techsentinel.news/posts/weekly-how-ai-assistants-are-moving-the-security-goalposts/) — *techsentinel.news*
- [FlashRT cuts the GPU bill on long-context prompt injection attacks](https://aisec.blog/posts/flashrt-towards-computationally-and-memory-efficient-red-tea/) — *aisec.blog*
- [OpenAI's Under-18 Principles: a guardrail engineer reads the new Model Spec](https://guardml.io/posts/weekly-updating-our-model-spec-with-teen-protections/) — *guardml.io*
- [Cybersecurity Burnout Is a Structural Problem, Not a Personal One](https://techsentinel.news/posts/cyber-burnout/) — *techsentinel.news*
