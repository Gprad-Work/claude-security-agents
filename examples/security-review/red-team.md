# Example: RedTeam on ClariNote PRD

> Agent: `RedTeam` (Opus) · Input: [`../sample-system/PRD.md`](../sample-system/PRD.md)
> Illustrative domain output — one of the newly added agents. Authorized, defensive review: the value is the *chains*, described at remediation altitude.

---

## Red Team Assessment (Authorized)

### Attacker objectives
1. **Bulk PHI exfiltration across all tenants** — the whole patient corpus is the extortion payload.
2. **Cluster/node control** — durable foothold, and the position from which backups can be reached and destroyed.
3. **Backup destruction** — precondition for ransomware leverage (recovery denial).
4. **Free LLM compute / API-key abuse** — low-effort monetization independent of the above.

### Attack surface map
| Entry point | Trust level | Reachable by |
|---|---|---|
| Login (email/password, no MFA) | Unauthenticated | Anyone on the internet |
| Public ECR image | Unauthenticated | Anyone (public-read) |
| Document upload → LLM worker | Authenticated (any clinic user, incl. front-desk) | Any tenant user |
| `/api/patients/{id}`, `/api/summaries/{id}` | Authenticated | Any tenant user |
| Google SSO refresh tokens (held by ClariNote) | Server-side secret | Anyone who pops ClariNote |

### Attack paths

**Path 1: Uploaded document → worker RCE → node → all-tenant PHI + backup destruction**
| Step | ATT&CK tactic | Weakness exploited (domain) | Yields | Detection that should fire |
|---|---|---|---|---|
| 1 | Initial Access | Malicious document uploaded by any tenant user; text flows unrestricted into the prompt (AI-001) | Attacker-controlled content in the worker's execution context | None today |
| 2 | Execution | Worker parsing/OCR/tooling exploited from injected content; worker runs as root (Container CN-002) | Code execution in a root worker | None (no egress/anomaly detection, SecOps S-006) |
| 3 | Priv Esc / Escape | Worker/API mounts node Docker socket (CN-001) | Launch privileged container → node root | None |
| 4 | Credential Access | Node/pod holds broad S3 IAM (Cloud C-001) + shared JWT secret (Platform P-001) | Read all tenants' S3 documents; mint admin JWTs for any clinic | None |
| 5 | Impact | Backups share account+key, no Object Lock (Cloud C-003 / Data D-002) | Exfiltrate corpus; delete/encrypt backups | None |
- Attacker capability: skilled (needs a worker-side exploit) · Likelihood: **Medium** · Blast radius: **total** (all PHI + recovery denial)
- Chain-breakers: **remove the Docker-socket mount (step 3)** collapses escape; least-privilege worker IAM (step 4) contains it; immutable backups (step 5) preserve recovery. The socket mount is the highest-leverage single fix.

**Path 2: Credential stuffing → tenant-hop → full patient corpus (no exploit needed)**
| Step | ATT&CK tactic | Weakness exploited (domain) | Yields | Detection that should fire |
|---|---|---|---|---|
| 1 | Initial Access | No MFA, no rate limit/lockout (AppSec A-003); reused clinician password | One valid tenant session | None (no auth-anomaly alerting, S-001) |
| 2 | Discovery | Sequential integer patient IDs (AppSec A-001) | Enumerable object space | None |
| 3 | Collection | Data-layer authz optional / path-only S3 isolation (A-001, Cloud C-002) | Read other clinics' patients and summaries | None (no PHI-access volume alerting, S-001) |
- Attacker capability: **opportunistic** (no exploit — just a login and a for-loop) · Likelihood: **High** · Blast radius: cross-tenant PHI read
- Chain-breakers: MFA (step 1) and per-object data-layer authz + UUIDs (steps 2–3). This path needs no sophistication and is the most likely to be used first.

**Path 3: Public image → API key + Drive grant → LLM abuse + clinician Drive breach**
| Step | ATT&CK tactic | Weakness exploited (domain) | Yields | Detection that should fire |
|---|---|---|---|---|
| 1 | Credential Access | Anthropic key baked into public ECR image (DevSecOps DS-001) | Working Claude API key | Depends on external secret-scanning |
| 2 | Impact / Lateral | Long-lived key + over-scoped Google `drive.readonly` refresh tokens if ClariNote is further compromised (TPRM TP-001) | LLM compute abuse; read of every SSO clinician's Drive | None |
- Attacker capability: **opportunistic** (registry scraping is automated) · Likelihood: **High** for key abuse · Blast radius: API spend + multi-org Drive exposure
- Chain-breakers: rotate + remove the key and make ECR private (step 1); drop the Drive scope (step 2).

### Assumptions challenged
| Assumption | How it fails | Consequence |
|---|---|---|
| "Tenant isolation via `WHERE clinic_id`" | One missed predicate, or storage-layer/RAG/backup access that bypasses the app entirely | Cross-tenant PHI (Paths 1–2) |
| "Preview containers need the Docker socket" | The request-serving container should never hold node-level power | Node escape (Path 1) |
| "JWT claims tell us the role" | Claims are client-presented; shared HS256 secret is forgeable | Admin impersonation across tenants |
| "Analytics/error tools aren't data stores" | They receive PHI via bodies/URLs | PHI outside the BA boundary, un-audited |

### Assume-breach analysis
Most likely single compromise: **one phished/stuffed clinician session (Path 2, no MFA)**. From there, sequential IDs and thin data-layer authz already reach cross-tenant PHI — no further exploit required. That a single low-privilege credential reaches other tenants' records is the headline: **blast radius is not contained at the identity or data layer.** Segmentation (per-object authz, non-sequential IDs, MFA) matters more here than hardening any single component.

### Priority fixes (attacker-cost view)
1. **Remove the Docker-socket mount + de-privilege/split the worker** — severs Path 1 at its escape step; converts app RCE from "cluster compromise" to "contained pod." [medium effort]
2. **Enforce MFA + per-object data-layer authz + UUID patient IDs** — severs Path 2 entirely (the highest-likelihood path) and blunts step 4 of Path 1. [medium]
3. **Rotate/remove the Anthropic key, make ECR private, drop the Drive scope** — severs Path 3 and removes forgeable-token fuel. [low]
4. **Immutable, isolated backups (Object Lock, separate key/account)** — preserves recovery even if Path 1 completes. [low–medium]
5. **PHI-access + egress anomaly detection** — turns every path above from silent to noisy. [medium]

### Verdict
**MULTIPLE VIABLE PATHS** — including one (Path 2) that needs no exploit at all: a single MFA-less clinician credential reaches other tenants' PHI through sequential IDs and thin authorization. The single most dangerous path is Path 1 (document → worker RCE → node → all-tenant PHI + backup destruction), and removing the Docker-socket mount is the one change that does the most to break it. The system's core weakness is uncontained blast radius: individually "medium" gaps in AppSec, Container, Cloud, and Platform compose into total compromise.
