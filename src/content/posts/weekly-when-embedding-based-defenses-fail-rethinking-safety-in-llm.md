---
title: "When Embedding-Based Defenses Fail: What MLOps Teams Need to Log in Multi-Agent LLM Systems"
description: "A new arXiv paper shows that embedding-distance detectors miss three classes of adversarial agent. The fix lives in your observability stack, not your prompt template."
pubDate: 2026-05-11
author: "SentryML Editorial"
tags: ["multi-agent", "observability", "llm-monitoring", "agent-telemetry", "drift-detection", "mlops"]
category: "deep-dive"
sources:
  - title: "When Embedding-Based Defenses Fail: Rethinking Safety in LLM-Based Multi-Agent Systems (Zhang, Zheng, Chen — arXiv 2605.01133)"
    url: "https://arxiv.org/abs/2605.01133"
  - title: "MASpi: A Unified Environment for Evaluating Prompt Injection Robustness in LLM-Based Multi-Agent Systems (OpenReview)"
    url: "https://openreview.net/forum?id=1khmNRuIf9"
  - title: "NIST AI Risk Management Framework"
    url: "https://www.nist.gov/itl/ai-risk-management-framework"
  - title: "OpenTelemetry Semantic Conventions for Generative AI"
    url: "https://opentelemetry.io/docs/specs/semconv/gen-ai/"
  - title: "Prompt Injection Attacks in Large Language Models and AI Agent Systems (MDPI Information, 2026)"
    url: "https://www.mdpi.com/2078-2489/17/1/54"
schema:
  type: "TechArticle"
---

A new paper out of arXiv this month, [*When Embedding-Based Defenses Fail: Rethinking Safety in LLM-Based Multi-Agent Systems*](https://arxiv.org/abs/2605.01133), takes a hammer to one of the more popular shields shipped with production agent frameworks: detect-and-prune routines that flag a malicious peer by the cosine distance of its messages from a benign cluster. The authors — Lingxi Zhang, Guangtao Zheng, and Hanjie Chen — argue, with a theorem and three working attacks, that embedding-based detectors share a single load-bearing assumption, and that assumption does not hold against an attacker who has read the same survey papers your defense team did.

If you operate a multi-agent system in production — a research assistant with a planner/critic split, a customer-service router that delegates to specialist agents, anything with `tools` plumbed through more than one LLM call — the operational takeaway is not that you need a new vendor product. It is that the signals you are almost certainly already capturing in your inference layer (logits, per-token confidence, log-probabilities) are the ones you need to start treating as first-class telemetry, not as debugging exhaust.

This piece walks through what the paper actually shows, why the failure mode is structural rather than incidental, what to add to your runbook, and where the proposed mitigation runs into operational friction that the paper does not address.

## The signal: three attacks that route around embedding distance

The paper's setup is the standard collaborative MAS configuration: a small population of LLM agents exchange messages over several rounds to converge on an answer, and the system runs a side-channel detector over those messages. The detector embeds each message — typically with the same sentence-transformer used for retrieval elsewhere in the stack — and looks for outliers in embedding space. A message whose embedding sits far from the benign centroid gets pruned, or its source agent gets quarantined.

The attacker's job is straightforward: produce messages that are *adversarially close* to benign ones in embedding space but still steer the group toward a poisoned conclusion. Zhang et al. demonstrate three concrete realizations:

- **Slow Drift.** The malicious agent introduces a tiny per-round perturbation toward the target conclusion. No single message moves far enough from the benign cluster to trip the detector, but the cumulative effect across rounds is a hijacked vote.
- **Benign Wrapper.** The malicious payload is embedded inside a long preamble of benign content. Because sentence-level embeddings are essentially averages over token representations, padding with benign tokens drags the embedding back into safe territory.
- **Chaos Seeding.** The attacker injects high-entropy noise into otherwise benign-shaped messages, increasing the variance of the benign cluster itself. Once the cluster's effective radius widens, real malicious payloads slip inside the new boundary.

The theoretical argument is the part worth lingering on. Embedding-based detectors are, formally, a thresholded distance in a fixed representation space. If the attacker has white-box or even gray-box access to that representation — and in practice they usually do, because most teams use one of three off-the-shelf sentence encoders — there always exists a manifold of payload-bearing messages that the detector cannot separate from benign ones. This is not a tuning problem. It is the same kind of structural failure you get when you try to detect prompt injection with a regex over the input string.

The companion observation is what unlocks the mitigation. Embedding distance throws away information the underlying generator already computed: the next-token logit distribution, and from it the per-token log-probability of every token in the message. Under several of these attacks, the *embedding* of the message is benign, but the *generation trajectory that produced it* is anomalous. Slow Drift messages often have lower-than-expected token confidence at the drift-bearing positions. Benign Wrapper messages have a confidence cliff at the boundary between the wrapper and the payload. Chaos Seeding messages have token-level entropy that looks nothing like a normal model decode.

Zhang et al. propose a defense that uses these confidence signals — either logits from the message author or perplexity scores from an auditor model — to down-weight messages whose generation trajectory looks suspicious, even when their embeddings do not. Their headline result is that the confidence-augmented defense improves robustness across multiple model families and communication topologies, with the caveat that effectiveness degrades over communication rounds. The takeaway: catch it early, or you do not catch it at all.

## Mechanics: why this is an observability problem, not a model problem

Step back from the paper for a moment and the deeper story is uncomfortably familiar to anyone who has been on call for an ML platform. The failure mode is the same one we have been living with in tabular ML for a decade: a feature whose univariate distribution looks fine but whose conditional structure has shifted. Drift-detection libraries learned this lesson the hard way, which is why modern monitoring stacks ship multivariate detectors, conditional KS tests, and integrated reconstruction-error baselines instead of just thresholding the L2 distance of an embedding from a centroid.

Embedding-distance detection in MAS is essentially univariate distribution monitoring with extra steps. It collapses a high-dimensional generative process down to a scalar distance, throws away the rest of the signal the model literally just computed, and then asks one threshold to do the work of a full anomaly-detection pipeline. When the planner-side team frames this as "we need a better detector model," they are reaching for a bigger hammer. The thing that broke is the choice to instrument only one channel.

The paper's confidence-signal mitigation lands cleanly in this frame. Logits and log-probabilities are not new telemetry; they are signals every inference server already produces and most teams discard before they hit a log pipeline. vLLM exposes them, TGI exposes them, the OpenAI and Anthropic APIs both expose them under their respective `logprobs` parameters, and llama.cpp prints them by default. The cost of capturing them is a tiny multiple of the request body. The cost of *retaining* them, in a system that fires hundreds of agent-to-agent messages per user request, is where the work actually lives.

This is the part where the observability angle stops being abstract. Your inference layer is already producing the signal. Your monitoring layer is almost certainly not consuming it. That gap is the gap the paper is exploiting.

## Operational takeaway: what to put in the runbook

For ML platform and MLOps teams running production MAS, there are a handful of changes worth queuing up before the next sprint cuts off.

**Instrument agent messages with per-token confidence, not just text.** Configure your inference endpoints to return logprobs (or top-k logprobs) for every agent-to-agent message, and persist them alongside the message text in your observability store. If you are using the [OpenTelemetry semantic conventions for GenAI](https://opentelemetry.io/docs/specs/semconv/gen-ai/), the `gen_ai.response.finish_reasons` and per-token attribute slots are the right home for this; the spec is still under active development, so expect to define a custom attribute for `gen_ai.response.token_logprobs` until the standard catches up. The relevant span lives at the agent-message granularity, not the orchestration root.

**Add an alert on mean token log-probability per agent, per round.** Slow Drift in particular manifests as a small but persistent dip in mean log-probability for the malicious agent compared to its peers. A simple rolling z-score on `mean_logprob_per_message`, grouped by agent ID and conversation, catches a meaningful fraction of the documented attacks without any model retraining. Threshold-tuning here is similar to the work you already do for drift detectors — start permissive, instrument the false positive rate, tighten over the first month.

**Track within-message log-probability variance.** Benign Wrapper attacks produce a confidence cliff between the wrapper text and the payload. A per-message metric like "max contiguous-window log-probability gradient" surfaces this even when mean confidence looks healthy. Bonus: this metric is also a useful smoke alarm for jailbreak attempts in single-agent settings, since most jailbreak suffixes show the same step change.

**Cap the number of communication rounds.** Zhang et al. note that their proposed defense's effectiveness degrades with rounds — a finding consistent with [MASpi's broader characterization of MAS prompt-injection propagation](https://openreview.net/forum?id=1khmNRuIf9), which observed that compromised agents tend to infect the rest of the network rapidly once given enough turns. If your application architecture allows it, hard-cap rounds at a lower number than your prompt engineering team would like. The marginal accuracy improvement past round three is usually small. The marginal attack surface is not.

**Treat agent-ID as a monitored dimension, not just a label.** Drift detection in ML observability tools like Arize, Fiddler, Evidently, and WhyLabs already supports per-segment monitoring. The agent ID — `planner`, `critic`, `tool_user_search`, whatever your role names are — is the natural segmentation dimension for MAS telemetry. Configure your dashboards so that you can see logprob distributions, refusal rates, and tool-call distributions broken out per agent role. When one role suddenly looks different from the others on a stable workload, treat it as a security alert, not just a quality regression.

**Map the defense to the NIST AI RMF's Measure function.** Most teams reading this are already being asked to produce risk-management artifacts. The [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework) organizes its guidance around Govern/Map/Measure/Manage, and the telemetry above slots into Measure cleanly. Specifically, the Generative AI Profile released in July 2024 calls out adversarial robustness as a measurement objective; an "agent confidence trajectory" metric on your platform is auditable evidence that you are doing it.

## Original analysis: why the proposed defense is necessary but not sufficient

Two things are worth flagging that the paper does not quite say outright.

First, the proposed confidence-based defense is itself a function of token-level statistics that the attacker can shape, just as the embedding-based defense was. The asymmetry that Zhang et al. lean on — embedding shapeable, logits not — holds only as long as the attacker's controlling LLM is similarly distributed to the defender's auditor. The moment the attacker is allowed to use a different model family (a different tokenizer, different temperature, different decoding strategy), the auditor's expected log-probability distribution stops being a clean baseline.

A pragmatic implication for platform teams: do not deploy a single auditor model checking confidence signals for the entire MAS. Use the same model that authored the message as the source of logprobs whenever possible (the so-called "self-reported confidence" channel), and reserve cross-model perplexity scoring for cases where you have a verified, identical decoding configuration on both ends. The attacker who controls one agent can lie about its own logprobs, but only if they control the inference endpoint, not just the prompt. For most architectures where every agent runs through the same hosted inference layer, self-reported confidence is harder to forge than the prompt was.

Second, the paper frames the problem as a binary detect-or-prune decision. Production MAS rarely operate that way. A practical defense looks more like a Bayesian weighting scheme: each message gets a trust score that decays with anomalous confidence signals, recovers with corroboration from other agents, and decays again on round-over-round drift. This is closer to how reputation systems work in classical distributed-systems literature than to how anomaly detectors are usually built. The right thing to copy from a decade of MLOps practice is the soft-label, calibrated-score approach to fraud and abuse scoring — not the hard-threshold abuse classifier that nobody trusts anymore.

For teams comparing tool options, the relevant capabilities are: agent-segmented telemetry, log-probability storage at the message granularity, configurable rolling-window anomaly detectors, and an ability to back-pressure or quarantine an agent role at runtime based on the trust score. Most of the LLM-observability vendors are still catching up here. If you are evaluating tooling specifically for agent-system monitoring, the closest fit today is whichever vendor lets you push custom numeric attributes per span and aggregate them server-side, rather than whichever one ships the prettiest "agent trace" view.

The broader lesson is one this site has been writing about for a while: as soon as your inference graph stops being a single call, the failure modes you face start looking less like "the model was wrong" and more like "the distributed system was poisoned." That is not the same problem, and a prompt template will not solve it. The next generation of MAS defenses is going to look much more like classical SIEM and anomaly detection than like prompt engineering, and the teams that have an observability culture will adapt faster than the teams that don't.

For organizations whose threat model leans more toward adversarial inputs and red-teaming methodology than toward monitoring, sister-site [guardml.io](https://guardml.io) tracks the defensive-guardrails ecosystem in more depth and is worth a parallel read.

## Where this leaves the operator

The honest summary: embedding-based detection in multi-agent systems has been a comforting placebo. Zhang et al. did the field a favor by showing exactly how it fails and offering a concrete, low-cost mitigation that anyone running production inference can roll out without a model retrain. The work to do is mostly observability work: capture logprobs you are already computing, persist them, alert on them per agent role, and stop pretending that one embedding distance per message is enough signal to detect a coordinated peer.

If you take exactly one task away from this post: open whatever monitoring config governs your agent system, find the line where messages get logged, and check whether you are also logging the token confidence vector. If you are not, that is the next ticket.

## Sources

- [Zhang, Zheng, Chen — *When Embedding-Based Defenses Fail: Rethinking Safety in LLM-Based Multi-Agent Systems*, arXiv:2605.01133](https://arxiv.org/abs/2605.01133) — The seed paper. Provides the theorem, the three attacks (Slow Drift, Benign Wrapper, Chaos Seeding), and the confidence-augmented defense proposal.
- [MASpi: A Unified Environment for Evaluating Prompt Injection Robustness in LLM-Based Multi-Agent Systems (OpenReview)](https://openreview.net/forum?id=1khmNRuIf9) — Companion empirical work characterizing how prompt-injection compromise propagates across agent populations. Useful for sizing how many rounds you can tolerate before contamination spreads.
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework) — The Govern/Map/Measure/Manage taxonomy the operational takeaways above are mapped to, plus the Generative AI Profile that calls out adversarial robustness as an explicit measurement objective.
- [OpenTelemetry Semantic Conventions for Generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — The live (still under development) spec for instrumenting LLM and agent operations. The right starting point for adding logprob attributes to agent spans rather than rolling your own ad-hoc schema.
- [Prompt Injection Attacks in Large Language Models and AI Agent Systems (MDPI Information, 2026)](https://www.mdpi.com/2078-2489/17/1/54) — Broader review of injection-attack taxonomies and defenses. Useful for putting the multi-agent failure mode in the context of the wider attack surface.
