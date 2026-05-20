Run a full multi-agent security review on the docs or artifact(s) at: $ARGUMENTS

You are orchestrating a real security review. Follow these four phases exactly. Do not collapse or skip any phase.

---

## Phase 1 — Triage

Spawn a SecurityTriage agent (subagent_type: SecurityTriage) with this prompt:

> Read all of the following artifacts, then produce your structured TRIAGE OUTPUT as described in your agent definition.
>
> Artifacts to review: $ARGUMENTS
>
> Read every file before forming any conclusions. Do not skip any.

Wait for the triage to return. Read its output carefully — it contains:
- System summary and threat model
- Selected domain agents and their key questions
- Agents not called and why
- Lead Hypotheses

Extract the selected domain agents and their key questions. You will use these in Phase 2.

---

## Phase 2 — Parallel Domain Agent Dispatch

In a single message, spawn ALL selected domain agents simultaneously using the Agent tool with run_in_background: true. Do not spawn them one at a time.

For each selected domain agent, use this prompt template:

> You are conducting a domain security review as part of a larger multi-agent security review orchestrated by a Security Lead.
>
> **System context:** [paste the triage threat model paragraph verbatim]
>
> **Artifacts to review:** $ARGUMENTS
>
> Read all provided artifacts. Then produce your full domain security review following your agent definition's output format.
>
> **Priority questions from the Security Lead (answer these explicitly in your findings):**
> [paste the key questions the triage assigned to this domain agent]
>
> Be thorough. Every finding must cite the specific document and section. Propose concrete fixes.

Map triage agent names to subagent_type values:
- AISecurity → AISecurity
- ProductAppSecurity → ProductAppSecurity
- GRCSecurity → GRCSecurity
- SecOps → SecOps
- DevSecOps → DevSecOps
- CloudSecurity → CloudSecurity
- InfraSecurity → InfraSecurity
- NetworkSecurity → NetworkSecurity
- PlatformSecurity → PlatformSecurity
- MobileSecurity → MobileSecurity

Wait for all domain agents to complete before proceeding to Phase 3.

---

## Phase 3 — Collect and Organise Domain Outputs

Collect all domain agent outputs. Organise them clearly — one section per domain agent, labelled with the domain name and finding count.

If any domain agent failed or timed out, note it and proceed with the outputs you have. Do not re-run failed agents — note the gap in the final report.

---

## Phase 4 — SecurityLead Synthesis

Determine the output path for the final report:
- If $ARGUMENTS is a directory, save to: $ARGUMENTS/../security-reviews/[YYYY-MM-DD]-run/security-lead-report.md
- If $ARGUMENTS is one or more files, save to a `security-reviews/[YYYY-MM-DD]-run/` directory adjacent to the first file's parent

Create the output directory if it doesn't exist.

Spawn a SecurityLead agent (subagent_type: SecurityLead) with this prompt:

> You are in Synthesis mode.
>
> **Artifacts reviewed:** $ARGUMENTS
>
> **Triage output (Phase 1 — threat model, agent selection, lead hypotheses):**
> [paste full triage output verbatim]
>
> **Domain agent reports:**
>
> ### [Domain name]
> [paste domain agent's full output]
>
> ### [Domain name]
> [paste domain agent's full output]
>
> [repeat for each domain agent]
>
> Run your Lead Review (Steps A–F from your agent definition) before writing anything.
>
> Then write the final unified security report and save it to: [output path]

---

## After completion

Report to the user:
- Which domain agents ran and their models
- Total finding count by severity
- Gate decision
- Path to the saved report
