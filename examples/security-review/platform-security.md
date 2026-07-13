# Example: PlatformSecurity on ClariNote PRD

> Agent: `PlatformSecurity` (Sonnet) · Input: [`../sample-system/PRD.md`](../sample-system/PRD.md)
> Illustrative domain output.

---

## Platform Security Review

### Critical findings
| # | Area | Finding | Scenario | Fix |
|---|---|---|---|---|
| P-001 | JWT signing / identity | Sessions are JWTs signed with **HS256 using one shared secret across all services** (§4). Every service can mint tokens for any `clinic_id`/`role`, and a leak of the secret from *any* service (or the public image, see DevSecOps DS-001) forges admin tokens for every tenant. | An attacker who obtains the shared secret mints a JWT with `role: admin, clinic_id: <any>` and accesses any clinic's PHI. Symmetric shared secrets across a fleet maximize this blast radius. | Move to asymmetric signing (RS256/EdDSA): a single signer holds the private key; services verify with the public key. Store the private key in a secrets manager/KMS, rotate it, and never bake it into images. |
| P-002 | Authorization trust model | Downstream services "trust `clinic_id` and `role` claims" from the JWT (§4) — authorization decisions are made on unverified client-presented claims rather than re-checked against the datastore. | Combined with mass assignment (AppSec A-002) or a forged token (P-001), an attacker asserts `role: admin` and downstream services honor it. | Treat JWT claims as identity, not authorization. Re-verify role/tenant membership server-side against the database on privileged actions. |

### High findings
| # | Area | Finding | Scenario | Fix |
|---|---|---|---|---|
| P-003 | OAuth scope (Google SSO) | Google SSO requests `.../auth/drive.readonly` (§5) — read access to the user's entire Google Drive, unrelated to authentication. Over-scoped and unnecessary. | ClariNote (or a stolen refresh token) can read clinicians' full Drive contents; a ClariNote compromise becomes a Drive-data compromise across all SSO users (ties to TPRM). | Request only `openid email profile` for SSO. Remove the `drive.readonly` scope entirely unless a named feature requires it, in which case use `drive.file`. |
| P-004 | Kubernetes RBAC / tenancy | Single EKS cluster, one namespace for all services, developers set their own pod specs (§4.1). No workload separation or RBAC boundaries described. | A compromised pod's service account, if broadly permitted, can read secrets or affect other services in the shared namespace (blast radius overlaps ContainerSecurity). | Namespace-per-trust-boundary, least-privilege RBAC per service account, `automountServiceAccountToken: false` where unused, and admission policy to enforce it. |

### Medium / Low findings
| # | Area | Finding | Scenario | Fix |
|---|---|---|---|---|
| P-005 | SSO enforcement / SP-side | Google SSO is optional per clinic and coexists with password login; no mention of SSO-only enforcement or JIT provisioning controls. | Clinics that enable SSO still have password fallback, widening the auth surface. | Allow admins to enforce SSO-only, disable password fallback per clinic, and validate the `hd`/domain claim. |
| P-006 | Token lifetime / revocation | No stated JWT lifetime or revocation (overlaps AppSec A-006). | Long-lived tokens can't be revoked after compromise. | Short-lived access tokens + refresh with server-side revocation. |

### What's done well
- SSO is offered for clinic admins (§2, §5) — the right direction for enterprise identity, once the scope (P-003) is minimized and enforcement (P-005) is added.

### Verdict
**BLOCK** — P-001 and P-002 together mean the platform's identity and authorization model is forgeable: a shared HS256 secret (leaked via the public image) plus claim-trusting services allow minting admin tokens for any tenant. Combined with the over-scoped Drive grant (P-003), an identity compromise is unusually severe. Re-architect signing and authorization enforcement before production.
