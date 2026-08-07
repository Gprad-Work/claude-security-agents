# /security-review

Multi-agent security review of any spec, PRD, ERD, or codebase. Runs specialist domain agents in parallel, synthesises findings, and issues a gate decision.

---

## Usage

```
/security-review <path-to-artifact>
/security-review Pesti/docs/PRD.md
/security-review src/api/
/security-review docs/PRD.md docs/ERD.md
```

`$ARGUMENTS` can be a file path, a directory, or multiple files separated by spaces.

---

## How it works

The command runs four phases:

**Phase 1 — Triage**
A `SecurityTriage` agent reads all artifacts, builds a threat model, and decides which domain agents are relevant. It does not produce findings — it produces a dispatch list with targeted questions for each domain.

**Phase 2 — Parallel domain review**
All selected domain agents run simultaneously. Each receives the threat model and its targeted questions. Running in parallel means a 10-agent review takes roughly the same wall time as one.

**Phase 3 — Collection**
All domain outputs are collected and organised by domain.

**Phase 4 — Synthesis**
A `SecurityLead` agent reads all domain reports and produces the final unified report: risk-ranked findings, de-duplicated, with a gate decision and acceptance criteria.

---

## Domain agents

The triage agent selects from these domains based on what's present in the artifact:

| Agent | Domain |
|---|---|
| `AISecurity` | Prompt injection, agentic attack chains, OWASP LLM Top 10 |
| `ProductAppSecurity` | OWASP Top 10, IDOR, injection, session management |
| `GRCSecurity` | GDPR, SOC2, HIPAA, CCPA, data retention |
| `SecOps` | Logging, alerting, IR readiness |
| `DevSecOps` | CI/CD, secrets, dependency security |
| `CloudSecurity` | IAM, storage ACLs, managed service posture |
| `InfraSecurity` | TLS, server hardening, secrets at rest |
| `NetworkSecurity` | VPC, firewall rules, egress |
| `PlatformSecurity` | OAuth/OIDC, Kubernetes RBAC, service mesh |
| `MobileSecurity` | iOS/Android, certificate pinning, deep links |
| `ContainerSecurity` | Dockerfile/image hygiene, registry, runtime hardening, Pod Security Standards |
| `DataSecurity` | Data classification, field-level encryption, exfiltration paths, backups |
| `TPRMSecurity` | Vendor compromise blast radius, OAuth scopes, subprocessor and concentration risk |
| `ThreatIntel` | Adversary TTP mapping, ATT&CK/KEV exposure, phishing and impersonation surface |
| `RedTeam` | Cross-domain attack-path chaining, assume-breach and blast-radius analysis (authorized) |
| `APISecurity` | OWASP API Top 10, BOLA/BFLA, GraphQL, resource consumption, SSRF, endpoint inventory |
| `PrivacyEngineering` | Privacy-by-design mechanism — minimization, consent, erasure, de-identification, tracking |
| `FraudAbuse` | Trust & Safety — fake accounts, bot/scraping, payment/promo fraud, resource/cost abuse |

A typical spec review dispatches 4–6 agents. A code review may dispatch all 18.

---

## Output

The final report is saved to a `security-reviews/YYYY-MM-DD-run/` directory adjacent to the reviewed artifacts.

Report structure:
- Executive summary (2–3 sentences)
- Gate decision: `GO` or `NO GO`, with acceptance criteria
- Risk-ranked finding table (Critical → High → Medium → Low)
- Full finding details: description, affected component, evidence, remediation
- Domain coverage map (which agents ran, which were skipped and why)

---

## Example gate decision

```
GATE: NO GO

Before this spec can proceed to implementation, the following must be resolved:

CRITICAL
- [SEC-001] Webhook endpoint has no signature verification — any caller can trigger
  reminders. Fix: verify HMAC-SHA256 of the raw request body before processing.
- [SEC-002] pg_cron job contains the CRON_SECRET as a literal string in the SQL body.
  The secret is stored in plaintext in pg_catalog. Fix: pass via session variable.

HIGH
- [SEC-003] Reminder fire endpoint has no authentication — any caller with the URL
  can trigger all pending reminders. Fix: verify CRON_SECRET header before processing.

Acceptance criteria: All CRITICAL and HIGH findings resolved and re-reviewed.
```

---

## Cost

A typical 5-domain spec review (SecurityTriage + 5 domain agents + SecurityLead):
- ~$1.50–2.00 per run (2 Opus agents — SecurityTriage + SecurityLead — plus 5 Sonnet domain agents; more if AISecurity or RedTeam is among the 5)

See [`docs/model-selection.md`](model-selection.md) for the full breakdown.
