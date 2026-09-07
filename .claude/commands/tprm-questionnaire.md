Create a TPRM questionnaire for a prospective vendor: $ARGUMENTS

You are orchestrating vendor intake. Execute all phases in order. Phase 1 is interactive and must be done by you directly, in this conversation — do not delegate it to a subagent.

---

## Phase 1 — Intake

Ask the user for the following. If `$ARGUMENTS` already answers one of these clearly, confirm it back briefly instead of re-asking.

The `AskUserQuestion` tool caps each question at 4 options, so only use it for #5 below (2 options). For #3 and #4, which each have more than 4 possible values, ask as a plain free-text question in your own message — list the example categories in the question text and let the user name whichever apply (comma-separated is fine); do not pass them to `AskUserQuestion`.

1. **Company name** and a one-line description of what they do (free text)
2. **Purpose / use case** — what will we use them for? (free text)
3. **Connection/integration type** — ask in plain text, offering these as examples: API key, OAuth, SSO/SAML, SDK/embedded client, webhook (inbound), file transfer, none/informational only. More than one may apply.
4. **Data categories shared** — ask in plain text, offering these as examples: PII, payment data, PHI, credentials/secrets, proprietary/IP, AI prompts/model content, none/metadata-only. More than one may apply.
5. **Criticality** — use `AskUserQuestion` with exactly 2 options: hard dependency (product breaks without it) vs. replaceable/optional.

Do not proceed to Phase 2 until you have all five.

---

## Phase 2 — Folder setup

- Slugify the company name (lowercase, spaces/punctuation → hyphens) to get `<slug>`
- Target folder: `tprm/vendors/<slug>/`
- Create the folder if it doesn't already exist. If it already exists and already contains `analysis.md`/`questionnaire.md`, tell the user and ask whether to overwrite or pick a different slug before continuing.

---

## Phase 3 — Research and questionnaire generation

Spawn a TPRMQuestionnaireWriter agent (subagent_type: TPRMQuestionnaireWriter) with this prompt:

> Research this prospective vendor and produce the TPRM analysis and questionnaire.
>
> Company name: [from intake]
> Company description: [from intake]
> Purpose / use case: [from intake]
> Connection/integration type: [from intake]
> Data categories shared: [from intake]
> Criticality: [from intake]
> Target folder: tprm/vendors/[slug]/
>
> Write analysis.md and questionnaire.md into the target folder as described in your agent definition.

Wait for it to finish and report back the risk tier and file paths.

---

## Phase 4 — Summary

Report to the user with this structure:

```
## TPRM Questionnaire Ready

| Field | Value |
|---|---|
| Vendor | [Company name] |
| Risk tier | [Critical / High / Medium / Low] |
| Analysis | tprm/vendors/[slug]/analysis.md |
| Questionnaire | tprm/vendors/[slug]/questionnaire.md |
```

[One-paragraph summary from the writer agent's report.]

Next step: send `questionnaire.md` to the vendor. Once you have their completed answers and any supporting evidence, place them under `tprm/vendors/[slug]/vendor-response/` and run `/tprm-review [slug]`.
