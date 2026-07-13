# Example: ThreatIntel on ClariNote PRD

> Agent: `ThreatIntel` (Sonnet) · Input: [`../sample-system/PRD.md`](../sample-system/PRD.md)
> Illustrative domain output — one of the newly added agents. Actor/TTP references are generalized from documented adversary behavior against this system class, not attributions of a specific named campaign.

---

## Threat Intelligence Review

### Threat landscape summary
ClariNote sits in the single most-targeted sector for data-extortion and ransomware — healthcare — and holds bulk PHI across many clinic tenants, which is high-value on criminal markets and carries heavy breach liability. The design offers multiple entry points that map cleanly onto commodity adversary playbooks: an internet-facing login with no MFA (credential stuffing / infostealer session theft), an LLM fed untrusted document text (prompt-injection-driven abuse), and a publicly exposed API key (LLMjacking). The realistic adversaries are opportunistic ransomware affiliates and access brokers, plus authenticated-user tenant-hopping; a targeted actor is plausible but not required for serious impact.

### Actor relevance
| Actor class | Interest in this system | Most likely playbook | ATT&CK techniques |
|---|---|---|---|
| Ransomware/extortion affiliate | Bulk PHI to encrypt + leak for double extortion | Phish or credential-stuff a clinician (no MFA) → session → PHI access → find over-broad IAM/backups → exfiltrate + destroy backups | T1566 Phishing, T1110 Brute Force, T1078 Valid Accounts, T1530 Data from Cloud Storage, T1490 Inhibit System Recovery, T1486 Data Encrypted for Impact |
| Access broker / infostealer operator | Sell validated clinician sessions/credentials | Harvest session cookies/tokens from infected devices; resell working access | T1539 Steal Web Session Cookie, T1078 Valid Accounts |
| Opportunistic API abuser | Free LLM compute | Scrape the public ECR image, extract the Anthropic key, resell/abuse it | T1552 Unsecured Credentials, T1078.004 Cloud Accounts |
| Malicious/curious tenant user | Other clinics' PHI | Enumerate sequential patient IDs; exploit thin authz | T1078 Valid Accounts, (app-layer IDOR) |

### Priority findings (intel-driven)
| # | Finding | Threat basis | Why it's prioritized | Recommendation |
|---|---|---|---|---|
| TI-001 | Public Anthropic API key in a public ECR image (DS-001) | "LLMjacking" — automated harvesting and resale of exposed LLM API keys is an active, documented pattern; public registries and repos are scanned continuously by criminals | The key is already exposed and internet-reachable; exploitation of exposed cloud keys is typically automated within hours | Rotate now, remove from image, private registry, spend alerting |
| TI-002 | No MFA + credential-stuffing/infostealer exposure (AppSec A-003) | Stolen/session-replayed credentials are the leading initial-access vector for SaaS breaches; healthcare is heavily targeted | Cheap, high-success initial access directly onto PHI | Enforce MFA (clinician/admin), detect impossible-travel and session anomalies |
| TI-003 | Backups share account+key with prod; no Object Lock (Cloud C-003/Data D-002) | Modern ransomware explicitly deletes/encrypts backups (T1490) before impact | Determines whether a ransomware hit is recoverable or existential | Immutable, isolated backups (Object Lock, separate key/account) |
| TI-004 | LLM fed untrusted document text (AI-001) | Prompt injection via ingested content is a documented, actively researched technique against LLM apps | Novel entry vector that most detection stacks don't cover | Prompt/output isolation + monitoring of anomalous model output |
| TI-005 | `node:18` base, unpinned deps, no scanning (DevSecOps DS-004) | Mass exploitation of disclosed CVEs in internet-facing software begins within days; unscanned images accumulate KEV exposure | Unknown/unmanaged vulnerability exposure on the public edge | SCA + image scanning gate; pin and rebuild base; track against KEV |

### Detection coverage handoff
| ATT&CK technique | Relevant to this system because | Detection exists? |
|---|---|---|
| T1078 Valid Accounts | Credential stuffing / tenant-hop onto PHI | No — no auth-anomaly alerting (SecOps S-001) |
| T1530 Data from Cloud Storage | Broad worker IAM over S3 | No — no egress/volume detection (S-006) |
| T1490 Inhibit System Recovery | Ransomware backup destruction | No — backups not immutable/monitored |
| T1552 Unsecured Credentials | Public key in image | Partial — depends on external secret-scanning |
| T1539 Steal Web Session Cookie | Infostealer session theft, no MFA | No |

→ Hand these to `DECoverageScanner` / `DEDetectionRuleWriter`. None currently have detection content.

### What's done well
- Human-in-the-loop clinical sign-off raises the cost of purely automated content-manipulation attacks, and the vendor set uses providers whose own threat monitoring is mature — a modest baseline to build on.

### Verdict
**HIGH THREAT EXPOSURE** — ClariNote's sector and data make it a default target for well-resourced extortion crews, and the design hands them the exact entry points their playbooks favor: no MFA, an exposed API key, and destroyable backups. The single most likely real-world compromise is credential/session theft of an MFA-less clinician leading to bulk PHI exfiltration and backup destruction. Enforcing MFA and making backups immutable are the two changes that most raise attacker cost.
