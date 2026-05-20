Scan the detection rule library for ATT&CK coverage gaps and produce prioritised rule recommendations.

---

## Phase 1 — Coverage Scan

Spawn a DECoverageScanner agent (subagent_type: DECoverageScanner) with this prompt:

> Run a full ATT&CK coverage scan on the detection rule library.
>
> 1. Run: cd detection-rules && python validators/coverage_report.py rules
> 2. Read all existing rule files to verify MITRE tag accuracy
> 3. Prioritise gaps by relevance to the stack described in this repository's CLAUDE.md or README (customize this line with your own stack — e.g. Okta, AWS, Kubernetes, Postgres)
> 4. Write rule recommendations for every HIGH priority gap (enough detail for the rule writer to act immediately)
> 5. Save the coverage brief to: detection-rules/coverage-report-[today's date].md
>
> Return the full brief.

Wait for the scanner to complete.

---

## Report to the user

After the scan completes:

1. Show the Coverage Snapshot table and Tactic Coverage Table
2. List the HIGH priority rule recommendations
3. Ask: **"Should I write the highest-priority missing rules now?"**
   - If yes: run `/rule-write` for each HIGH priority recommendation, in order of priority
   - If no: confirm the full brief was saved and show the path
