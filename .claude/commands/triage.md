Triage the following alert: $ARGUMENTS

You are orchestrating an incident response triage. Execute all four phases in order. Do not collapse or skip any phase.

---

## Phase 1 — Alert Parsing

Spawn an IRAlertParser agent (subagent_type: IRAlertParser) with this prompt:

> Parse the alert below and produce a structured Alert Context.
>
> If a rule ID (e.g. AUTH-001, AI-002) or rule file path is referenced in the alert, find and read the corresponding YAML in detection-rules/rules/ — search subdirectories. The rule ID is in the filename (e.g. rule-AUTH-001-impossible-travel.yml).
>
> Alert input:
> $ARGUMENTS

Wait for the Alert Context output before proceeding.

---

## Phase 2 — SIEM Investigation

Spawn an IRSIEMInvestigator agent (subagent_type: IRSIEMInvestigator) with this prompt:

> Investigate the SIEM for events correlated to the following Alert Context.
>
> Query around the full investigation timeframe and all extracted entities. Run pivot queries. Test every false positive condition. Build a complete timestamped event timeline.
>
> Alert Context:
> [paste the full Alert Context output from Phase 1 verbatim]

Wait for the Event Timeline before proceeding.

---

## Phase 3 — Triage Analysis

Determine the output path for the triage report:
- Extract the rule ID from the Alert Context (e.g. AUTH-001)
- Today's date: use YYYY-MM-DD format
- Output directory: `detection-rules/cases/YYYY-MM-DD-[rule-id]/`
- Output file: `triage-report.md`
- Create the directory if it does not exist.

Spawn an IRAnalyst agent (subagent_type: IRAnalyst) with this prompt:

> Produce a triage verdict for this incident, then save the report.
>
> Alert Context:
> [paste the full Alert Context from Phase 1 verbatim]
>
> SIEM Event Timeline:
> [paste the full Event Timeline from Phase 2 verbatim]
>
> After producing the report in your response, save it to: [full output path]

Wait for the full triage report before proceeding.

---

## Phase 4 — Summary

Report to the user with this exact structure:

```
## Triage Complete

| Field | Value |
|---|---|
| Rule | [rule ID] — [rule title] |
| Verdict | [TRUE POSITIVE / FALSE POSITIVE / TRUE NEGATIVE (PENDING) / NEEDS INVESTIGATION] |
| Confidence | [HIGH / MEDIUM / LOW] |
| Severity | [CRITICAL / HIGH / MEDIUM / LOW / N/A] |
| Kill chain | [MITRE stage(s) and technique IDs, or N/A] |
| Affected users | [list or N/A] |
| Report saved | [full path to triage-report.md] |
```

If the verdict is TRUE POSITIVE, also list the immediate containment actions from the report.
If the verdict is NEEDS INVESTIGATION, list what specific data is missing.
