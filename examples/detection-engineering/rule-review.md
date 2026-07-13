# Example: /rule-review → DEReviewRule

> Command: `/rule-review data-access/rule-DATA-001-bulk-patient-enumeration.yml`
> Agent: `DEReviewRule` (Opus)
> Peer review of the rule authored in [`rule-write.md`](rule-write.md). Runs the validators, then applies senior judgment.

---

## Rule Review — DATA-001

| Field | Value |
|---|---|
| Rule | DATA-001 — Bulk patient record enumeration by a single actor |
| `validate_schema.py` | PASS |
| `efficacy_check.py` | 82 / 100 |
| **Verdict** | **APPROVE WITH FIXES** |

### Automated results
- **Schema:** valid — all required fields present, aggregation block well-formed, `timeframe` present with `distinct_count()`.
- **Efficacy 82:** strong specificity (exact route + method + status), a real filter, populated `fields`, `high` level with 2 FPs. Points deducted for a static threshold with no per-tenant normalization and a single groupby dimension.

### Qualitative review

**Detection logic — sound, with one evasion path.**
The `distinct_count(patient_id)` over a 5-minute window correctly targets enumeration breadth rather than request volume — this is the right primitive and won't be tripped by a clinician reloading one chart. However, the rule is **evadable by slow enumeration**: an attacker reading 25 patients per 5-minute window stays under threshold indefinitely while still scraping thousands per day. A patient system should also carry a **longer-window, lower-rate companion** (e.g., `distinct_count(patient_id) >= 200` over `24h`) so "low and slow" is caught even when the burst rule isn't.

**False-positive coverage — adequate but verify the filter field.**
The two FP scenarios are realistic and the service-account filter handles the main benign bulk-reader. **Fix:** confirm `actor.role: service` is actually emitted in the application log schema — if roles aren't logged on the data-access event, the filter silently matches nothing (fails open, more FPs) or errors. Prefer filtering on a dedicated `actor.type` field if one exists.

**MITRE mapping — correct.**
T1530 (Data from Cloud Storage / repository) is the right primary technique for bulk PHI reads; T1078 (Valid Accounts) correctly captures that this is abuse of a legitimate session. Consider adding `attack.t1213` (Data from Information Repositories) as it maps closely to application-DB enumeration.

**Operational quality — good, one addition.**
`fields` gives the analyst what they need (actor, clinic, IP, UA). **Suggestion:** the triage playbook should cross-reference whether the accessed `patient_id`s span multiple `clinic_id`s — cross-tenant breadth is a much stronger TP signal than in-tenant volume and directly indicates the IDOR (AppSec SEC-001) being exploited. Consider a second, higher-severity rule keyed on `distinct_count(clinic_id) >= 2` by actor.

### Required fixes before `stable`
1. Add a low-and-slow companion rule (longer window, higher cumulative threshold) — or document that DATA-001 is burst-only.
2. Verify `actor.role`/`actor.type` is present on the data-access event schema; adjust the filter field if not.

### Recommended (not blocking)
3. Add `attack.t1213`.
4. Author a cross-tenant-breadth variant (`distinct_count(clinic_id) >= 2`) at `critical`.

### Verdict rationale
The rule is specific, correctly aggregated, and catches the primary attack. It ships as `status: test` now; the low-and-slow gap and the filter-field verification must be resolved before promotion to `stable`. This is exactly the kind of rule that survives contact with a real SOC — provided the threshold is tuned against real access baselines rather than left at the default 30.
