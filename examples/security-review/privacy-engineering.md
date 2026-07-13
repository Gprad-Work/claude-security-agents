# Example: PrivacyEngineering on ClariNote PRD

> Agent: `PrivacyEngineering` (Sonnet) · Input: [`../sample-system/PRD.md`](../sample-system/PRD.md)
> Illustrative domain output. Note the mechanism-vs-compliance split from GRC — the Lead merges the overlapping erasure finding.

---

## Privacy Engineering Review

### PII data map
| PII category | Sensitivity | Basis / purpose | Where it lives & flows (all copies) | Privacy control state |
|---|---|---|---|---|
| Patient documents / summaries | Special-category (health) | Clinical summarization | S3, Postgres, **Pinecone embeddings**, logs, Sentry, staging, backups | No minimization; no purpose limit on LLM/embeddings; erasure doesn't reach copies |
| Patient demographics | Identifier + quasi-identifier | Record identification | Postgres, Twilio (name+phone), logs | Sent to Twilio; retained indefinitely |
| Patient contact (phone) | Identifier | SMS reminders | Twilio | No described consent/opt-out for SMS |
| Navigation/URL data | Quasi (may embed IDs) | Analytics | Segment | Identifiers likely leak in URL paths |

### Critical findings
| # | Principle | Component | Finding | Privacy harm scenario | Fix |
|---|---|---|---|---|---|
| PRIV-001 | Erasure (mechanism) | Deletion flow (§7) | Deletion is soft delete (`deleted_at`) on the clinic row. The erasure *mechanism* does not reach S3 documents, Pinecone embeddings, logs, Segment, or backups (shared prod KMS key). Even if a legal request is honored on paper, the data map shows data surviving in ≥5 stores. | A patient/clinic exercises erasure; their documents, summary embeddings, and logged PHI persist across storage indefinitely, re-surfacing in retrieval or a later breach. | Build an erasure job that enumerates the data map and deletes/anonymizes in every store; adopt per-tenant crypto-shredding so backups are covered. *(GRC G-002 owns the legal requirement; this is the implementation gap.)* |
| PRIV-002 | Purpose limitation / secondary use | Summarization + embeddings (§3.2) | PHI is sent to the Claude API and retained as embeddings in Pinecone for retrieval. Nothing enforces that this special-category data isn't used beyond the stated clinical purpose, and it's unverified whether the vendor retains/trains on it. Embeddings (derived PHI) aren't classified or protected as PHI. | Health data is repurposed or retained by a subprocessor without basis; derived embeddings are treated as non-sensitive and leak (see AISecurity AI-002 cross-tenant retrieval). | Contractually confirm no training/retention on PHI and verify the data actually isn't sent beyond purpose; classify and protect embeddings as PHI; scope retrieval by tenant. |

### High findings
| # | Principle | Component | Finding | Privacy harm scenario | Fix |
|---|---|---|---|---|---|
| PRIV-003 | De-identification honesty | Staging refresh (§7) | Staging is "realistic test data" refreshed from a production dump — this is **not** anonymized or even pseudonymized; it is raw PHI relabeled as test data. | Real patients' health data sits in a lower-trust environment; any staging exposure is a real breach (overlaps DataSecurity D-003, framed here as a de-identification failure). | Use synthetic or irreversibly de-identified data for staging; if realistic data is required, apply documented de-identification with re-identification-risk assessment. |
| PRIV-004 | Tracking / analytics leakage | Segment (§5–6) | URL paths sent to Segment likely embed `patient_id`/`summary_id`, linking analytics events to identifiable patients outside the privacy boundary. | Patient identifiers (and via them, health context) accumulate in a third-party analytics platform with its own retention and access. | Strip identifiers/PHI from URLs and event properties before they reach Segment; keep PHI inside the boundary. |
| PRIV-005 | Data minimization | Schema / retention (§7) | Documents and summaries are retained indefinitely with no minimization; all extracted text is stored and embedded. | Ever-growing special-category store maximizes breach impact and violates minimization. | Define per-category retention with automated enforcement reaching all copies; minimize what is stored/embedded to purpose. |

### Medium / Low findings
| # | Principle | Component | Finding | Privacy harm scenario | Fix |
|---|---|---|---|---|---|
| PRIV-006 | DSAR fulfillment (access/portability) | (absent) | No mechanism to produce a complete cross-store export of a subject's data. | An access request can't be fully satisfied across S3/Pinecone/backups. | Build export tooling driven by the same data map as erasure. |
| PRIV-007 | Consent (SMS) | Twilio reminders (§5) | No described consent/opt-out for SMS to patients. | Patients receive messages with no opt-out; contact data shared with Twilio without a clear basis. | Capture/track SMS consent and honor opt-out end-to-end. |

### What's done well
- Data is at least conceptually organized by clinic and patient, giving a foundation for a data map — the raw material for erasure and access tooling, once it's built to traverse every store.

### Verdict
**BLOCK** — The privacy *mechanisms* don't match the promises: erasure can't reach most of the stores that hold PHI (PRIV-001), special-category data is repurposed into an LLM and embeddings with no enforced boundary (PRIV-002), and "test data" is undisguised production PHI (PRIV-003). These are implementation gaps, not just policy gaps — GRC owns the requirements; these must be built before real patient data flows.
