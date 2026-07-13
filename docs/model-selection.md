# Model Selection

Why each agent is assigned to Opus or Sonnet.

---

## Decision framework

Three factors drive the choice:

1. **Reasoning type** — open-ended adversarial reasoning needs Opus; structured checklist application is fine on Sonnet
2. **Cost of error** — a wrong finding in SecurityLead or IRAnalyst propagates to a gate or containment decision; a missed low-level finding in a domain checklist agent is recoverable
3. **Output structure** — constrained schema outputs (severity / finding / impact / mitigation) benefit less from Opus's extra reasoning than free-form synthesis

**Cost context (per million tokens, input + output):**
- Opus 4: ~$15 / $75
- Sonnet 4: ~$3 / $15

---

## Security review agents

### SecurityLead — Opus

SecurityLead does not apply a checklist. It reads an arbitrary artifact, decides which domains are relevant (a judgment call requiring full threat model understanding), dispatches agents, then synthesises findings from multiple domains into a non-duplicated, risk-ranked report with a gate decision. Each step is judgment under uncertainty — Opus's extended reasoning materially reduces dispatch errors and synthesis gaps.

### AISecurity — Opus

Prompt injection is not a checklist vulnerability. The agent must reason about multi-step attack chains, identify indirect injection vectors not visible in the primary code path, and distinguish theoretical from practical risk for the specific architecture under review. Sonnet tends to produce generic warnings ("sanitize user input") rather than specific, contextual attack vectors. For AI security, generic is useless.

### RedTeam — Opus

RedTeam is not a checklist agent. Its entire value is open-ended adversarial reasoning: reading across every domain agent's output and the artifact itself to chain individually-minor weaknesses into end-to-end attack paths that no single-domain reviewer would surface. This is the same class of reasoning as AISecurity and SecurityLead — synthesis and adversarial imagination under uncertainty — and Sonnet tends to re-list single-domain findings rather than construct novel cross-domain chains. Opus earns its cost here.

### All other domain agents — Sonnet

ProductAppSecurity, GRCSecurity, SecOps, DevSecOps, CloudSecurity, InfraSecurity, NetworkSecurity, PlatformSecurity, MobileSecurity, ContainerSecurity, DataSecurity, TPRMSecurity, and ThreatIntel all apply structured domain knowledge against a specific artifact. The task is: read the spec, check each section against a known set of patterns, report findings with specificity. This is structured retrieval and application, not adversarial reasoning. Sonnet matches Opus quality on bounded checklist tasks at ~5× lower cost. (ThreatIntel does prioritization rather than open-ended attack synthesis — its actor-and-TTP mapping is structured against known adversary catalogs, so Sonnet fits.)

---

## Detection engineering agents

### DEReviewRule — Opus

`efficacy_check.py` gives a score. That score doesn't tell you whether a rule will be tuned out by analysts at 3am after generating 200 false positives in the first week. Opus provides the qualitative judgment that automated scoring cannot — detection logic correctness, evasion path analysis, operational context. A bad rule ships to production and wastes SOC time.

### DEDetectionRuleWriter — Sonnet

Writing a schema-compliant Sigma YAML rule is a structured generation task. The output schema is fixed, the field names are known, and the MITRE tags are enumerated. Sonnet produces equivalent quality to Opus on this task at a fraction of the cost.

### DECoverageScanner — Sonnet

ATT&CK gap analysis is systematic: run the script, read the output, map tactics to rules, prioritise by relevance to the stated stack. No adversarial reasoning required.

---

## Incident response agents

### IRAnalyst — Opus

Phase 3 issues a TP/FP verdict. A wrong call means either a real attack goes unresponded to, or an analyst team gets burned on a false escalation. Recognising a false positive across ambiguous, conflicting evidence — where some indicators look suspicious and others provide an innocent explanation — is Opus-level reasoning.

### IRAlertParser — Sonnet

Parsing an alert into structured entities (actor, IP, resource, timestamp, investigation timeframe) is a structured extraction task. The output schema is defined. Sonnet handles this reliably.

### IRSIEMInvestigator — Sonnet

SIEM investigation is systematic: query for events in the timeframe, pivot on extracted entities, test each FP condition. The pivot patterns are repeatable. Sonnet at equivalent quality to Opus for the same cost savings.

---

## Cost estimate

A typical security review run dispatches SecurityTriage + a handful of relevant domain agents + SecurityLead (the Lead selects only the domains with real surface — it does not run all 15 every time). A representative run of ~12 agents:

| Model | Agents | Approx. cost |
|---|---|---|
| Opus | SecurityLead + AISecurity (+ RedTeam when a chaining review is warranted) | ~$0.65 × 2–3 = $1.30–2.00 |
| Sonnet | ~9 domain agents + SecurityTriage | ~$0.13 × 10 = $1.30 |
| **Total** | | **~$1.60–3.00 per run** |

All-Opus would run ~$7–8 for the same review. The 4× saving compounds across multiple spec versions and review types. The full library is 15 domain agents, but the Lead's job is to dispatch only what adds signal — a run that fires all 15 usually means the triage was too broad, not that the system needed it.
