# Detection Engineering

Three commands and three agents for writing, reviewing, and maintaining a quality-gated Sigma detection rule library.

---

## Commands

| Command | What it does |
|---|---|
| `/rule-write <scenario>` | Write a new rule + automated peer review |
| `/rule-review <rule-id or path>` | Peer review existing rule(s) |
| `/coverage-scan` | ATT&CK gap analysis across the full library |

---

## The pipeline

```
/rule-write "brute force on admin login endpoint"
        │
        ▼
DEDetectionRuleWriter (Sonnet)
  - Picks category and assigns rule ID
  - Generates schema-compliant Sigma YAML
  - Adds MITRE tags, filter blocks, aggregation logic
  - Self-checks against efficacy criteria
  - Saves to detection-rules/rules/<category>/
        │
        ▼
DEReviewRule (Opus)
  - Runs validate_schema.py and efficacy_check.py
  - Reviews detection logic, thresholds, FP coverage
  - Checks for evasion paths
  - Issues verdict: APPROVE / APPROVE WITH FIXES / REVISE / REJECT
        │
        ▼
Result reported to user
  - F-grade rules → moved to _drafts/, not shipped
  - Fixes applied inline if verdict is APPROVE WITH FIXES
```

---

## Repo structure expected

The agents assume this layout in your project:

```
detection-rules/
├── rules/
│   ├── auth/
│   │   └── rule-AUTH-001-impossible-travel.yml
│   ├── api/
│   ├── data-access/
│   └── ...
├── validators/
│   ├── validate_schema.py
│   ├── efficacy_check.py
│   └── coverage_report.py
└── coverage-report-YYYY-MM-DD.md
```

The validators are Python scripts that the agents run directly. You need to provide these — the agents expect them at `detection-rules/validators/`. Example implementations:

- `validate_schema.py` — validates Sigma YAML structure and required fields
- `efficacy_check.py` — scores rules 0–100 based on specificity, FP risk, field coverage
- `coverage_report.py` — maps rules to ATT&CK tactics and produces a gap report

---

## Rule ID format

Rules are assigned IDs by category:

| Category | Prefix | Example |
|---|---|---|
| Authentication | `AUTH` | `AUTH-001` |
| API activity | `API` | `API-003` |
| Data access | `DATA` | `DATA-002` |
| Repository events | `REPO` | `REPO-001` |
| AI pipeline | `AI` | `AI-002` |

The rule writer assigns the next sequential number within a category automatically.

---

## Efficacy score

The reviewer runs `efficacy_check.py` and uses the score as a signal:

| Score | Verdict signal |
|---|---|
| 80–100 | APPROVE |
| 60–79 | APPROVE WITH FIXES |
| 40–59 | REVISE |
| 0–39 | REJECT — do not ship |

The score is a floor, not a ceiling. Opus may approve a 65-scoring rule if the use case justifies it, or reject an 80-scoring rule if the detection logic has a structural evasion path.

---

## Coverage scan

`/coverage-scan` maps your current rule library against the MITRE ATT&CK framework:

```
/coverage-scan
```

Output:
- Coverage snapshot: how many tactics have at least one rule, percentage covered
- Tactic coverage table: per-tactic rule count and gaps
- HIGH priority recommendations: specific rule specs ready to hand to `/rule-write`

After the scan you're asked: **"Should I write the highest-priority missing rules now?"** — if yes, it chains into `/rule-write` for each HIGH priority gap in order.

---

## Customisation

Before using, update two things in `.claude/agents/detection-engineering/de-rule-reviewer.md`:

**Section F — stack-specific checks**: replace the example log source field names with your own (Okta, AWS CloudTrail, your app's event schema, etc.)

**Author field** in `.claude/agents/detection-engineering/de-rule-writer.md`: replace `<your-name>` with your name or team name.
