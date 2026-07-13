# Example: APISecurity on ClariNote PRD

> Agent: `APISecurity` (Sonnet) · Input: [`../sample-system/PRD.md`](../sample-system/PRD.md)
> Illustrative domain output. Note the deliberate overlap-with-handoff to ProductAppSecurity — the Lead merges these in synthesis.

---

## API Security Review

### Endpoint inventory (as understood from the artifact)
| Endpoint | Method(s) | Auth | Object-level authz? | Rate limited? | Notes |
|---|---|---|---|---|---|
| `/api/patients/{patient_id}` | GET | JWT | **Unknown / app-layer only** | Unknown | Sequential integer IDs |
| `/api/summaries/{summary_id}` | GET, PATCH | JWT | **Unknown / app-layer only** | Unknown | PATCH takes full object |
| upload (web/mobile) | POST | JWT | n/a | Unknown | Triggers OCR + LLM (cost-bearing) |
| *(no versioning, inventory, or deprecation policy described)* | | | | | **API9 gap** |

### Critical findings
| # | API Top 10 | Endpoint | Finding | Exploit scenario | Fix |
|---|---|---|---|---|---|
| API-001 | API1 BOLA | `/api/patients/{id}`, `/api/summaries/{id}` | Object-level authorization is described only as an application-layer `WHERE clinic_id = ?` convention, not verified per endpoint at the data layer. Combined with sequential integer IDs, every object-referencing route is a BOLA candidate. | An authenticated user iterates `/api/patients/1..N`; any route missing the predicate returns other clinics' patients. Sequential IDs turn one gap into a full scrape. | Enforce ownership at the data layer on every object-referencing endpoint (derive `clinic_id` from the session, not the path); use unguessable IDs; return 404 on cross-tenant. *(Merges with ProductAppSecurity A-001 — same root cause, API framing is per-endpoint coverage across the whole surface.)* |

### High findings
| # | API Top 10 | Endpoint | Finding | Exploit scenario | Fix |
|---|---|---|---|---|---|
| API-002 | API3 (property-level) | `PATCH /api/summaries/{id}` | The endpoint accepts the full summary object; there is no response/request property allowlist. Both excessive data exposure (responses) and mass assignment (writes) apply. | Caller sets `clinic_id`/`signed_by`/`status` on write (mass assignment), or reads back internal properties never meant to leave the server. | Define explicit request-input and response-output schemas; bind only editable properties; set workflow/ownership fields server-side. *(Coordinates with ProductAppSecurity A-002.)* |
| API-003 | API4 Unrestricted Resource Consumption | upload → summarization | No rate limits or per-tenant quota described, and each upload triggers OCR + a Claude API call — a downstream cost multiplier. | A user (or a compromised account) floods uploads to run up Anthropic token spend and starve the job queue — cost-amplification DoS. | Per-tenant rate limits and quota on cost-bearing endpoints; cap extracted-text length per request; spend-anomaly alerting. *(Ties to FraudAbuse cost-abuse and AISecurity AI-005.)* |
| API-004 | API9 Improper Inventory Management | all | No API inventory, versioning scheme, or deprecation policy. Staging APIs are refreshed from prod (§7) and may be reachable; debug/internal routes are unaccounted for. | A shadow or staging endpoint with weaker controls (or prod PHI) is discovered and hit directly. | Maintain a current endpoint inventory; adopt explicit versioning + deprecation; ensure non-prod APIs are network-isolated and unreachable from the internet. |

### Medium / Low findings
| # | API Top 10 | Endpoint | Finding | Exploit scenario | Fix |
|---|---|---|---|---|---|
| API-005 | API4 | `/api/patients` reads | No described pagination/result caps on list-style access. | A single call requests an unbounded set, easing bulk extraction. | Enforce a hard server-side max page size; require pagination. |
| API-006 | API8 (misconfig) / CORS | all | CORS, method enforcement, and content-type handling are unspecified. | A permissive CORS policy or reflected origin enables cross-origin API abuse from a malicious page. | Explicit CORS allowlist (no credentialed `*`); reject unsupported methods/content types. |

### What's done well
- Endpoints follow a consistent, predictable REST shape, which makes systematic authorization enforcement and inventory tractable once added.

### Verdict
**BLOCK** — API-001 (BOLA across the object-referencing surface, amplified by sequential IDs) is a cross-tenant PHI read reachable by any authenticated caller, and there is no API inventory or resource-consumption control behind it. Enforce data-layer object authorization on every endpoint and add rate/quota limits before the API is exposed to real tenants. Several findings merge with ProductAppSecurity — the Lead should de-duplicate.
