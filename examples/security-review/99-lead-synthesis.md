# Example: SecurityLead synthesis on ClariNote PRD

> Agent: `SecurityLead` (Opus) · Phase 4 of `/security-review` · Inputs: the triage + all domain reports in this folder
> This is the unified final report the skill saves. Illustrative.

---

# Security Review: ClariNote — Clinical Document Summarization (PRD)

> **Reviewed by:** Security Lead (Opus) + AISecurity, ProductAppSecurity, GRCSecurity, SecOps, DevSecOps, CloudSecurity, PlatformSecurity, MobileSecurity, ContainerSecurity, DataSecurity, TPRMSecurity, ThreatIntel, RedTeam
> **Artifact type:** PRD (pre-implementation)
> **Date:** 2026-07-13
> **Review method:** SecurityLead triage → parallel domain agent dispatch → Lead synthesis

---

## Section 1 — Executive Brief
> Audience: Security Lead, Product Team, Engineering Manager

### Overall security posture
As specified, ClariNote cannot safely or lawfully process real PHI. The design has one dominant, systemic problem — **uncontained blast radius** — where individually moderate weaknesses across the app, container, cloud, and identity layers compose into total, cross-tenant compromise. Separately, there are two issues that are dangerous *today*, before a line of production code ships: a live Anthropic API key in a public container registry, and the absence of BAAs with vendors already slated to receive PHI.

### The story in one paragraph
ClariNote is a multi-tenant healthcare AI product whose crown jewels are bulk PHI across many clinics sharing one database and one S3 bucket. The realistic attacker is an opportunistic ransomware/extortion crew — healthcare's default adversary — and the design hands them their preferred entry points. The Red Team found a no-exploit-required path: one MFA-less clinician credential (credential stuffing) plus sequential patient IDs plus application-only tenant isolation reaches other clinics' PHI directly. A second, higher-sophistication path turns a malicious uploaded document into worker code execution, then — via a root container mounting the node's Docker socket, broad worker IAM, and a forgeable shared-secret JWT — into node control, all-tenant PHI, and destruction of backups that share prod's account and key. None of this requires a nation-state; it requires a for-loop and a reused password. A Security Manager should treat this as **NO GO for implementation** until the blast-radius fixes and the two live issues are resolved.

### Gate decision

| Question | Answer |
|---|---|
| Safe to proceed to next phase? | **NO — with conditions** |
| Blocking items (must fix before implementation proceeds) | SEC-001, SEC-002, SEC-003, SEC-004, SEC-005 |
| Pre-ship items (must fix before production) | SEC-006 through SEC-011 |
| Post-ship items (fix within 30 days) | SEC-012 through SEC-015 |

---

## Section 2 — Lead's Synthesis
> Audience: Security Lead, Engineering Manager

### Dominant themes

**Theme 1: Uncontained blast radius (the core problem).**
The system has no effective segmentation at any layer. Tenant isolation is application-only [SEC-001]; the worker runs as root with the node's Docker socket and broad S3 IAM [SEC-004, SEC-005]; JWTs use one shared HS256 secret trusted downstream [SEC-003]; backups share prod's account and key [SEC-007]. Each is "medium" alone; together they mean one credential or one exploit reaches everything. This is the theme that turns the RedTeam paths from findings into an existential risk.

**Theme 2: PHI leaks through the soft edges, not the database.**
Full request bodies go to CloudWatch, request context to Sentry, prod data to staging weekly, and URL paths to Segment [SEC-006, SEC-009]. The database may be well-protected; the PHI still escapes through logs, analytics, and non-prod — channels with weaker access control and no audit trail.

**Theme 3: Two things are already on fire.**
The Anthropic key in a public image [SEC-002] and missing BAAs [SEC-008] are not future risks — the key is exposed now, and processing PHI without BAAs is a present violation. These are decoupled from the redesign and should move today.

**Theme 4: The system can't see itself.**
No PHI-access alerting, no immutable audit trail, single-region CloudTrail, no egress monitoring [SEC-010]. Every RedTeam path is silent end-to-end. Even after the structural fixes, ClariNote would neither detect nor be able to reconstruct an incident.

### Cross-domain risks

| Risk | Domains | Finding IDs | Why it's worse than each finding alone |
|---|---|---|---|
| Document → worker RCE → node → all-tenant PHI + backup wipe | AI + Container + Cloud + Platform + Data | SEC-001,3,4,5,7 | No single agent sees the whole chain; the AI injection surface and the container/cloud blast radius are only catastrophic *together* |
| MFA-less login + sequential IDs + app-only authz | AppSec + SecOps + ThreatIntel | SEC-001, SEC-010 | The highest-likelihood path needs no exploit; absence of detection makes it silent |
| Public key + over-scoped Drive grant | DevSecOps + TPRM + Platform | SEC-002, SEC-011 | A registry scrape plus a disproportionate OAuth scope escalates "leaked LLM key" into "multi-org Drive breach" |

### Lead overrides and annotations

| Finding | Domain agent's rating | Lead's rating | Reasoning |
|---|---|---|---|
| Mobile local-storage/pinning (M-001/M-002) | Critical/High | **HIGH (pre-ship)** | Real, but PRD lacks mobile detail and it's not on the critical PHI-breach path; must be answered before the mobile build, not before backend implementation |
| Container runtime hardening baseline (CN-005) | Medium | **MEDIUM** | Agreed; important defense-in-depth but subordinate to removing the Docker socket (SEC-004) |
| LLM cost/DoS (AI-005) | Medium | **LOW** | Genuine but low-impact next to PHI exposure; monitor post-ship |

### Coverage gaps
- **InfraSecurity/NetworkSecurity were not dispatched** (PRD too thin on TLS/host and VPC/firewall). Re-review at the infra-design stage — the flat single-namespace network is flagged via Container/RedTeam but deserves a dedicated network pass once subnet/security-group design exists.
- The triage Lead Hypotheses predicted the cross-tenant PHI path, the public key, and the container-escape chain — all confirmed. Nothing from the initial threat model went unaddressed.

---

## Section 3 — Unified Risk Register
> Audience: All

| ID | Severity | Domain(s) | Finding | Status |
|---|---|---|---|---|
| SEC-001 | CRITICAL | AppSec, Cloud, AI | Cross-tenant PHI via app-only isolation + sequential IDs + mass assignment + unscoped RAG/S3 | MUST FIX |
| SEC-002 | CRITICAL | DevSecOps, TPRM | Live Anthropic API key baked into a public-read ECR image | MUST FIX (today) |
| SEC-003 | CRITICAL | Platform | Shared HS256 JWT secret across services; claims trusted downstream (forgeable admin tokens) | MUST FIX |
| SEC-004 | CRITICAL | Container | API/worker runs as root and mounts the node Docker socket (node escape on any RCE) | MUST FIX |
| SEC-005 | HIGH | Cloud, Container | Workers run as root with broad S3 IAM, sharing the API image | MUST FIX |
| SEC-006 | HIGH | Data, SecOps | Full request bodies + Sentry context log PHI to lower-trust pipelines | SHOULD FIX |
| SEC-007 | HIGH | Cloud, Data | Backups share prod account + KMS key; no Object Lock (ransomware/erasure blast radius) | SHOULD FIX |
| SEC-008 | HIGH | GRC | No BAAs with PHI-receiving vendors; no hard-delete/erasure path | SHOULD FIX |
| SEC-009 | HIGH | Data, DevSecOps | Staging refreshed from prod PHI; identifiers leak to Segment | SHOULD FIX |
| SEC-010 | HIGH | SecOps | No PHI-access alerting, no immutable audit trail, single-region CloudTrail, no egress detection | SHOULD FIX |
| SEC-011 | HIGH | TPRM, Platform | Google SSO over-scoped (`drive.readonly`); long-lived un-offboarded vendor keys | SHOULD FIX |
| SEC-012 | MEDIUM | AppSec | No MFA / rate-limit / lockout on auth | SHOULD FIX |
| SEC-013 | MEDIUM | Mobile | On-device PHI storage + certificate pinning unspecified | MONITOR (pre-mobile-build) |
| SEC-014 | MEDIUM | Container, DevSecOps | No admission control / image scanning / SBOM; unpinned base and actions | MONITOR |
| SEC-015 | LOW | AI | LLM input-size/cost controls | MONITOR |

*Note: MFA (SEC-012) is rated Medium as a standalone control but is a **blocking dependency** of SEC-001's fix — it appears in the pre-implementation actions below.*

---

## Section 4 — Domain Findings (Full)
> Audience: Engineering Manager, Security Lead

Full per-domain findings are in the sibling files in this folder, each mapped to the Risk Register IDs above:
- [`ai-security.md`](ai-security.md) → SEC-001, SEC-015
- [`product-app-security.md`](product-app-security.md) → SEC-001, SEC-012
- [`grc-security.md`](grc-security.md) → SEC-008
- [`secops.md`](secops.md) → SEC-006, SEC-010
- [`devsecops.md`](devsecops.md) → SEC-002, SEC-009, SEC-014
- [`cloud-security.md`](cloud-security.md) → SEC-001, SEC-005, SEC-007
- [`platform-security.md`](platform-security.md) → SEC-003, SEC-011
- [`mobile-security.md`](mobile-security.md) → SEC-013
- [`container-security.md`](container-security.md) → SEC-004, SEC-005, SEC-014
- [`data-security.md`](data-security.md) → SEC-006, SEC-007, SEC-009
- [`tprm-security.md`](tprm-security.md) → SEC-002, SEC-011
- [`threat-intel.md`](threat-intel.md) → prioritization + detection handoff for SEC-001/002/007/010
- [`red-team.md`](red-team.md) → the chains across SEC-001,3,4,5,7

---

## Section 5 — Recommended Actions
> Audience: Product Team, Engineering Manager

**Today (decoupled from the redesign):**
1. **Rotate the Anthropic key, remove it from the Dockerfile, make ECR private.** Owner: Platform. Low effort. Addresses SEC-002.
2. **Stop sending PHI to vendors without a BAA; start BAA execution with Anthropic, AWS, Twilio, Sentry.** Owner: Compliance + Eng. Addresses SEC-008.

**Before implementation begins:**
3. **Enforce per-object authorization at the data layer + non-sequential IDs, and enforce MFA.** Owner: AppSec + Backend. Addresses SEC-001, SEC-012.
4. **Remove the Docker-socket mount; split and de-privilege the worker (non-root, least-privilege IAM).** Owner: Platform/Infra. Addresses SEC-004, SEC-005.
5. **Re-architect JWT signing to asymmetric keys; re-verify authorization server-side.** Owner: Platform. Addresses SEC-003.
6. **Scope RAG/S3 access to `clinic_id`; confirm vector retrieval can't cross tenants.** Owner: AI/Backend. Addresses SEC-001.

**Before first production PHI:**
7. **Redact PHI from logs/Sentry; mask staging; strip identifiers from analytics.** Addresses SEC-006, SEC-009.
8. **Isolate backups (separate key + account, Object Lock); add hard-delete/erasure path.** Addresses SEC-007, SEC-008.
9. **Add PHI-access alerting, immutable audit logging, all-region CloudTrail, egress detection.** Addresses SEC-010.
10. **Drop the Google `drive.readonly` scope; institute vendor onboarding + key offboarding.** Addresses SEC-011.

**Post-ship (within 30 days):**
11. **Admission control (PSS `restricted`), image scanning + SBOM, pin base/actions.** Addresses SEC-014.
12. **Finalize mobile storage/pinning design before the mobile build.** Addresses SEC-013.
