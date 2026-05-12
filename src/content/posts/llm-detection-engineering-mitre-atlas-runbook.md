---
title: "Detection Engineering for LLM Apps: A MITRE ATLAS-Mapped Runbook for Prompt Injection Alerting"
description: "Mapping LLM application telemetry to MITRE ATLAS techniques. Concrete log shapes, alerting heuristics, and a runbook structure that scales beyond ad-hoc grep rules."
pubDate: 2026-05-07
author: "Priya Anand"
tags: ["detection-engineering", "blue-team", "mitre-atlas", "llm-security", "siem", "incident-response"]
category: "defense"
sources:
  - title: "MITRE ATLAS — Adversarial Threat Landscape for AI Systems"
    url: "https://atlas.mitre.org/"
  - title: "OWASP Top 10 for LLM Applications"
    url: "https://owasp.org/www-project-top-10-for-large-language-model-applications/"
  - title: "MITRE ATLAS Mitigations Catalog"
    url: "https://atlas.mitre.org/mitigations/"
schema:
  type: "TechArticle"
heroImage: https://aisec-imagegen.th3gptoperator.workers.dev/featured/sentryml.com/llm-detection-engineering-mitre-atlas-runbook.png
heroAlt: "MITRE ATLAS LLM detection runbook visualization"
---

The detection engineering practice for traditional infrastructure is mature: events flow into a SIEM, rules map to MITRE ATT&CK techniques, alerts route through a tiered SOC, and runbooks specify investigation paths. The equivalent for LLM applications is mostly absent. Most teams I review are running on per-prompt regex rules in a homegrown YAML file, and calling it "AI security."

Here's how to do it as actual detection engineering, mapped to [MITRE ATLAS](https://atlas.mitre.org/) — the AI/ML equivalent of ATT&CK.

## What you log

Detection without a signal stream is impossible. Before any rules, instrument the LLM application to emit:

```
{
  "timestamp": "2026-05-07T14:23:01Z",
  "request_id": "uuid",
  "user_id": "user-1234",
  "session_id": "session-uuid",
  "feature": "support-bot",
  "model": "claude-sonnet-4-6",
  "input_text": "...",  // raw user input
  "input_source": "direct" | "rag-retrieval" | "tool-output" | "image-ocr",
  "system_prompt_hash": "sha256:...",
  "tools_invoked": ["search_docs", "send_email"],
  "tool_args": {...},
  "output_text": "...",
  "output_classified_pii": ["email", "name"] | [],
  "output_classified_secrets": ["api-key"] | [],
  "guardrail_decisions": [{"check": "injection", "verdict": "pass", "score": 0.12}],
  "latency_ms": 1240,
  "tokens_in": 412,
  "tokens_out": 188
}
```

The `input_source` field is critical. Most detection rules assume the user typed the input. In a RAG application or browsing agent, that's wrong — and the source of most missed indirect injection attacks.

## ATLAS-mapped detection rules

Each rule cites the ATLAS technique it covers. A real runbook has 30-50 rules; this is a representative subset.

### AML.T0051 — LLM Prompt Injection (direct)

**Telemetry**: `input_text` from `input_source: direct`
**Heuristic**: regex/embedding-similarity match against a corpus of known injection patterns. Threshold tuned to 5% false-positive rate on 7 days of production traffic.
**Severity**: medium (most direct injections fail; alert volume is high)
**Runbook**: log only by default; escalate to alert when paired with a tool invocation in the same session.

### AML.T0051.001 — LLM Prompt Injection (indirect)

**Telemetry**: `input_text` from `input_source: rag-retrieval | tool-output | image-ocr`
**Heuristic**: similarity match against injection corpus, plus structural anomaly (unusual instruction-imperative density inside retrieved content).
**Severity**: high (indirect injection is the highest-impact class; defenders rarely catch it)
**Runbook**: alert immediately. Investigate the retrieval source — was it user-controlled? Was it recently modified? If a public document, file with the source's abuse contact.

### AML.T0048 — External Harms (data exfiltration via tool-call abuse)

**Telemetry**: `tools_invoked`, `tool_args`
**Heuristic**: detect tool calls whose arguments contain content from prior turns that should not have left the session. Specifically: PII or `system_prompt_hash`-derivable content appearing as a `send_email` body or `web_request` URL parameter.
**Severity**: critical
**Runbook**: block the tool call, terminate the session, page on-call.

### AML.T0040 — ML Model Inference API (sensitive prompt extraction)

**Telemetry**: `output_text` plus `system_prompt_hash`
**Heuristic**: output substring overlap with the system prompt. Use rolling-hash comparison; a 50+ character contiguous match is suspicious.
**Severity**: medium-high
**Runbook**: block the response. Investigate whether the leak is reproducible and whether the system prompt should be considered compromised (yes, if it leaked once, assume the attacker can do it again).

### AML.T0024 — Exfiltrate ML Model (extraction via repeated inference)

**Telemetry**: per-user request rate, embedding similarity of consecutive prompts
**Heuristic**: a single user submitting >1000 highly-similar but slightly-varied prompts in 24h is fingerprint-extracting or scraping training data.
**Severity**: medium
**Runbook**: rate-limit the user. If they have a paid plan, enforce per-day caps. If anonymous, ban.

### AML.T0034 — Cost Harvesting

**Telemetry**: per-user `tokens_in + tokens_out` and dollar attribution
**Heuristic**: a single user driving a 10x daily-cost spike with no business reason. Often a stolen API key or an automation bug.
**Severity**: medium
**Runbook**: throttle, investigate, attribute (was the user themselves attacked, or is this internal misuse?).

### AML.T0067 — LLM Trusted Output Components Manipulation

**Telemetry**: `output_text` rendered in a downstream UI that interprets HTML/Markdown/links
**Heuristic**: output contains markdown link syntax with non-HTTPS URLs, JavaScript URIs, or external-domain redirects.
**Severity**: high (cross-application XSS is in scope here)
**Runbook**: sanitize the output before render; investigate why the model produced the malicious output (was it injected upstream?).

## Runbook structure

Each rule's runbook follows the same shape:

```
1. Verify the alert (raw event link, related session events, model card)
2. Triage: is this in-scope? (false positive corpus, baseline check)
3. Containment: what to do RIGHT NOW (block user, kill session, rate-limit)
4. Investigation: what to determine next (lateral movement, blast radius, root cause)
5. Recovery: how to restore service safely
6. Reporting: incident classification, customer comms, regulatory if applicable
7. Post-incident: rule tuning, training data updates, system prompt rotation
```

If you can't fill in 1-3 in under 5 minutes for a given rule, the rule isn't ready for production.

## Cross-mapping to OWASP LLM Top 10

ATLAS is structured around adversary techniques; [OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/) is structured around vulnerabilities. They're complementary lenses:

| ATLAS technique | OWASP LLM | Detection focus |
|---|---|---|
| AML.T0051 / T0051.001 | LLM01 Prompt Injection | input + retrieval inspection |
| AML.T0067 | LLM02 Insecure Output Handling | output sanitization |
| AML.T0040 | LLM07 System Prompt Leakage | output-vs-system-prompt comparison |
| AML.T0048 | LLM08 Excessive Agency | tool-call argument validation |
| AML.T0034 | (no direct equivalent) | per-user cost anomalies |

Mapping both frameworks at the rule level is overhead the team can absorb if it's done at design time. Retrofitting is painful. Do it early.

## What to skip

Things I see teams over-invest in that don't move detection efficacy:

- **Vendor "AI security platforms"** that sit between your app and the model API. They add latency, opaque rules, and a single point of failure. The signals you need are inside your application; the vendor sees a subset.
- **LLM-judged detection** ("ask GPT if this is injection"). Adversarial inputs that bypass the production model also bypass the judge model. Use deterministic rules first; reserve LLM judgment for ambiguous tier-2 triage.
- **Real-time blocking on low-confidence rules**. False positives erode user trust faster than missed attacks erode security posture, in apps with low attacker prevalence. Log and review before enforcing.

The detection-engineering principle that transfers cleanly from ATT&CK to ATLAS: investment goes into reducing time-to-detect for the high-impact, low-frequency events. Prompt injection is high-frequency and mostly low-impact; tool-call abuse is rare but catastrophic. Allocate accordingly.


---

*→ This post is part of the [ML Observability Hub](/posts/ml-observability-hub) — the complete index of [ML monitoring](https://mlmonitoring.report/) and MLOps resources on SentryML.*

For more context, [LLM operations guide](https://llmops.report/) covers related topics in depth.
