Write a new detection rule for the following threat scenario: $ARGUMENTS

You are orchestrating detection rule creation and automated review. Execute both phases in order.

---

## Phase 1 — Write the Rule

Spawn a DEDetectionRuleWriter agent (subagent_type: DEDetectionRuleWriter) with this prompt:

> Write a complete, production-ready Sigma-format detection rule for the following threat scenario.
>
> Threat scenario: $ARGUMENTS
>
> Determine the correct category, assign the next sequential rule number, generate a UUID, write the rule, self-check it against all efficacy checks, fix any issues, and save it.
>
> Return the full rule YAML and the path it was saved to.

Wait for the rule writer to complete and note: (1) the saved file path, (2) any efficacy issues the writer flagged and fixed.

---

## Phase 2 — Automated Review

Spawn a DEReviewRule agent (subagent_type: DEReviewRule) with this prompt:

> Review the detection rule that was just written.
>
> Rule file: [paste the file path returned by Phase 1]
>
> Run the efficacy checker, then complete your full peer review. Return your verdict and all findings.

Wait for the review to complete.

---

## Report to the user

```
## Rule Written & Reviewed

| Field | Value |
|---|---|
| Threat scenario | $ARGUMENTS |
| Rule file | [path] |
| Rule ID | [e.g. AUTH-004] |
| Title | [rule title] |
| Level | [level] |
| MITRE | [technique IDs] |
| Efficacy score | [N/100] |
| Review verdict | APPROVE / APPROVE WITH FIXES / REVISE / REJECT |
```

If the verdict is APPROVE WITH FIXES or REVISE: list the blocking issues and apply the fixes immediately using the Edit tool, then confirm the fixes were made.

If the verdict is REJECT: explain why and do not leave the file in the rules directory — move it to `detection-rules/rules/_drafts/` and tell the user what needs to change.
