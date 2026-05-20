# Incident Response — /triage

Multi-agent alert triage. Pass in an alert and get a TP/FP verdict with evidence in ~45 seconds.

---

## Usage

```
/triage rule-AUTH-001 actor=suspicious-user@example.com ts=2026-05-20T14:37Z
/triage "GitHub default branch updated by service account gp-infra-bot at 14:37 UTC, correlated with branch protection removal (rule-REPO-002)"
/triage INCIDENT-2026-0518-001
```

`$ARGUMENTS` can be:
- A rule ID (agent will find the rule YAML in `detection-rules/rules/`)
- Free-text alert description
- An incident ID (agent will look up context)

---

## How it works

Three agents run in sequence, each feeding structured output to the next.

**Phase 1 — IRAlertParser (Sonnet)**
Reads the raw alert. If a rule ID is referenced, finds and reads the rule YAML. Extracts:
- Rule metadata (title, description, level, MITRE techniques)
- Key entities: actor, IP, resource, timestamp
- Investigation timeframe (± window around the event)
- Initial investigation questions

**Phase 2 — IRSIEMInvestigator (Sonnet)**
Receives the structured Alert Context from Phase 1. If a SIEM MCP server is configured:
- Queries for related alerts in the investigation timeframe
- Pivots on extracted entities (same actor, same IP, matching field values)
- Reconstructs a timestamped event timeline
- Tests false positive conditions from the rule's FP list

Without a SIEM MCP, this phase produces a partial timeline from alert context only.

**Phase 3 — IRAnalyst (Opus)**
Receives Alert Context + Event Timeline. Produces:
- Verdict: `TRUE_POSITIVE` | `FALSE_POSITIVE` | `TRUE_NEGATIVE (PENDING)` | `NEEDS_INVESTIGATION`
- Confidence: `HIGH` | `MEDIUM` | `LOW`
- Severity (if TP): `CRITICAL` | `HIGH` | `MEDIUM` | `LOW`
- MITRE kill chain position and technique IDs
- Cited evidence (specific events from the timeline)
- Containment actions (if TP)
- What data is missing (if NEEDS_INVESTIGATION)

The report is saved to `detection-rules/cases/YYYY-MM-DD-[rule-id]/triage-report.md`.

---

## SIEM MCP setup

`/triage` works without a SIEM but the event timeline will only contain what's in the alert. With a SIEM MCP server, Phase 2 can pull related raw events.

Add your SIEM to `.claude/settings.json` in your project root:

**Splunk**
```json
{
  "mcpServers": {
    "splunk": {
      "command": "uvx",
      "args": ["splunk-mcp-server"],
      "env": {
        "SPLUNK_URL": "https://your-splunk-instance:8089",
        "SPLUNK_TOKEN": "your-token"
      }
    }
  }
}
```

**Elastic / OpenSearch**
```json
{
  "mcpServers": {
    "elastic": {
      "command": "uvx",
      "args": ["elastic-mcp-server"],
      "env": {
        "ELASTIC_URL": "https://your-cluster",
        "ELASTIC_API_KEY": "your-key"
      }
    }
  }
}
```

**GitHub (for repository event investigation)**
```json
{
  "mcpServers": {
    "github-events": {
      "command": "npx",
      "args": ["@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "your-token" }
    }
  }
}
```

Multiple MCP servers can be configured simultaneously. The SIEM investigator will use whichever are available.

---

## Example output

```
## Triage Complete

| Field    | Value |
|---|---|
| Rule     | REPO-001 — GitHub Default Branch Updated |
| Verdict  | FALSE POSITIVE |
| Confidence | HIGH |
| Severity | N/A |
| Kill chain | N/A |
| Affected users | gp-infra-bot (service account) |
| Report saved | detection-rules/cases/2026-05-20-REPO-001/triage-report.md |
```

The full report contains the reconstructed event timeline, behavioural analysis, and the specific reasoning for the verdict — all citations drawn from the alert and SIEM data.

---

## Case directory structure

Each triage run creates:

```
detection-rules/cases/
└── YYYY-MM-DD-[rule-id]/
    └── triage-report.md
```

The report is the durable record. Close the alert in your ticketing system with a link to this file.
