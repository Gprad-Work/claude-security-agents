# Example: GRCSecurity on ClariNote PRD

> Agent: `GRCSecurity` (Sonnet) · Input: [`../sample-system/PRD.md`](../sample-system/PRD.md)
> Illustrative domain output.

---

## GRC Security Review

### Applicable frameworks
- **HIPAA** — the product creates, receives, and stores PHI on behalf of clinics; ClariNote is a Business Associate.
- **GDPR** — EU clinics' patient data is EU personal data (and special-category health data under Art. 9).
- **SOC2** — B2B buyers (clinics) will require it; Security + Confidentiality TSCs apply.

### Critical findings (compliance blockers)
| # | Framework | Requirement | Gap | Remediation |
|---|---|---|---|---|
| G-001 | HIPAA §164.308(b) / §164.314 | BAA with subcontractors that handle PHI | The PRD's open question ("Do we need a BAA with every vendor, or only those that store PHI?") reveals BAAs are not in place. Anthropic, AWS, Twilio, and Sentry all receive PHI. A BA must have a BAA with every subcontractor that creates/receives/maintains/transmits PHI — "stores" is not the test. | Execute BAAs with Anthropic, AWS, Twilio, and any vendor receiving PHI (incl. Sentry if request context contains PHI) before production. Vendors without a BAA must not receive PHI. |
| G-002 | GDPR Art. 17 / HIPAA disposal | Right to erasure / secure disposal | Account deletion is soft delete (`deleted_at`) only (§7). Documents/summaries are retained indefinitely, and backups share the production KMS key with no separate deletion path. Erasure requests cannot be fulfilled. | Implement a hard-delete/anonymization path covering Postgres, S3, the vector store, and backups. Consider crypto-shredding via per-tenant keys so backups are covered. Define retention limits per data category. |

### High findings
| # | Framework | Requirement | Gap | Remediation |
|---|---|---|---|---|
| G-003 | GDPR Art. 44–49 | International transfers / residency | EU patient data has no residency or transfer mechanism (§9 open question). Single AWS account and US-based subprocessors (Segment, Twilio, Anthropic) imply EU→US transfers with no stated SCCs or regional isolation. | Decide EU data residency (regional storage), put SCCs/DPAs in place for each transfer, and document a Transfer Impact Assessment. |
| G-004 | HIPAA §164.312(b) | Audit controls over PHI access | No audit log of PHI access/disclosure (§6 — "no specific alerting on PHI access"). HIPAA requires recording access to ePHI. | Implement immutable audit logging of every PHI read/write/disclosure (actor, patient, action, time). Coordinate with SecOps. |
| G-005 | GDPR Art. 6/9; HIPAA minimum necessary | Lawful basis, minimization, secondary use | Summaries are sent to an LLM and embeddings retained for retrieval; Segment receives URL paths that may embed IDs. No stated lawful basis, minimization, or restriction on secondary use of PHI. | Document lawful basis and minimum-necessary scoping; confirm the LLM vendor does not train on or retain PHI (contractual); strip PHI/IDs from analytics. |

### Medium / Low findings
| # | Framework | Requirement | Gap | Remediation |
|---|---|---|---|---|
| G-006 | SOC2 CC7 / breach rule | Breach detection & notification | No IR/breach-notification procedure described; vendor breach-notification SLAs unaddressed. HIPAA requires notification within 60 days; GDPR within 72 hours of awareness. | Document an IR/breach plan and require breach-notification SLAs in vendor contracts (see TPRM). |
| G-007 | SOC2 CC8 | Change management | Developers set their own pod specs; no admission control or change gates (§4.1). | Document change management and enforce guardrails (ties to ContainerSecurity/DevSecOps). |

### What's done well
- Roles and a support-access concept are defined (§2), giving a basis for minimum-necessary access controls once enforced.

### Verdict
**NON-COMPLIANT** — As specified, ClariNote cannot lawfully process real PHI: BAAs are absent (G-001) and there is no erasure/disposal path (G-002), each independently blocking. HIPAA audit controls (G-004) and GDPR transfer mechanisms (G-003) must also be resolved before EU or production data. None require redesign — they require controls the PRD currently omits.
