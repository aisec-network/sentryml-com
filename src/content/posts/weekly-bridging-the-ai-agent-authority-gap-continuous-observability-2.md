---
title: "The Authority Gap Is an Observability Problem: What MLOps Teams Should Actually Instrument"
description: "Orchid Security's framing of agent governance as a delegation problem lands in the lap of ML observability teams. The instrumentation we already own decides whether the authority graph is real or theatre."
pubDate: 2026-05-05
author: "Daniel Park"
tags: ["agent-observability", "identity", "mlops", "opentelemetry", "governance", "runbook"]
category: "deep-dive"
sources:
  - title: "Bridging the AI Agent Authority Gap: Continuous Observability as the Decision Engine"
    url: "https://thehackernews.com/2026/04/bridging-ai-agent-authority-gap.html"
  - title: "OWASP LLM06:2025 — Excessive Agency"
    url: "https://genai.owasp.org/llmrisk/llm062025-excessive-agency/"
  - title: "Semantic Conventions for GenAI agent and framework spans (OpenTelemetry)"
    url: "https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans/"
  - title: "AI Agent Observability — Evolving Standards and Best Practices (OpenTelemetry blog)"
    url: "https://opentelemetry.io/blog/2025/ai-agent-observability/"
  - title: "Best AI Observability Tools for Autonomous Agents in 2026 (Arize)"
    url: "https://arize.com/blog/best-ai-observability-tools-for-autonomous-agents-in-2026/"
  - title: "Arize AI and Google Cloud lay down standardized telemetry mandate to keep enterprise agents in check (The New Stack)"
    url: "https://thenewstack.io/ai-agent-telemetry-standardization/"
schema:
  type: "TechArticle"
---

Orchid Security spent a guest column at The Hacker News last week reframing the AI agent governance debate as a [delegation problem rather than an identity problem](https://thehackernews.com/2026/04/bridging-ai-agent-authority-gap.html). The argument is straightforward: agents do not appear in the environment with their own authority. They are triggered by users, invoked by services, or provisioned by other agents, inheriting whatever permissions sit upstream. Treating an agent as a new actor to govern misses the point. The thing to govern is the chain that hands it power.

That framing matters to anyone who runs ML in production, because the proposed control plane is one we are already supposed to own. Orchid calls it "continuous observability as the decision engine." Strip the marketing and you have a real architectural claim: every authorization decision a deployed agent makes should be conditioned on live telemetry about the agent, the operator that delegated to it, and the resource it is touching. That is not an IAM project. That is an observability project, and most of it lands on the team running ML platforms.

## The signal: governance is being rebuilt on top of telemetry

The Orchid piece is the latest in a roughly six-month run of announcements that all converge on the same idea. CrowdStrike picked up SGNL specifically to bolt identity decisions onto runtime context. Datadog [shipped native support for the OpenTelemetry GenAI semantic conventions](https://www.datadoghq.com/blog/llm-otel-semantic-convention/). Arize and Google Cloud announced a [standardized telemetry mandate for enterprise agents](https://thenewstack.io/ai-agent-telemetry-standardization/), aligning the Gemini agent service around OpenTelemetry and OpenInference. Each of those moves treats spans, traces, and events as the source of authority data, not the audit log that gets glanced at after an incident.

This is a real shift. The first wave of LLM observability looked like classic APM with a token-count column added. Tools competed on prompt logging and cost dashboards. The agent wave is different because the failure mode is different. Agents fail in ways that look like success — well-formed outputs, syntactically valid tool calls, plausible reasoning traces — and the only way to catch the bad ones is to inspect the decision sequence, not the final response. Arize's own framing puts it bluntly: in the DevOps era we monitored server health, in the MLOps era we monitored model drift, [in the agent era we monitor decisions](https://arize.com/blog/best-ai-observability-tools-for-autonomous-agents-in-2026/).

If the decisions themselves are the unit of governance, the team that owns the trace pipeline owns the governance plane. That is currently the ML platform team in most organizations.

## Mechanics: what an "authority span" actually has to carry

OWASP's [LLM06:2025 Excessive Agency](https://genai.owasp.org/llmrisk/llm062025-excessive-agency/) decomposes the failure mode into three root causes: excessive functionality (tools the agent does not need), excessive permissions (scopes broader than the task requires), and excessive autonomy (high-impact actions executed without verification). The mitigation list reads like an observability checklist:

- Run extensions in the user's security context, not a generic privileged identity.
- Track user authorization and security scope so downstream actions execute as the actual user with minimum scope.
- Log and monitor extension activity to see where undesirable actions occur.
- Rate-limit extension actions to constrain damage when something goes wrong.

You cannot enforce any of those without telemetry that knows who delegated, what scope was inherited, what tool was selected, and what resource was touched. The OpenTelemetry [GenAI agent span conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans/) — currently still experimental but stabilizing — give you the shape of that record. Agent spans carry attributes for `gen_ai.agent.id`, `gen_ai.agent.name`, `gen_ai.tool.call.id`, `gen_ai.tool.name`, and the model invocation chain that produced the call. Framework vendors layer their own semantic extensions on top.

What is missing from the standard, and what every governance vendor is now scrambling to add, is the identity edge. Who initiated the session? What OAuth subject did the agent inherit? Was the delegation step a user click, a scheduled job, or another agent passing the baton? The OpenTelemetry [agent observability working notes](https://opentelemetry.io/blog/2025/ai-agent-observability/) acknowledge that this is the next frontier: a span that says "the agent called `write_to_repo`" is useless if it cannot be joined back to the human OAuth token that authorized the session three hops upstream.

In practice, the trace your platform exports today probably has the prompt, the model name, the tool name, the latency, and the token count. It probably does not have:

- The delegating principal at the top of the chain (user, service account, or parent agent), captured as a stable identifier rather than a session token.
- The scope of credentials the agent is operating under, including any scope narrowing applied at delegation time.
- The provenance of the tool definition itself — was it loaded from a trusted registry, or did the agent discover it via MCP at runtime?
- A linkage between the model's reasoning span and the authorization decision that approved or blocked the resulting action.

Add those four attributes to every agent span and the "continuous observability as decision engine" pitch stops being a slide. It becomes a queryable graph.

## What changes in the ML platform stack

If you run an ML platform built around the typical 2025 vintage stack — Arize or WhyLabs for monitoring, MLflow or Weights & Biases for the registry, a feature store, a vector store, and some flavor of agent framework on top — the agent authority work breaks across roughly three surfaces.

**Trace ingestion.** OpenTelemetry collectors and the GenAI semantic conventions are the place to add identity attributes. Arize, Langfuse, LangSmith, and Braintrust all ingest OTel-compatible spans; if you set the attributes at the SDK boundary they show up everywhere downstream without per-vendor work. The mistake to avoid is putting identity context in the prompt or in opaque metadata blobs. It belongs in indexed span attributes so you can query "every action taken under principal X in the last 24 hours" without a text scan.

**Evaluation pipelines.** The same offline eval harness you use for hallucination scoring and tool-use correctness should grow a delegation-correctness check. Given a trace, did the agent ever exceed the scope it was delegated? Did it call a tool not in the originally provisioned set? Did it operate on a resource outside the authorized boundary? These are deterministic checks once the spans carry the right attributes, and they catch a class of regression that LLM-as-judge metrics will not.

**Drift monitoring.** This is where ML observability traditions actually pay rent. Agent behavior drifts as models update, prompts evolve, and tool catalogs expand. The drift you care about for governance is not output distribution — it is action distribution. Which tools are being called, in what frequency, against which resource types? A baseline of action distribution per agent role, plus a daily drift check, surfaces the case where a routine knowledge-retrieval agent quietly started calling `update_user_record` because someone added it to the toolset for an unrelated experiment. WhyLabs' drift primitives generalize cleanly to categorical action streams; the Evidently and Arize libraries do too. You do not need a new platform for this, you need a new metric.

## Original analysis: the authority gap is real, but the proposed cure under-prices the eval problem

The Orchid argument is correct on the structural point — agents are delegated, not autonomous, and governance has to follow the delegation chain. Where the framing under-delivers is in implying that observability alone can serve as the decision engine. Observability gives you the relational context. It does not give you the policy.

Two specific gaps stand out.

The first is that "continuous observability as the decision engine" depends on a model of acceptable behavior, and that model has to be derived from somewhere. The vendor pitch leans on dynamic risk scoring, but in practice every team that ships an agent ends up writing an explicit policy: this agent role can call these tools against these resource patterns under these conditions. That is not a thing telemetry produces. It is a thing humans write, ideally with help from the trace history. The observability layer's job is to make policy authoring tractable by surfacing real action distributions, and to enforce policy at runtime by blocking calls that violate it. Selling observability as the decision engine without acknowledging the policy substrate beneath it is how teams end up with dashboards no one consults during an incident.

The second gap is evaluation cost. If every agent action has to be conditioned on a fresh observability evaluation, the latency budget collapses. The Arize-Google Cloud telemetry mandate is partly an answer to this: standardize the trace shape so that the policy engine can read it cheaply, and so that the same engine can sit behind multiple agent runtimes without reimplementation. But the harder question — what fraction of agent decisions actually need real-time policy evaluation versus post-hoc audit — is one most vendors are dodging. A reasonable answer for most ML platforms is a tiered model: synchronous policy checks for write actions and any call that crosses a trust boundary, asynchronous evaluation for retrieval and reasoning steps, and a tight feedback loop where async findings update the synchronous policy.

The cleanest counter-argument to the entire authority-gap framing comes from the offensive side. As [aisec.blog](https://aisec.blog) and others keep documenting, [prompt injection](https://aisec.blog/posts/flashrt-towards-computationally-and-memory-efficient-red-tea/) routinely converts a legitimate delegation into an illegitimate action without ever breaking the identity chain. The agent had the right to call the tool. It just called it for the wrong reason because something it ingested told it to. Identity-graph governance does not catch that. You need content-level guardrails — input validation, output filtering, intent verification — sitting alongside the authority graph. The defensive guardrail tooling tracked at [guardml.io](https://guardml.io) is the other half of this story, and any honest pitch for observability-driven governance should name it.

The synthesis: the authority gap is real, observability is the right control plane to instrument it, and the ML platform team is the right owner. But the work splits roughly even between three things — instrumenting the identity edge in your traces, deriving policies from the resulting graph, and pairing the whole thing with content-level guardrails that catch the cases identity context cannot. Treating it as a pure IAM problem misses the telemetry. Treating it as a pure observability problem misses the policy and the prompt-layer attacks.

## Operational takeaway: what to put in the runbook this quarter

If you own an ML platform with agents in production, the work for the next 90 days is concrete.

1. **Audit your trace schema for identity attributes.** Pull a sample of agent traces from the last week. Can you answer "which human ultimately authorized this action" without leaving the trace viewer? If not, add attributes for delegating principal, OAuth subject, scope inherited, and scope narrowed at delegation. Do this in the SDK layer so it propagates to every downstream tool.

2. **Adopt the GenAI semantic conventions explicitly.** Even if your vendor stack supports them by default, name them in your platform docs. The next vendor swap is easier when your trace schema is portable, and the policy work compounds across deployments.

3. **Add an action-distribution drift monitor per agent role.** Categorical drift on tool-call frequency, resource type, and error rate. Alert on novelty — a tool called for the first time, a resource pattern outside the historical envelope. This catches both expansion of capability and the early phase of an attack.

4. **Stand up a delegation-correctness eval in your offline harness.** Given a recorded trace and the policy in force at that time, did the agent stay in scope? Run this on every model or prompt change. Treat regressions like accuracy regressions — block the deploy.

5. **Decide your synchronous-vs-asynchronous policy boundary.** Write down which classes of agent action require an inline authorization check and which can be audited after the fact. Without that line, you either pay the latency cost on every action or skip enforcement on the ones that matter.

6. **Wire the feedback loop into [ai-alert.org](https://ai-alert.org)-style incident tracking.** When an agent does something it should not have, the post-mortem belongs in the same incident database as the model regressions and the data-quality breaks. The governance signal and the reliability signal are the same signal now.

The Hacker News piece reads like a pitch deck because it is one. The underlying point is durable. The teams who already run trace pipelines, drift monitors, and offline eval harnesses are the ones with the substrate to make observability-driven governance real. The work is mostly in adding identity context to telemetry that already exists, then treating the resulting graph as a first-class platform artifact rather than an audit afterthought.

## Sources

- [Bridging the AI Agent Authority Gap: Continuous Observability as the Decision Engine](https://thehackernews.com/2026/04/bridging-ai-agent-authority-gap.html) — Orchid Security's guest analysis at The Hacker News, the news hook for this piece.
- [OWASP LLM06:2025 — Excessive Agency](https://genai.owasp.org/llmrisk/llm062025-excessive-agency/) — Canonical decomposition of agent over-privilege into excessive functionality, permissions, and autonomy, with mitigation guidance.
- [Semantic Conventions for GenAI agent and framework spans (OpenTelemetry)](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans/) — Current experimental spec for the agent span attributes any portable trace schema should carry.
- [AI Agent Observability — Evolving Standards and Best Practices (OpenTelemetry blog)](https://opentelemetry.io/blog/2025/ai-agent-observability/) — Working-group context on where the GenAI conventions are heading and what gaps remain.
- [Best AI Observability Tools for Autonomous Agents in 2026 (Arize)](https://arize.com/blog/best-ai-observability-tools-for-autonomous-agents-in-2026/) — Vendor landscape and the framing of decisions as the new monitoring unit; useful for tool selection.
- [Arize AI and Google Cloud lay down standardized telemetry mandate to keep enterprise agents in check (The New Stack)](https://thenewstack.io/ai-agent-telemetry-standardization/) — Coverage of the Arize/Google Cloud push to align the Gemini agent service on OpenTelemetry and OpenInference.

## Related across the network

- [AI Agents Are Rewriting the Threat Model, and Most Security Teams Aren't Ready](https://techsentinel.news/posts/weekly-how-ai-assistants-are-moving-the-security-goalposts/) — *techsentinel.news*
- [AI Assistants Are Rewriting the Threat Model, Not Just the Workflow](https://techsentinel.news/posts/how-ai-assistants-are-moving-the-security-goalposts/) — *techsentinel.news*
- [FlashRT cuts the GPU bill on long-context prompt injection attacks](https://aisec.blog/posts/flashrt-towards-computationally-and-memory-efficient-red-tea/) — *aisec.blog*
- [Germany names UNKN: what the BKA's REvil and GandCrab dox actually buys](https://ai-alert.org/posts/weekly-germany-doxes-unkn-head-of-ru-ransomware-gangs-revil-gandcra/) — *ai-alert.org*
- [AI Content Moderation: How LLM Filters Work and Where They Break](https://guardml.io/posts/ai-content-moderation/) — *guardml.io*
