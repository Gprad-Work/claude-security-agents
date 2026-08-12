# Model Selection

Why each agent is assigned to Opus or Sonnet.

> **Source of truth:** the `model:` field in each agent's frontmatter under `.claude/agents/` is authoritative — it is what actually runs. This document explains those assignments and is kept in sync with them; if they ever disagree, the frontmatter wins and this doc is the bug. The current split is **7 agents on Opus** (`SecurityLead`, `SecurityTriage`, `AISecurity`, `RedTeam`, `DEReviewRule`, `IRAnalyst`, `VMExploitabilityAnalyst`) and everything else on Sonnet.

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

### SecurityTriage — Opus

SecurityTriage performs the same dispatch decision as the Lead's triage mode: read the artifacts, form a threat model, and select which specialists to run. A wrong selection here fails silently — an un-dispatched domain produces no findings, so a whole vulnerability class is simply never examined, and nothing in the pipeline flags the omission. That is the same "wrong dispatch = missed vulnerability class" cost that puts SecurityLead on Opus, so SecurityTriage is on Opus too. (It is the one place this library spends Opus on an agent that produces no findings itself — justified because its output determines whether the expensive-to-miss findings ever get generated.)

### AISecurity — Opus

Prompt injection is not a checklist vulnerability. The agent must reason about multi-step attack chains, identify indirect injection vectors not visible in the primary code path, and distinguish theoretical from practical risk for the specific architecture under review. Sonnet tends to produce generic warnings ("sanitize user input") rather than specific, contextual attack vectors. For AI security, generic is useless.

### RedTeam — Opus

RedTeam is not a checklist agent. Its entire value is open-ended adversarial reasoning: reading across every domain agent's output and the artifact itself to chain individually-minor weaknesses into end-to-end attack paths that no single-domain reviewer would surface. This is the same class of reasoning as AISecurity and SecurityLead — synthesis and adversarial imagination under uncertainty — and Sonnet tends to re-list single-domain findings rather than construct novel cross-domain chains. Opus earns its cost here.

### All other domain agents — Sonnet

ProductAppSecurity, GRCSecurity, SecOps, DevSecOps, CloudSecurity, InfraSecurity, NetworkSecurity, PlatformSecurity, MobileSecurity, ContainerSecurity, DataSecurity, TPRMSecurity, ThreatIntel, APISecurity, PrivacyEngineering, and FraudAbuse all apply structured domain knowledge against a specific artifact. The task is: read the spec, check each section against a known set of patterns, report findings with specificity. This is structured retrieval and application, not adversarial reasoning. Sonnet matches Opus quality on bounded checklist tasks at ~5× lower cost. (ThreatIntel does prioritization rather than open-ended attack synthesis; APISecurity and PrivacyEngineering apply well-defined catalogs — the OWASP API Top 10 and privacy-by-design principles; and FraudAbuse maps intended functionality to enumerated abuse patterns. All are structured application, so Sonnet fits. FraudAbuse is a borderline call — abuse modeling has an adversarial streak — but its patterns are catalogued enough that Sonnet holds; promote it to Opus if you find it producing generic advice.)

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

## Vulnerability management agents

### VMExploitabilityAnalyst — Opus

This is the pipeline's decisive judgment call, and the exact analogue of IRAnalyst. Reachability tells you the vulnerable code *can* be called; exploitability requires weighing conflicting evidence across data flow (does attacker input reach the sink in exploitable form?), configuration (are the exploit's preconditions present?), and exposure (can an attacker reach the entry point?) to decide whether the flaw is *real here*. A wrong "not exploitable" ships a live vulnerability; a wrong "exploitable" burns engineering time and erodes trust in the whole triage. That is judgment under uncertainty with an asymmetric, costly error — Opus.

### VMFindingParser — Sonnet

Normalizing scanner output (Snyk/Trivy/Dependabot/OSV/…) into a structured Finding Context is bounded extraction against known formats. The schema is fixed. Sonnet handles it reliably.

### VMReachabilityAnalyst — Sonnet

Tracing imports and call paths to the vulnerable sink is systematic graph-walking — the same "methodical, repeatable pivoting" that puts IRSIEMInvestigator on Sonnet. It is deliberately scoped to *reachability only* (it defers the hard judgment to Phase 3) and flags anything it can't resolve as INDETERMINATE rather than guessing. For heavily dynamic, reflective, or DI-driven codebases where call paths are genuinely hard to establish statically, promote it to Opus — but Sonnet is the right default.

### VMRemediationPlanner — Sonnet

Ranking findings against a defined priority model and producing fix plans with breaking-change assessment is structured application, not open-ended reasoning — the exploitability judgment has already been made in Phase 3. Sonnet fits.

---

## Cost estimate

A typical security review run dispatches SecurityTriage + a handful of relevant domain agents + SecurityLead (the Lead selects only the domains with real surface — it does not run all 18 every time). A representative run of ~12 agents:

| Model | Agents | Approx. cost |
|---|---|---|
| Opus | SecurityTriage + SecurityLead + AISecurity (+ RedTeam when a chaining review is warranted) | ~$0.65 × 3–4 = $1.95–2.60 |
| Sonnet | ~8 domain agents | ~$0.13 × 8 = $1.05 |
| **Total** | | **~$3.00–3.65 per run** |

All-Opus would run ~$7–8 for the same review. The saving still compounds across multiple spec versions and review types. The full library is 18 domain agents, but the Lead's job is to dispatch only what adds signal — a run that fires all 18 usually means the triage was too broad, not that the system needed it.
