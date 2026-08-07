# Example: FraudAbuse on ClariNote PRD

> Agent: `FraudAbuse` (Sonnet) · Input: [`../sample-system/PRD.md`](../sample-system/PRD.md)
> Illustrative domain output. ClariNote is a B2B tool with contracted, paying clinic customers — so the consumer-abuse surface is genuinely smaller than a marketplace. This example shows the agent calibrating to that rather than padding.

---

## Fraud & Abuse (Trust & Safety) Review

### Abuse surface summary
ClariNote is a B2B SaaS sold to vetted, contracted clinics — there is no open self-serve consumer signup, no payments between users, no user-generated public content, and no promo/referral economy. That removes most classic consumer-abuse vectors. What remains and matters: **account takeover of clinician accounts** (high-value PHI access, and the design has no MFA), and **resource/cost abuse of the LLM summarization pipeline** (each upload spends Anthropic tokens). A secondary concern is insider/authorized-user abuse — a legitimate clinic user acting at illegitimate scale.

### Critical findings
| # | Abuse type | Feature / flow | Finding | Abuse scenario (attacker motive + method + your cost) | Fix |
|---|---|---|---|---|---|
| FA-001 | Account takeover economics | Clinician login (§4) | Accounts protect bulk PHI but have no MFA and no described velocity/anomaly control on login. The *value* behind a taken-over account is extreme (multi-tenant PHI via the BOLA path), making these accounts a high-ROI target. | Attacker credential-stuffs or phishes one clinician (cheap), then uses the session to enumerate PHI across clinics (RedTeam Path 2). Cost to you: a multi-clinic breach from one $0 login. | Enforce MFA and step-up on clinician/admin accounts; add login velocity/anomaly signals (impossible travel, new-device, VPN/hosting ASN). *(Control design shared with ProductAppSecurity A-003 / SecOps; framed here as raising attacker cost on a high-value account.)* |

### High findings
| # | Abuse type | Feature / flow | Finding | Abuse scenario | Fix |
|---|---|---|---|---|---|
| FA-002 | Resource / cost abuse | Upload → summarization (§3.1–3.2) | Every upload triggers OCR + a Claude API call, with no described per-tenant quota or spend cap. A compromised or malicious authorized account can convert your LLM budget into an expense. | An abuser (or a compromised front-desk account, which *can* upload) submits large/many documents to run up Anthropic token spend and delay legitimate jobs. Cost: direct API spend + degraded service. | Per-tenant quota on summarization; cap document size/count; spend-anomaly alerting on Anthropic usage. *(Ties to APISecurity API-003 and AISecurity AI-005 — same control, abuse-economics view.)* |
| FA-003 | Authorized-user abuse at scale | Patient/summary access | A legitimate clinic user can access far more records than clinical use requires, and there's no velocity/breadth control or detection. The "attacker" here is an insider or a curious/malicious staff member, not an outsider. | A front-desk or clinician account scrapes patient records for resale or curiosity; indistinguishable from normal use without breadth detection. | Velocity/breadth limits and detection on PHI access (DATA-001 rule); least-privilege per role; audit visibility. *(Detection wiring handoff: SecOps S-001.)* |

### Medium / Low findings
| # | Abuse type | Feature / flow | Finding | Abuse scenario | Fix |
|---|---|---|---|---|---|
| FA-004 | Ban/detection evasion | Account model | If a clinic contract is terminated for abuse, there's no described durable signal (device/payment/behavioral) to prevent immediate re-onboarding under a new clinic identity. | A bad-acting clinic re-signs under a new entity. Low likelihood given B2B contracting, but no friction exists. | Retain durable abuse signals; gate re-onboarding review. |
| FA-005 | Response tooling | Operations | No described mechanism to rate-limit, quarantine, or suspend an abusing account without a code deploy. | When abuse is detected, response is slow. | Build runtime controls to throttle/suspend accounts and claw back quota. |

### What's done well
- The B2B, contracted-customer model is itself a strong anti-abuse control: vetted customers, no anonymous signup, and no inter-user payment or content surface eliminate whole categories of consumer fraud by design. The residual risk concentrates on account value and cost abuse, which is the right place to focus.

### Verdict
**MEDIUM RISK** — Most consumer-abuse vectors don't apply to this B2B model, and the report says so rather than inventing them. The two that genuinely matter are account-takeover economics (FA-001 — high-value accounts with no MFA) and LLM cost abuse (FA-002 — an unmetered, cost-bearing pipeline). Both share controls already recommended by other domains; the abuse lens confirms they're priorities because of *what an account is worth* and *what the pipeline costs*, not just because of a vulnerability.
