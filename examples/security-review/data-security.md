# Example: DataSecurity on ClariNote PRD

> Agent: `DataSecurity` (Sonnet) · Input: [`../sample-system/PRD.md`](../sample-system/PRD.md)
> Illustrative domain output — one of the newly added agents.

---

## Data Security Review

### Sensitive data map
| Data category | Classification | Where it lives (all copies) | Protection | Leakage paths identified |
|---|---|---|---|---|
| Patient documents (PHI) | Restricted | S3 uploads bucket; OCR temp; worker memory | SSE at rest (assumed); path-based tenant separation only | Full request-body logs; Sentry context; staging dumps |
| Clinical summaries (PHI) | Restricted | Postgres `summaries`; Pinecone embeddings; backups | DB/disk encryption only; no field-level | Analytics URLs; logs; cross-tenant RAG (see AI-002) |
| Patient demographics + contact | Restricted | Postgres; Twilio (name+phone) | TLS in transit | Twilio; logs |
| Session JWTs / secrets | Restricted | Env vars; **image layer** (Anthropic key) | None at the image layer | Public ECR (see DevSecOps DS-001) |

### Critical findings
| # | Data category | Finding | Exfiltration / exposure scenario | Fix |
|---|---|---|---|---|
| D-001 | All PHI | On error, the **full request body is logged to CloudWatch**, and **Sentry captures request context** (§6). Uploaded content, demographics, and summaries land in log/error pipelines that far more people can read than the database, with weaker controls and long retention. | An engineer with CloudWatch/Sentry access — or anyone who compromises those pipelines — reads PHI that never should have left the datastore. This is also a HIPAA disclosure with no audit trail. | Redact request bodies and PHI fields before logging; configure Sentry `beforeSend` scrubbing and disable request-body capture; treat logs as a Restricted store with access controls and retention limits. |
| D-002 | Backups (all PHI) | RDS snapshots and the S3 backup bucket are in the **same account** and use the **same KMS key** as production (§7). Backups carry the full PHI corpus with none of the app's tenant checks. | One compromised admin credential or key policy reads or destroys production *and* backups together — the ransomware/insider worst case. Erasure requests also can't reach immutable backups (ties to GRC G-002). | Separate backup key + account; restrict key policy; enable Object Lock/versioning; adopt per-tenant crypto-shredding so erasure and breach-scoping are tractable. |

### High findings
| # | Data category | Finding | Exfiltration / exposure scenario | Fix |
|---|---|---|---|---|
| D-003 | All PHI | **Staging is refreshed weekly from a production DB dump** (§7). Real PHI now lives in a lower-trust environment (broader access, weaker monitoring, often wired to CI). | A staging compromise, an over-privileged developer, or a leaked staging credential exposes real patient data. | Mask or synthesize data for staging; never copy raw PHI into non-prod. If realistic data is needed, use irreversibly de-identified sets. |
| D-004 | Summaries (PHI) | No **field-level encryption**; protection is disk/TLS only. A compromised app, DB credential, or broad read role sees all PHI in plaintext. Derived embeddings in Pinecone inherit PHI sensitivity but aren't called out as protected. | A leaked read-replica credential or an app-level RCE dumps plaintext summaries; Pinecone vectors are treated as non-sensitive and under-protected. | Apply field-level/application-layer encryption to the highest-sensitivity fields; classify and protect embeddings as PHI; scope DB roles per service. |
| D-005 | Demographics/URLs | **Segment receives URL paths** (§5) which may embed `patient_id`/`summary_id`, and analytics/session tools are common silent leak channels. | Identifiers (and via them, PHI linkage) flow to an analytics vendor outside the PHI boundary. | Strip identifiers/PHI from analytics events and URLs sent to third parties; keep PHI inside the BA boundary. |

### Medium / Low findings
| # | Data category | Finding | Exfiltration / exposure scenario | Fix |
|---|---|---|---|---|
| D-006 | Egress | No egress/DLP monitoring on workers holding broad S3 access. | Bulk exfiltration is invisible (overlaps SecOps S-006). | Add egress anomaly detection / DLP on data-plane workloads. |
| D-007 | Lifecycle | Indefinite retention of documents/summaries (§7). | Ever-growing PHI store increases breach impact and violates minimization. | Define and enforce per-category retention with automated deletion reaching all copies (DB, S3, Pinecone, backups). |

### What's done well
- Encryption at rest via managed services (RDS/S3 KMS) is presumably on by default — a reasonable baseline. The gaps are field-level protection, key/backup isolation, and closing the log/analytics/staging leak channels.

### Verdict
**BLOCK** — Following the data, PHI leaks through channels weaker than the database itself: full-body logs and Sentry (D-001), a prod-cloned staging environment (D-003), and analytics URLs (D-005), while shared-key backups (D-002) put both confidentiality and recoverability at risk. Close the leak channels and isolate backups before production PHI flows.
