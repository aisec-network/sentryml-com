---
title: "The Agent Authority Gap Is an Observability Problem in a Security Costume"
description: "Security vendors are pitching 'continuous observability' as the answer to ungoverned AI agents. ML platform teams already shipped most of the pipes. The missing piece is identity context inside the trace span — and that is a schema fight, not a tooling fight."
pubDate: 2026-05-03
author: "Priya Anand"
tags: ["agent-observability", "tracing", "identity", "mlops", "phoenix", "langsmith"]
category: "deep-dive"
sources:
  - title: "Bridging the AI Agent Authority Gap: Continuous Observability as the Decision Engine (The Hacker News)"
    url: "https://thehackernews.com/2026/04/bridging-ai-agent-authority-gap.html"
  - title: "OWASP Top 10 for Agentic Applications 2026"
    url: "https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/"
  - title: "The OWASP Agentic Top 10 2026: What It Means for AI Agents and Non-Human Identities (Entro Security)"
    url: "https://entro.security/blog/the-owasp-agentic-top-10-2026-what-it-means-for-ai-agents-and-non-human-identities/"
  - title: "Arize Phoenix documentation"
    url: "https://arize.com/docs/phoenix"
  - title: "Agent Drift: Quantifying Behavioral Degradation in Multi-Agent LLM Systems Over Extended Interactions (arXiv 2601.04170)"
    url: "https://arxiv.org/abs/2601.04170"
  - title: "Demystifying evals for AI agents (Anthropic)"
    url: "https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents"
  - title: "Why AI Agents Break in Production (Latitude)"
    url: "https://latitude.so/blog/why-ai-agents-break-in-production"
schema:
  type: "TechArticle"
heroImage: https://aisec-imagegen.th3gptoperator.workers.dev/featured/sentryml.com/weekly-bridging-the-ai-agent-authority-gap-continuous-observability.png
heroAlt: "The Agent Authority Gap Is an Observability Problem in a Security Costume"
---

A vendor essay made the rounds this week arguing that the way out of the AI agent governance mess is "continuous observability as the decision engine." The piece, [published on The Hacker News on April 24](https://thehackernews.com/2026/04/bridging-ai-agent-authority-gap.html) and contributed by Orchid Security, frames the problem as an "authority gap": agents are delegated actors that inherit messy, fragmented permissions from humans, service accounts, and machine identities, and security teams keep trying to govern them in isolation. The proposed fix is a sequenced one — first illuminate the inherited identity surface, then evaluate authority dynamically at runtime based on the delegator's posture, the target application, intent, and effective scope.

Read it twice and a different shape emerges. The security industry is reaching for a primitive the MLOps community has been shipping for two years: a structured trace of every model call, every tool invocation, every retrieved chunk, every retry. Phoenix, LangSmith, Langfuse, Helicone, and Datadog LLM Observability all collect that data already. What they do not collect — and what the security pitch quietly assumes you have — is the identity and authority context attached to each span. That is the actual gap, and it is not going to be closed by a new product category.

## The signal

The Hacker News article is one of three converging pressure points this quarter. The second is the [OWASP Top 10 for Agentic Applications 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/), released as a peer-reviewed framework with three of its top risks — Agent Goal Hijack (ASI01), Tool Misuse and Exploitation (ASI02), and Identity and Privilege Abuse (ASI03) — sitting squarely in observability territory. The [Entro Security walkthrough](https://entro.security/blog/the-owasp-agentic-top-10-2026-what-it-means-for-ai-agents-and-non-human-identities/) of that list is blunt: "you cannot secure [AI agents](https://techsentinel.news/posts/weekly-how-ai-assistants-are-moving-the-security-goalposts/) without securing the non-human identities and secrets that power them," and the recommended controls — short-lived dynamic credentials, JIT privileged access, agent identities tethered to a human owner — are useless without telemetry that ties them to actual runtime behavior.

The third pressure point is a January [arXiv paper on agent drift](https://arxiv.org/abs/2601.04170) that quantified semantic deviation in multi-agent LLM systems and found that drift emerged after a median of 73 interactions in their simulations. That number is ugly. It means an agent that passes pre-deployment eval can be off-spec inside a single business day in production. The paper's recommended detection method — scheduled replay of a 50–500 trace golden set, run daily or on every deploy, with sustained drops flagged as drift events — is straight out of the [ML monitoring](https://mlmonitoring.report/) playbook. We have done this for tabular models for a decade.

The signal across all three is the same: agent behavior in production is a high-dimensional time series of structured events, and someone needs to sit on top of it.

## Mechanics: what an authority-aware trace actually looks like

If you instrument a LangChain or LlamaIndex agent with [Phoenix's auto-instrumentation](https://arize.com/docs/phoenix), you get a span tree per request. The root span is the user message. Nested under it are LLM calls, retrieval calls, tool invocations, and any custom logic the developer wrapped. Each span carries timing, token counts, model version, prompt template hash, and tool input/output. Phoenix accepts traces over OTLP and is built on the OpenInference instrumentation standard, which means the same data flows into Datadog, Honeycomb, or any OTel-compatible backend.

What is missing from a default span:

- **Delegating identity.** Whose session triggered this agent? A human user? Another agent? A scheduled job? What is the chain?
- **Effective scope.** What permissions did this specific invocation carry? Not "what permissions does the service account have in IAM" — what got passed in the token at this moment.
- **Tool authority context.** When the agent called `send_email` or `query_db`, did the authority for that tool come from the original delegator, or did the agent borrow scope from somewhere else along the chain?
- **Intent classification.** Was the action consistent with the stated user intent, or did the agent's interpretation drift?

None of these are on by default in any of the major observability platforms. They are not difficult to add — Phoenix, LangSmith, and Langfuse all let you attach arbitrary attributes to spans — but the industry has not converged on a schema. Until it does, every team rolls their own, and security teams cannot write portable detection rules against trace data.

OpenInference and OpenTelemetry's GenAI semantic conventions are the obvious place for this to land. The conventions already cover model name, token counts, and tool calls. They do not yet have agreed-upon attributes for `delegating.identity`, `delegation.chain.depth`, `tool.authority.source`, or `agent.scope.effective`. Until those exist, the "continuous observability decision engine" the security vendors are selling is going to be a series of bespoke integrations against each customer's homegrown trace schema.

## What the MLOps stack already does well

Most of the agent observability platforms that matter ship the heavy lifting:

- **Trace collection at scale.** Phoenix, LangSmith, Langfuse, and Helicone all handle high-volume span ingestion with framework-aware parsing. Tool calls, retrieval steps, and intermediate model thoughts are first-class.
- **Eval harnesses.** Phoenix ships LLM-as-judge primitives, embedding drift, and dataset-level regression. LangSmith has dataset replay against [new model](https://guardml.io/posts/weekly-updating-our-model-spec-with-teen-protections/) versions. Anthropic's [evals guide](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) lays out the pattern: unit evals on discrete steps, regression suites for subjective quality, continuous production trace sampling. This is the loop that catches agent drift.
- **Cost and latency telemetry.** Token spend per request, per user, per tool. Runaway-loop detection. p95 latency by tool. The [Latitude failure mode rundown](https://latitude.so/blog/why-ai-agents-break-in-production) lists tool-call retry storms and cost explosions as the two most common production fires; both are visible in any decent trace UI.
- **Replay and diff.** LangSmith node-by-node state diffs, Phoenix trace comparisons. You can take a production trace and re-run it through a candidate model or prompt and compare outputs deterministically.

This is the part security vendors are taking credit for when they say "continuous observability." It already exists. The MLOps team probably already owns the deployment.

## Original analysis: the gap is the schema, not the tool

Here is the thesis the source articles avoid. Security teams are not actually missing observability. They are missing **agreed-upon attributes** on the spans the MLOps team is already collecting, and they are missing the political relationship that lets them push detection logic into a telemetry pipeline owned by another org.

The Orchid pitch — "evaluate the authority profile of the delegator, the context of the target application, the intent behind the requested action, and the effective scope of execution" — is a join. It joins identity data (which lives in your IdP, your secrets manager, your NHI inventory) against trace data (which lives in Phoenix, LangSmith, or Datadog). The interesting engineering question is not "should we observe agents?" — that is settled. The interesting question is where the join happens and who pays the latency cost.

Three plausible architectures, each with real tradeoffs:

1. **Inline authority enrichment at the agent runtime.** The agent framework (LangGraph, AutoGen, custom) attaches identity and scope attributes to every span at emission time. This is cleanest and lowest-latency, but every agent framework needs to learn the schema. It also assumes the runtime has access to the full identity context, which is often the actual problem the security pitch is trying to solve.

2. **Sidecar enrichment at the trace collector.** An OTel processor pulls the request's auth context from a header or correlation ID and decorates spans before they ship. This is what most enterprise security teams will end up doing because it does not require the ML team to change their code. It puts the authority logic in the security team's lane, which is also where the people who care about it sit.

3. **Post-hoc enrichment in the observability backend.** Phoenix or Datadog joins trace IDs against an audit log from your IAM or NHI platform on read. This is the easiest to ship and the worst at runtime decisions, because by the time the join happens the action has already executed.

The vendor article reads like an argument for option 1 with the implication that you need to buy their identity layer to make it work. The honest read is that most teams will land on option 2 — and that means the [ML platform](https://mlopsplatforms.com/) team is going to get tickets asking them to add a couple of OTel processors, not to rip out their observability stack.

The other piece worth naming: agents are going to fail in ways that look like drift but are actually authority leakage. An agent that starts hitting tools it has never hit before could be drifting semantically (a prompt regression, a model upgrade, a poisoned memory entry), or it could be exercising authority it inherited from a different session via a context-poisoning attack. The trace looks similar in both cases. Telling them apart requires the identity attributes on the span. ML monitoring teams that treat this as a pure quality problem will miss the security signal; security teams that treat it as a pure access-control problem will miss the drift. The runbook has to handle both.

For the offensive-side view of how those authority chains get manipulated in practice — [prompt injection](https://aisec.blog/posts/flashrt-towards-computationally-and-memory-efficient-red-tea/) that tricks an agent into impersonating a more privileged delegator — [aisec.blog](https://aisec.blog) has been tracking the techniques as they appear in the wild.

## Operational takeaway

If you own an agent in production, three things to add to the runbook this week:

**Add a `delegation.chain` attribute to your traces.** Even if it is just a list of upstream session IDs and a depth counter, get it into the span. Phoenix, LangSmith, and Langfuse all accept arbitrary span attributes. You will need this the first time an agent does something weird and the question is whether it was acting on the original user's behalf or on a downstream agent's behalf.

**Wire up a daily golden-set replay.** The [agent drift paper](https://arxiv.org/abs/2601.04170) median of 73 interactions to first semantic drift is a good forcing function. A 100–300 trace replay set that runs against current production every night, with score deltas alerting at a threshold you tune over a few weeks, will catch model regressions, prompt-template drift, and (if you also track tool-call distributions) authority anomalies. Anthropic's [eval guide](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) is a sober starting point for designing the rubrics.

**Add a "tool call entropy" alert.** Track the distribution of tools each agent identity calls. A sudden expansion of the toolset for a given delegator is one of the cleaner runtime signals for both [OWASP ASI02 (Tool Misuse)](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) and the [behavioral drift](https://arxiv.org/abs/2601.04170) the academic literature is starting to document. Cheap to compute from existing trace data, easy to baseline, easy to tune.

The vendor framing of an "authority gap" is real but partial. The full picture is that the gap closes when ML observability data carries identity context and security teams build their detection logic on top of the same trace pipeline the ML team already operates. That is a coordination problem, not a product purchase. Get the schema right, get the join right, and the "continuous decision engine" is mostly already running in your stack.

## Sources

- [Bridging the AI Agent Authority Gap: Continuous Observability as the Decision Engine (The Hacker News)](https://thehackernews.com/2026/04/bridging-ai-agent-authority-gap.html) — the contributed essay from Orchid Security that prompted this piece. Sets up the "authority gap" framing and the sequenced governance model.
- [OWASP Top 10 for Agentic Applications 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) — the peer-reviewed risk taxonomy. ASI01 (Goal Hijack), ASI02 (Tool Misuse), ASI03 (Identity and Privilege Abuse) are the three that map most directly to runtime trace data.
- [The OWASP Agentic Top 10 2026: What It Means for AI Agents and Non-Human Identities (Entro Security)](https://entro.security/blog/the-owasp-agentic-top-10-2026-what-it-means-for-ai-agents-and-non-human-identities/) — vendor walkthrough that connects the OWASP list to the non-human identity governance discussion. Useful for the JIT credential and lifecycle recommendations.
- [Arize Phoenix documentation](https://arize.com/docs/phoenix) — current state of auto-instrumentation, OTLP support, and trace schema for the major agent frameworks.
- [Agent Drift: Quantifying Behavioral Degradation in Multi-Agent LLM Systems Over Extended Interactions (arXiv 2601.04170)](https://arxiv.org/abs/2601.04170) — the source for the 73-interaction median drift figure and the three-flavor taxonomy of semantic, coordination, and behavioral drift.
- [Demystifying evals for AI agents (Anthropic)](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) — the practical guide to the eval loop the runbook section assumes.
- [Why AI Agents Break in Production (Latitude)](https://latitude.so/blog/why-ai-agents-break-in-production) — failure mode catalog. Useful for grounding the cost-explosion and retry-storm references.

## Related across the network

- [AI Agents Are Rewriting the Threat Model, and Most Security Teams Aren't Ready](https://techsentinel.news/posts/weekly-how-ai-assistants-are-moving-the-security-goalposts/) — *techsentinel.news*
- [AI Assistants Are Rewriting the Threat Model, Not Just the Workflow](https://techsentinel.news/posts/how-ai-assistants-are-moving-the-security-goalposts/) — *techsentinel.news*
- [FlashRT cuts the GPU bill on long-context prompt injection attacks](https://aisec.blog/posts/flashrt-towards-computationally-and-memory-efficient-red-tea/) — *aisec.blog*
- [AI Content Moderation: How LLM Filters Work and Where They Break](https://guardml.io/posts/ai-content-moderation/) — *guardml.io*
- [OpenAI's Under-18 Principles: a guardrail engineer reads the new Model Spec](https://guardml.io/posts/weekly-updating-our-model-spec-with-teen-protections/) — *guardml.io*


---

*→ This post is part of the [ML Observability Hub](/posts/ml-observability-hub) — the complete index of ML monitoring and MLOps resources on SentryML.*

## See also

- [LLM operations guide](https://llmops.report/)
- [ML observability tools](https://mlobserve.com/)
