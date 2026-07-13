# Example: /coverage-scan → DECoverageScanner

> Command: `/coverage-scan`
> Agent: `DECoverageScanner` (Sonnet)
> Maps the current rule library against MITRE ATT&CK, prioritized by relevance to the ClariNote stack (healthcare SaaS, EKS, Node, Claude API, multi-tenant PHI). Illustrative — assumes a small starter library where only `DATA-001` (from [`rule-write.md`](rule-write.md)) plus a couple of auth rules exist.

---

## Coverage Scan — 2026-07-13

### Snapshot
| Metric | Value |
|---|---|
| Rules in library | 3 |
| ATT&CK tactics with ≥1 rule | 3 / 11 relevant |
| Estimated coverage of stack-relevant techniques | ~20% |
| Highest-risk uncovered tactic | Impact (ransomware / backup destruction) |

### Tactic coverage (stack-relevant tactics only)
| Tactic | Rules | Covered techniques | Priority gaps for this stack |
|---|---|---|---|
| Initial Access | 1 | T1110 (auth brute force) | T1566 phishing/consent-phishing of clinicians |
| Credential Access | 2 | T1078, T1110 | **T1552 unsecured credentials (exposed API keys)**, T1539 session cookie theft |
| Collection | 1 | T1530 (DATA-001) | T1213 information-repository enumeration (cross-tenant variant) |
| Discovery | 0 | — | T1087 account/tenant discovery |
| Defense Evasion | 0 | — | T1562 impair defenses (audit-log tampering) |
| Exfiltration | 0 | — | **T1041 / T1567 bulk egress from workers** |
| Impact | 0 | — | **T1490 inhibit recovery (backup deletion)**, T1486 data-encrypted-for-impact |
| Persistence | 0 | — | T1136 create account, T1098 add credentials |

### HIGH priority recommendations
Ordered by threat relevance to *this* stack (cross-referenced with the ThreatIntel handoff in [`../security-review/threat-intel.md`](../security-review/threat-intel.md)).

1. **`SECRETS-001` — Exposed API key used from an unexpected source (T1552).**
   Directly covers the LLMjacking / public-key exposure (review SEC-002). Detect Anthropic/AWS key usage from an IP or ASN outside the known egress ranges.
   → `/rule-write "Anthropic or AWS API key used from an IP outside known ClariNote egress ranges"`

2. **`DATA-002` — Cross-tenant record access (T1213).**
   The critical variant flagged in the DATA-001 review: a single actor touching ≥2 distinct `clinic_id`s. Highest-fidelity signal for the IDOR (SEC-001).
   → `/rule-write "single user accessing patient records across more than one clinic_id"`

3. **`CLOUD-001` — Backup deletion or Object Lock disablement (T1490).**
   Covers the ransomware end-state (review SEC-007). Alert on `DeleteBackup`, `DeleteDBSnapshot`, `PutBucketVersioning=Suspended`, or KMS key-deletion scheduling from any non-break-glass principal.
   → `/rule-write "RDS snapshot or S3 backup deletion, or Object Lock/versioning disabled, by a non-break-glass principal"`

4. **`DATA-003` — Anomalous bulk egress from a worker pod (T1041/T1567).**
   Covers the exfiltration step of RedTeam Path 1. Alert on outbound volume from worker workloads exceeding baseline to non-vendor destinations.
   → `/rule-write "worker pod outbound data volume to a non-allowlisted destination exceeds baseline"`

### MEDIUM priority
- `AUTH-00x` — impossible-travel / new-device sign-in (T1078.004, T1539) — pairs with MFA enforcement (SEC-012).
- `AI-001` — anomalous LLM output signalling prompt-injection success (T1059-adjacent) — covers AI-001 from the review.
- `DEFENSE-001` — audit-log gaps / CloudTrail stop (T1562) — protects the detection layer itself.

### Note
Coverage is thin because the library is new — that's expected. The four HIGH items map 1:1 onto the CRITICAL/HIGH findings from the ClariNote security review, so writing them closes the detection gap on the paths most likely to be attacked. 

**Write the highest-priority missing rules now?** (yes → chains into `/rule-write` for SECRETS-001, DATA-002, CLOUD-001, DATA-003 in order)
