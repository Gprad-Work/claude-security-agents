# Example: CloudSecurity on ClariNote PRD

> Agent: `CloudSecurity` (Sonnet) · Input: [`../sample-system/PRD.md`](../sample-system/PRD.md)
> Illustrative domain output.

---

## Cloud Security Review

### Critical findings
| # | Service | Finding | Scenario | Fix |
|---|---|---|---|---|
| C-001 | IAM (workers) | Workers "run with elevated permissions to reach S3 and the vector store" (§4), sharing the API image (§4.1). "Elevated" with no scoping usually means wildcard S3 access across the whole `clarinote-uploads` bucket. | A worker compromise (reachable via prompt-injection → RCE per RedTeam) yields read of every clinic's documents in S3 — cross-tenant PHI at the storage layer, beneath the app's `WHERE clinic_id` checks. | Scope the worker role to least privilege: prefix-conditioned S3 access, no `s3:*`. Separate the worker role from the API role. Use per-request scoped credentials where possible. |

### High findings
| # | Service | Finding | Scenario | Fix |
|---|---|---|---|---|
| C-002 | S3 (uploads bucket) | Tenant isolation for documents rests on the key *path* `{clinic_id}/{patient_id}/{doc_id}` (§3.1). Path conventions are not access control; nothing described enforces that Clinic A's credentials can't read Clinic B's prefix. | Any credential or code path that omits the prefix condition lists/read the whole bucket across tenants. | Enforce prefix isolation with IAM conditions (`s3:prefix`), block public access at the account and bucket level, enable default SSE-KMS, and turn on S3 access logging / Object Lock for audit. |
| C-003 | KMS | Backups (RDS snapshots + S3 backup bucket) use the **same KMS key** as production, all in one account (§7). Key or account compromise decrypts prod and backups together. | A single compromised admin/credential can read or destroy production data and its backups — the classic ransomware blast radius. | Separate keys for prod vs. backup; cross-account or logically isolated backup with restrictive key policies; enable key-usage logging. |
| C-004 | CloudTrail | Enabled in the primary region only (§6). | Cross-region attacker activity is unlogged; incident reconstruction and threat detection are partial (ties to SecOps S-004). | Org-wide, all-region CloudTrail with log-file validation to a locked, access-logged bucket. |

### Medium / Low findings
| # | Service | Finding | Scenario | Fix |
|---|---|---|---|---|
| C-005 | Account structure | Prod, backup, and (implied) staging share one AWS account (§7). No blast-radius separation. | One account compromise reaches everything. | Separate accounts (or at least strong SCP/OU boundaries) for prod, backup, and non-prod. |
| C-006 | S3 backup bucket protections | Backup bucket in the same account with no described versioning/Object Lock/MFA-delete. | Ransomware or a malicious insider deletes backups alongside prod. | Enable versioning, Object Lock (compliance mode), and MFA-delete; restrict who can delete. |

### What's done well
- Managed services (RDS, S3, EKS) are used rather than self-run infrastructure, and encryption-at-rest is available by default — the gaps are in scoping and isolation, not platform choice.

### Verdict
**HIGH RISK** — The dominant cloud risk is blast radius: over-broad worker IAM (C-001) plus path-only S3 isolation (C-002) means a single worker or credential compromise likely exposes all tenants' documents, and shared-key backups (C-003) put recovery at risk too. Scope IAM and enforce storage-layer tenant isolation before production PHI.
