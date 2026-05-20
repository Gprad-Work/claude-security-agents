Review the following detection rule: $ARGUMENTS

$ARGUMENTS can be:
- A rule ID (e.g. AUTH-001)
- A file path (e.g. detection-rules/rules/auth/rule-AUTH-001-impossible-travel.yml)
- A glob pattern (e.g. detection-rules/rules/auth/ to review all rules in a category)

---

## Single rule review

If $ARGUMENTS identifies a single rule, spawn a DEReviewRule agent (subagent_type: DEReviewRule) with this prompt:

> Review the following detection rule.
>
> Target: $ARGUMENTS
>
> If given a rule ID, find the file with: find detection-rules/rules -name "*[ID]*"
> If given a file path, use it directly.
>
> Run the efficacy checker, then complete your full peer review (detection logic, thresholds, FP coverage, MITRE accuracy, operational quality, stack-specific issues). Return your full review and verdict.

---

## Multi-rule review (category or directory)

If $ARGUMENTS is a directory path, spawn a DEReviewRule agent for each rule file found. Run them in parallel (run_in_background: true for all). Collect all verdicts and report a summary table:

| Rule | Score | Verdict | Blocking issues |
|---|---|---|---|
| AUTH-001 | 85/100 | APPROVE | — |
| AUTH-002 | 60/100 | REVISE | Broad FP list at HIGH level |

---

## Report to the user

After the review(s) complete:

1. Show the full review output for each rule
2. For any rule with verdict APPROVE WITH FIXES: list the blocking issues and ask the user whether to apply fixes now
3. For any rule with verdict REVISE or REJECT: summarise what needs to change and ask whether to spawn the rule writer to produce a replacement
