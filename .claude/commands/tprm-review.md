Review a completed TPRM questionnaire for the vendor at: $ARGUMENTS

You are orchestrating the vendor risk determination. Execute all phases in order.

---

## Phase 1 — Locate materials

- Resolve `$ARGUMENTS` to a vendor folder: if it looks like a slug (no `/`), use `tprm/vendors/[slug]/`; if it looks like a path, use it directly.
- Verify `analysis.md` and `questionnaire.md` exist in that folder. If either is missing, stop and tell the user to run `/tprm-questionnaire` first — do not proceed.
- Look for vendor-supplied material under `<folder>/vendor-response/` (any files — completed questionnaire answers, evidence docs, certifications). If that subfolder is missing or empty, ask the user where the vendor's completed answers and evidence are before continuing. Do not guess or proceed without them.

---

## Phase 2 — Review

Read `analysis.md`, `questionnaire.md`, and every file under `vendor-response/` (the agent you spawn will also read them directly — pass the paths, and inline any short text content you already have).

Spawn a TPRMSecurity agent (subagent_type: TPRMSecurity) with this prompt:

> You are in **Questionnaire Review Mode**. A vendor has completed a TPRM questionnaire; synthesize their answers and evidence against the concerns raised during intake, and produce the final risk determination.
>
> Analysis (from intake): [full path to analysis.md — read it]
> Questionnaire sent: [full path to questionnaire.md — read it]
> Vendor response and evidence: [full path(s) to files under vendor-response/ — read them]
>
> Follow your Questionnaire Review Mode instructions and output format exactly. After producing the report in your response, save it to: [folder]/tprm-review-report.md

Wait for it to finish before proceeding.

---

## Phase 3 — Summary

Report to the user with this structure:

```
## TPRM Review Complete

| Field | Value |
|---|---|
| Vendor | [Company name] |
| Risk tier | [Critical / High / Medium / Low — as determined in this review] |
| Verdict | [Approve / Approve with conditions / Reject / Needs more info] |
| Report saved | [full path to tprm-review-report.md] |
```

Then list the top 2–3 conditions or findings driving the verdict.
