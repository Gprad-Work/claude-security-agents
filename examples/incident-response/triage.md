# Example: /triage → IRAlertParser → IRSIEMInvestigator → IRAnalyst

> Command: `/triage rule-DATA-001 actor=jsmith@clinic-northgate.example ts=2026-07-13T02:14Z`
> Agents: `IRAlertParser` (Sonnet) → `IRSIEMInvestigator` (Sonnet) → `IRAnalyst` (Opus)
> A full alert-to-verdict run on an alert firing the DATA-001 rule from [`../detection-engineering/rule-write.md`](../detection-engineering/rule-write.md). SIEM data shown here is illustrative (as if a SIEM MCP were configured).

---

## Phase 1 — Alert Context (IRAlertParser)

**Rule:** DATA-001 — Bulk patient record enumeration by a single actor (level: high; T1530, T1078)
**Fired:** 2026-07-13T02:14:07Z
**Trigger:** `distinct_count(patient_id) = 47` by `actor.user_id=jsmith@clinic-northgate.example` over 5m (threshold ≥ 30)

**Key entities**
| Entity | Value |
|---|---|
| Actor | `jsmith@clinic-northgate.example` (role: clinician, clinic_id: northgate) |
| Source IP | 185.220.101.44 |
| Resource | `/api/patients/{id}` (GET, 200) |
| Timestamp | 2026-07-13T02:14:07Z (02:14 local — outside clinic hours) |

**Investigation timeframe:** 2026-07-13T01:44Z → 02:44Z (±30m)

**Initial questions**
1. Do the 47 accessed `patient_id`s belong only to northgate, or do they cross `clinic_id` (IDOR indicator)?
2. Is 185.220.101.44 a known device/location for jsmith?
3. What preceded the reads — a normal login, or an anomalous auth event?
4. Sequential IDs (enumeration) or a clinically plausible set?

---

## Phase 2 — Event Timeline (IRSIEMInvestigator)

| Time (UTC) | Event | Detail |
|---|---|---|
| 02:11:33 | `auth.login.success` | jsmith, IP 185.220.101.44, new device (first-seen), **no MFA** (not enforced) |
| 02:11:34 | `auth.geo` | IP geolocates to a hosting/VPN ASN; jsmith's prior 90d logins all from a residential ASN in-region |
| 02:12:01 | `api.GET /api/patients/1001` | 200 |
| 02:12:01–02:14:05 | `api.GET /api/patients/{1001..1060}` | **sequential IDs**, 200s and some 404s; 47 distinct successful reads |
| 02:13:40 | cross-tenant check | of 47 successful reads, **31 belong to clinic_id ≠ northgate** (foxlake, riverside, others) |
| 02:14:07 | **DATA-001 fires** | distinct_count(patient_id)=47/5m |
| 02:15:22 | `api.GET /api/summaries/{...}` | began pulling summary bodies for enumerated patients |

**Pivots:**
- IP 185.220.101.44 — no prior history for this actor or clinic; matches a commodity VPN range.
- Password reset / credential-change events for jsmith in prior 7d: none → likely credential-stuffed or phished, not an insider using their own new device.

**FP conditions tested (from the rule):**
- Service account? **No** — jsmith is a human clinician role (filter correctly did not suppress).
- Admin bulk/migration task? **No change ticket**; activity is at 02:14 local, sequential IDs, cross-tenant — inconsistent with legitimate chart-prep.

---

## Phase 3 — Verdict (IRAnalyst)

## Triage Complete

| Field | Value |
|---|---|
| Rule | DATA-001 — Bulk patient record enumeration by a single actor |
| Verdict | **TRUE POSITIVE** |
| Confidence | HIGH |
| Severity | **CRITICAL** |
| Kill chain | Initial Access (T1078 Valid Accounts) → Discovery/Collection (T1530, T1213) → in progress: Exfiltration |
| Affected users | jsmith@clinic-northgate (compromised account); patients across ≥3 clinics (northgate, foxlake, riverside) |
| Report saved | `detection-rules/cases/2026-07-13-DATA-001/triage-report.md` |

### Reasoning
This is a compromised-credential account being used to exploit the cross-tenant IDOR (the exact SEC-001 path from the security review). Five independent signals align: (1) login from a first-seen VPN IP inconsistent with 90 days of history, (2) no MFA to stop it, (3) strictly sequential patient-ID access, (4) **31 of 47 records belong to other clinics** — impossible in legitimate clinical use and a direct IDOR indicator, and (5) escalation into summary-body reads. The service-account filter correctly did not suppress this. Cross-tenant breadth makes this CRITICAL rather than a single-tenant curiosity.

### Immediate containment
1. **Disable jsmith's session and account**; force credential reset.
2. **Block 185.220.101.44** at the edge; check for other accounts authenticating from it.
3. **Scope the exposure**: enumerate exactly which `patient_id`/`clinic_id` records were returned 200 (breach-notification scope spans multiple clinics → multiple BA notifications).
4. **Page the IDOR owner** — the vulnerability (SEC-001), not just the account, is the root cause; the account reset does not close the hole.

### Next investigation steps
- Pull full response sizes to confirm what PHI was actually returned per record.
- Check whether the enumerated summaries were exfiltrated (egress volume from the API to 185.220.101.44) — DATA-003 (proposed in the coverage scan) would have caught this leg.
- Review other clinician accounts for similar first-seen-IP logins in the prior 7 days.

### What this incident proves about the design
The alert worked, but it fired at the *collection* stage because the earlier controls were absent: MFA would have blocked step 1, and per-object data-layer authorization would have made steps 3–4 return 403s instead of PHI. Detection is not a substitute for the SEC-001/SEC-012 fixes — it is the backstop for when they're missing.
