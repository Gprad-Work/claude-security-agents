# Examples

Live, worked examples for every agent in this library. Each one runs against a **single shared sample system** — a fictional B2B healthcare SaaS called **ClariNote** — so you can see how the different domains view the same artifact and where their findings overlap and chain.

> These outputs are illustrative — hand-written to demonstrate each agent's format, depth, and voice on a realistic artifact. They are what a good run *looks like*, not a captured transcript. The sample system is fictional; any resemblance to a real product is coincidental.

---

## The sample system

- [`sample-system/PRD.md`](sample-system/PRD.md) — the product spec every example reviews. It intentionally contains a spread of realistic security gaps so each domain agent has something genuine to find.

ClariNote lets clinics upload patient documents; an LLM summarizes them into structured clinical notes. It has a clinician web app and a mobile app, runs on EKS, stores PHI in Postgres + S3, and integrates several third-party vendors. That surface exercises all fifteen security-review domains plus the detection-engineering and incident-response pipelines.

---

## Security-review pipeline

Run end-to-end with `/security-review examples/sample-system/PRD.md`.

| Example | Agent | What it shows |
|---|---|---|
| [`security-review/00-triage.md`](security-review/00-triage.md) | `SecurityTriage` | The dispatch decision — which domains get called and why |
| [`security-review/ai-security.md`](security-review/ai-security.md) | `AISecurity` | Prompt injection via uploaded documents, RAG cross-tenant leakage |
| [`security-review/product-app-security.md`](security-review/product-app-security.md) | `ProductAppSecurity` | IDOR on patient records, mass assignment, session gaps |
| [`security-review/grc-security.md`](security-review/grc-security.md) | `GRCSecurity` | HIPAA/GDPR gaps, BAA coverage, retention |
| [`security-review/secops.md`](security-review/secops.md) | `SecOps` | Logging and alerting coverage for PHI access |
| [`security-review/devsecops.md`](security-review/devsecops.md) | `DevSecOps` | CI/CD secrets, unpinned actions, dependency posture |
| [`security-review/cloud-security.md`](security-review/cloud-security.md) | `CloudSecurity` | S3 ACLs, IAM wildcards, CloudTrail gaps |
| [`security-review/platform-security.md`](security-review/platform-security.md) | `PlatformSecurity` | OAuth scopes, SSO, Kubernetes RBAC |
| [`security-review/mobile-security.md`](security-review/mobile-security.md) | `MobileSecurity` | Local PHI storage, certificate pinning, deep links |
| [`security-review/container-security.md`](security-review/container-security.md) | `ContainerSecurity` | Dockerfile hygiene, runtime hardening, escape surface |
| [`security-review/data-security.md`](security-review/data-security.md) | `DataSecurity` | Field-level encryption, PHI in logs, backup blast radius |
| [`security-review/tprm-security.md`](security-review/tprm-security.md) | `TPRMSecurity` | Vendor compromise blast radius, OAuth grants, subprocessors |
| [`security-review/threat-intel.md`](security-review/threat-intel.md) | `ThreatIntel` | Healthcare ransomware TTPs, KEV exposure, ATT&CK mapping |
| [`security-review/red-team.md`](security-review/red-team.md) | `RedTeam` | Chains the above into end-to-end attack paths |
| [`security-review/99-lead-synthesis.md`](security-review/99-lead-synthesis.md) | `SecurityLead` | The unified, de-duplicated, risk-ranked final report + gate decision |

`InfraSecurity` and `NetworkSecurity` are **not** dispatched for this artifact — the PRD is too thin on TLS/host and VPC/firewall detail to give them signal. The triage example shows the skip-with-reason decision; that restraint is the intended behavior, not a gap.

---

## Detection-engineering pipeline

| Example | Agent / command | What it shows |
|---|---|---|
| [`detection-engineering/rule-write.md`](detection-engineering/rule-write.md) | `/rule-write` → `DEDetectionRuleWriter` | A production-ready Sigma rule authored from a threat scenario |
| [`detection-engineering/rule-review.md`](detection-engineering/rule-review.md) | `/rule-review` → `DEReviewRule` | A scored peer review of that rule |
| [`detection-engineering/coverage-scan.md`](detection-engineering/coverage-scan.md) | `/coverage-scan` → `DECoverageScanner` | An ATT&CK coverage gap brief for the ClariNote stack |

---

## Incident-response pipeline

| Example | Agent / command | What it shows |
|---|---|---|
| [`incident-response/triage.md`](incident-response/triage.md) | `/triage` → `IRAlertParser` → `IRSIEMInvestigator` → `IRAnalyst` | A full alert-to-verdict triage on a ClariNote alert |

---

## Regenerating these

These examples were written against the PRD as it stands. If you change `sample-system/PRD.md`, re-run the relevant command to regenerate the affected outputs — e.g. `/security-review examples/sample-system/PRD.md`.
