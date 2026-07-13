# Example: ProductAppSecurity on ClariNote PRD

> Agent: `ProductAppSecurity` (Sonnet) · Input: [`../sample-system/PRD.md`](../sample-system/PRD.md)
> Illustrative domain output.

---

## Product & Application Security Review

### Critical findings
| # | OWASP | Endpoint / Feature | Finding | Exploit scenario | Fix |
|---|---|---|---|---|---|
| A-001 | A01 | `GET /api/patients/{patient_id}` (§3.4) | Patient IDs are sequential integers and the PRD describes tenant isolation only as "add `WHERE clinic_id = ?` in the service layer." Object-level ownership is not enforced per request, and the enumerable ID makes any missed check catastrophic. | An authenticated user in Clinic A calls `/api/patients/1`, `/2`, `/3`… Any endpoint path or query that forgets the `clinic_id` predicate returns another clinic's patient. Sequential IDs turn one gap into a full-corpus scrape. | Enforce authorization at the data layer for every patient/summary fetch: the query must join `clinic_id` from the authenticated session, not a parameter. Use non-sequential UUIDs. Return 404 (not 403) on cross-tenant access. |
| A-002 | A01 / A08 | `PATCH /api/summaries/{summary_id}` (§3.3) | The endpoint "accepts the full summary object from the client and saves it." This is mass assignment: any field the model has — `clinic_id`, `patient_id`, `signed_by`, `status`, `id` — can be set by the caller. | A caller PATCHes a summary with `clinic_id` changed, moving a record between tenants, or sets `signed_by` to a clinician who never reviewed it, forging a signed clinical record. | Bind only an explicit allowlist of editable fields (the summary body). Set `signed_by`, `signed_at`, `clinic_id`, and `status` server-side from session + workflow state. Re-verify object ownership before write. |

### High findings
| # | OWASP | Endpoint / Feature | Finding | Exploit scenario | Fix |
|---|---|---|---|---|---|
| A-003 | A07 | Authentication (§4) | Email/password login with no described rate limiting, lockout, or MFA. Healthcare accounts are high value. | Credential stuffing against the login endpoint using breach corpora; no lockout or MFA means a valid reused password grants PHI access. | Add per-account and per-IP rate limiting on login/reset, account lockout with owner notification, and enforce MFA at least for clinician and admin roles. |
| A-004 | A01 | Function-level authz — Front-desk role (§2) | Front-desk "cannot view signed summaries," but the PRD shows summaries fetched via `GET /api/summaries/{id}` with no per-role gate described. UI-hiding is not enforcement. | A front-desk user calls `GET /api/summaries/{id}` directly and reads signed summaries the UI hides from them. | Enforce role checks server-side on every summary read; deny front-desk access to signed summaries at the API, not the client. |

### Medium / Low findings
| # | OWASP | Endpoint / Feature | Finding | Exploit scenario | Fix |
|---|---|---|---|---|---|
| A-005 | A04 | Upload endpoint (§3.1) | No described validation of file type, size, or count on upload. | Malicious or oversized files drive OCR/worker cost and may carry injection payloads (see AI-001) or malware for downstream viewers. | Validate content type and size server-side, cap per-tenant upload rate, and scan files. |
| A-006 | A09 | Session / JWT (§4) | JWTs are trusted downstream and there's no described server-side revocation; logout likely just drops the client token. | A stolen token remains valid until expiry with no way to revoke after a reported compromise. | Add a server-side session/revocation list or short-lived tokens with refresh; invalidate on logout and on suspected compromise. |

### What's done well
- Roles are explicitly enumerated (§2) with intended data boundaries — the model is clear; the gap is enforcement, which is fixable without redesign.

### Verdict
**BLOCK** — A-001 and A-002 are a cross-tenant PHI read-and-write pair: sequential IDs plus data-layer-optional authorization plus mass assignment mean an ordinary authenticated user can likely read other clinics' patients and forge signed records. These must be fixed and re-reviewed before any real PHI is processed.
