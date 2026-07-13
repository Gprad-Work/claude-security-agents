# Example: TPRMSecurity on ClariNote PRD

> Agent: `TPRMSecurity` (Sonnet) · Input: [`../sample-system/PRD.md`](../sample-system/PRD.md)
> Illustrative domain output — one of the newly added agents.

---

## Third-Party Risk Management Review

### Vendor inventory
| Vendor | Purpose | Data categories | Credentials held | Compromise blast radius |
|---|---|---|---|---|
| Anthropic (Claude API) | Summarization | Extracted document text (PHI) | Long-lived API key (**exposed in public image**) | Key abuse (LLMjacking) + PHI-in-transit exposure; key is already public (DS-001) |
| AWS | Hosting (RDS/S3/EKS) | All data | Account credentials/roles | Total — all PHI and compute |
| Pinecone | Vector store | Summary embeddings (PHI-derived) | API key | Cross-tenant PHI-derived data if unscoped (ties to AI-002) |
| Twilio | SMS reminders | Patient name + phone | API key | Patient contact data; SMS-spoofing/phishing of patients as ClariNote |
| Segment | Analytics | URL paths (may embed IDs) | Write key | Identifier/PHI leakage outside BA boundary (D-005) |
| Sentry | Error tracking | Stack traces + request context (PHI) | DSN/token | PHI exposure via error context; no BAA (G-001) |
| Google Workspace | SSO (optional) | OAuth identity **+ `drive.readonly`** | OAuth client + refresh tokens | Read of clinicians' entire Google Drive (P-003) |

### Critical findings
| # | Vendor | Finding | Failure scenario | Fix |
|---|---|---|---|---|
| TP-001 | Google Workspace | SSO requests `https://www.googleapis.com/auth/drive.readonly` (§5) — full read of each clinician's Google Drive, unrelated to login. ClariNote holds refresh tokens for this grant. | If ClariNote is compromised (and its identity model is weak — see Platform P-001/DevSecOps DS-001), the attacker uses stored refresh tokens to read the entire Drive of every SSO clinician: patient spreadsheets, referrals, anything in Drive. A single app breach becomes a multi-org document breach. | Drop the `drive.readonly` scope; request only `openid email profile`. If Drive access is ever needed, use `drive.file` and store refresh tokens in a KMS-backed store with rotation. |
| TP-002 | Anthropic | The Claude API key is long-lived and, per DevSecOps DS-001, currently sits in a public container image. Even after rotation, the model is "long-lived keys, no offboarding" for all vendors (§5). | The public key is abused for API spend and exposes the summarization data path; long-lived keys across all vendors mean no containment when any leaks. | Rotate immediately; move to runtime secret injection; adopt short-lived/rotatable credentials and per-vendor key ownership; monitor spend. |

### High findings
| # | Vendor | Finding | Failure scenario | Fix |
|---|---|---|---|---|
| TP-003 | All PHI-receiving vendors | No formal vendor onboarding/security review ("integrated as needed by engineers," §5) and no BAAs (§9 open question). Anthropic, AWS, Twilio, and Sentry all receive PHI. | A vendor is added with no assessment; PHI flows to a subprocessor with no BAA and unknown security posture — a HIPAA violation (GRC G-001) and an unmanaged breach channel. | Institute a lightweight vendor-onboarding gate proportional to data sensitivity; require a BAA before any vendor receives PHI; maintain the inventory above as living. |
| TP-004 | Sentry / Segment | These receive request context / URL paths that include PHI or identifiers (§5–6), but are typically treated as "non-data" tools and excluded from PHI governance. | PHI silently accumulates in error-tracking and analytics platforms outside the BA boundary, with their own access and retention. | Scrub PHI before it reaches Sentry/Segment (see D-001/D-005); bring them into the BAA/DPA scope or stop sending PHI. |
| TP-005 | All | Subprocessor/fourth-party risk is unaddressed. EU clinics' data flows to US vendors with their own subprocessors (residency implications, GRC G-003). | A subprocessor change or breach (e.g., a vendor's support tooling) exposes PHI with no notification path. | Track critical vendors' subprocessor lists and change notices; require breach-notification SLAs (24–72h) in contracts. |

### Medium / Low findings
| # | Vendor | Finding | Failure scenario | Fix |
|---|---|---|---|---|
| TP-006 | Concentration | AWS is a single point of failure for hosting, backups, and (via same account) staging (§7). | An AWS account compromise or major outage takes everything down at once. | Accept and document, but isolate accounts (see CloudSecurity C-005) and define continuity for the backup path. |
| TP-007 | Lifecycle | No offboarding process for vendor credentials/OAuth grants. | Orphaned keys and grants outlive their use — a common breach source. | Add an offboarding checklist: revoke keys, remove OAuth grants, confirm data deletion. |

### What's done well
- The vendor set is small and purpose-clear, and uses reputable providers with mature security programs — the gaps are governance (reviews, BAAs, scoping, lifecycle), not reckless vendor choices.

### Verdict
**HIGH RISK** — The sharpest third-party risk is the over-scoped Google Drive grant (TP-001): it turns a ClariNote breach into a read of every SSO clinician's entire Drive, wildly disproportionate to an SSO login. Combined with a public long-lived Anthropic key (TP-002) and no BAAs or onboarding review (TP-003), the vendor layer is currently ungoverned. Minimize the OAuth scope, rotate keys, and gate PHI-receiving vendors behind BAAs before production.
