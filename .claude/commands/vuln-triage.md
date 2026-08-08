Triage the following vulnerability finding(s): $ARGUMENTS

You are orchestrating a vulnerability triage. The input is scanner output (SCA/dependency findings), an SBOM + advisory, or a CVE/GHSA ID. Execute all four phases in order. Do not collapse or skip any phase — reachability without exploitability is noise, and exploitability without evidence is an opinion.

The point of this pipeline: sit on top of the user's existing scanner, then prove what is actually **reachable** and **exploitable** in *this* codebase, so a wall of findings becomes the short list that matters — each decision backed by documented evidence.

---

## Phase 1 — Finding parsing

Spawn a VMFindingParser agent (subagent_type: VMFindingParser) with this prompt:

> Parse the vulnerability finding(s) below into a structured Finding Context. Extract the vulnerable symbol (sink), affected/fixed versions, dependency path, and exploit preconditions for each. Do not judge reachability, exploitability, or priority. If a scan file or SBOM path is referenced, read it.
>
> Finding input:
> $ARGUMENTS

Wait for the Finding Context before proceeding.

---

## Phase 2 — Reachability analysis

Spawn a VMReachabilityAnalyst agent (subagent_type: VMReachabilityAnalyst) with this prompt:

> Determine whether each vulnerable sink in the Finding Context is reachable from this application's entry points. Trace imports and function-level call paths through the codebase. Output REACHABLE / NOT REACHABLE / INDETERMINATE per finding with the concrete call path as evidence. Do not judge exploitability.
>
> Finding Context:
> [paste the full Finding Context from Phase 1 verbatim]

Wait for the Reachability Report before proceeding.

---

## Phase 3 — Exploitability analysis

Spawn a VMExploitabilityAnalyst agent (subagent_type: VMExploitabilityAnalyst) with this prompt:

> Determine whether each reachable finding is actually exploitable in this system. Test data flow (does attacker-controlled input reach the sink in exploitable form?), configuration conditions (are the exploit's required flags/modules present?), and entry-point exposure. Produce an evidence-backed EXPLOITABLE / NOT EXPLOITABLE / NEEDS REVIEW verdict per finding. Restate NOT REACHABLE findings as NOT EXPLOITABLE (unreachable).
>
> Finding Context:
> [paste the full Finding Context from Phase 1 verbatim]
>
> Reachability Report:
> [paste the full Reachability Report from Phase 2 verbatim]

Wait for the Exploitability Verdicts before proceeding.

---

## Phase 4 — Remediation and reporting

Determine the output path for the triage report:
- Use the primary/highest-priority CVE ID, or the scan label, as the slug
- Today's date: use YYYY-MM-DD format
- Output directory: `vuln-management/cases/YYYY-MM-DD-[slug]/`
- Output file: `triage-report.md`
- Create the directory if it does not exist.

Spawn a VMRemediationPlanner agent (subagent_type: VMRemediationPlanner) with this prompt:

> Produce the prioritized, evidence-backed remediation plan from the three prior phase outputs, then save the report. Rank by real exploitability first (not raw CVSS), assign FIX NOW / SCHEDULE / ACCEPT per finding, give the concrete fix with breaking-change and transitive-dependency assessment, and assemble the audit trail.
>
> Finding Context:
> [paste the full Finding Context from Phase 1 verbatim]
>
> Reachability Report:
> [paste the full Reachability Report from Phase 2 verbatim]
>
> Exploitability Verdicts:
> [paste the full Exploitability Verdicts from Phase 3 verbatim]
>
> After producing the report in your response, save it to: [full output path]

Wait for the full report before proceeding.

---

## Phase 5 — Summary

Report to the user with this exact structure:

```
## Vulnerability Triage Complete

| Field | Value |
|---|---|
| Findings triaged | [N] |
| Genuinely exploitable | [n] (was [N] scanner findings — [X]% noise reduction) |
| Fix now | [n] |
| Schedule | [n] |
| Accepted (with evidence) | [n] |
| Needs human review | [n] |
| Report saved | [full path to triage-report.md] |
```

Then list the FIX NOW items (CVE, package, fix) and, if any are NEEDS REVIEW, what a human must check to close them.
