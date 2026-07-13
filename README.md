# Claude Security Agents

A collection of Claude Code agents and slash commands for security engineering — spec-phase threat modeling, detection rule authoring, and incident response triage.

Built for use with [Claude Code](https://claude.ai/code). Drop the `.claude/` directory into any project and the agents and commands are immediately available.

---

## What's included

### Slash commands

| Command | What it does |
|---|---|
| `/security-review <path>` | Full multi-agent security review of a spec, PRD, ERD, or codebase. Runs 2–10 specialist agents in parallel, synthesises findings, issues a gate decision. |
| `/triage <alert>` | Incident response triage. Parses an alert, queries the SIEM (if MCP is configured), and returns a TP/FP verdict with evidence in ~45 seconds. |
| `/rule-write <scenario>` | Writes a production-ready Sigma detection rule for a described threat scenario, then runs automated peer review. |
| `/rule-review <rule-id or path>` | Peer reviews one or more Sigma rules — efficacy scoring, logic analysis, FP coverage, MITRE accuracy. |
| `/coverage-scan` | Scans the detection rule library against the MITRE ATT&CK framework, surfaces coverage gaps, and recommends the highest-priority missing rules. |

### Agents

**Security review pipeline**
- `SecurityTriage` — reads artifacts, builds a threat model, selects which domain agents to dispatch
- `SecurityLead` — synthesises domain findings into a unified risk-ranked report with a gate decision
- `AISecurity` — prompt injection, indirect injection, agentic attack chains, OWASP LLM Top 10
- `ProductAppSecurity` — OWASP Top 10, IDOR, injection, session management, business logic
- `GRCSecurity` — GDPR, SOC2, HIPAA, CCPA, data retention, audit trails
- `SecOps` — logging coverage, alerting, IR readiness, forensic posture
- `DevSecOps` — CI/CD security, secrets management, dependency scanning, supply chain
- `CloudSecurity` — IAM, storage ACLs, managed service posture, cloud misconfiguration
- `InfraSecurity` — TLS, secrets at rest, server hardening, cloud storage
- `NetworkSecurity` — VPC design, firewall rules, segmentation, egress filtering
- `PlatformSecurity` — OAuth/OIDC, Kubernetes RBAC, service mesh, API gateways
- `MobileSecurity` — iOS/Android, certificate pinning, local storage, deep links
- `ContainerSecurity` — Dockerfile/image hygiene, registry security, runtime hardening, Pod Security Standards, escape surface
- `DataSecurity` — data classification, field-level encryption, key management, exfiltration/leakage paths, backups, non-prod data
- `TPRMSecurity` — vendor compromise blast radius, OAuth grant scopes, subprocessor risk, vendor lifecycle, concentration risk
- `ThreatIntel` — adversary TTP mapping to your stack, ATT&CK/KEV exposure, phishing and impersonation surface, intel operationalization
- `RedTeam` — chains cross-domain weaknesses into end-to-end attack paths, assume-breach and blast-radius analysis (authorized review)

**Detection engineering pipeline**
- `DEDetectionRuleWriter` — writes schema-compliant Sigma YAML with MITRE tags and filter blocks
- `DEReviewRule` — peer reviews rules: efficacy scoring + qualitative judgment
- `DECoverageScanner` — maps the rule library to ATT&CK tactics, prioritises gaps

**Incident response pipeline**
- `IRAlertParser` — parses alert input, extracts entities, scopes investigation timeframe
- `IRSIEMInvestigator` — queries SIEM via MCP, pivots on entities, builds event timeline
- `IRAnalyst` — produces TP/FP/TNP verdict with MITRE kill chain and containment actions

---

## Setup

### 1. Install

Clone this repo or copy the `.claude/` directory into your project root:

```bash
git clone git@github.com:<your-username>/claude-security-agents.git
cp -r claude-security-agents/.claude /your-project/
```

### 2. Verify

Open Claude Code in your project. Type `/` — you should see `security-review`, `triage`, `rule-write`, `rule-review`, and `coverage-scan` in the command list.

### 3. Configure for your stack

A few agents reference your stack for context. Update these before use:

- **`/security-review`** — works on any artifact out of the box
- **`/coverage-scan`** — update the stack line in `.claude/commands/coverage-scan.md` with your services (e.g., Okta, AWS, Kubernetes)
- **`/rule-review`** — update the stack-specific section in `.claude/agents/detection-engineering/de-rule-reviewer.md` with your log source field names
- **`/rule-write`** — update `author: <your-name>` in `.claude/agents/detection-engineering/de-rule-writer.md`

### 4. SIEM MCP (optional — for `/triage`)

`/triage` works without a SIEM but produces richer results when agents can query related events directly. Add your SIEM's MCP server to `.claude/settings.json`:

```json
{
  "mcpServers": {
    "splunk": {
      "command": "uvx",
      "args": ["splunk-mcp-server"],
      "env": { "SPLUNK_TOKEN": "..." }
    }
  }
}
```

See [`docs/incident-response.md`](docs/incident-response.md) for supported SIEMs and configuration examples.

---

## Model assignments

| Agent | Model | Why |
|---|---|---|
| `SecurityLead` | Opus | Meta-reasoning, agent dispatch, synthesis — wrong dispatch = missed vulnerability class |
| `AISecurity` | Opus | Adversarial imagination — prompt injection requires attacker mindset, not pattern matching |
| `RedTeam` | Opus | Cross-domain attack-path synthesis — chaining weaknesses into kill chains is adversarial reasoning, not a checklist |
| All other domain agents | Sonnet | Structured checklist application — bounded answer space, near-Opus quality at 5× lower cost |
| `DEReviewRule` | Opus | Operational judgment — only a senior engineer knows if a rule will be tuned out at 3am |
| `DEDetectionRuleWriter` | Sonnet | Structured Sigma YAML generation — fixed schema, pattern matching |
| `DECoverageScanner` | Sonnet | ATT&CK gap analysis — systematic, not adversarial |
| `IRAnalyst` | Opus | TP/FP verdict under uncertainty — wrong call means a real attack missed |
| `IRAlertParser` | Sonnet | Structured extraction — known schema, repeatable |
| `IRSIEMInvestigator` | Sonnet | SIEM queries and pivots — systematic, not adversarial |

See [`docs/model-selection.md`](docs/model-selection.md) for the full rationale.

---

## Examples

See [`examples/`](examples/) for live, worked output from **every agent** in this library — all running against one shared sample system (a fictional healthcare SaaS, [`examples/sample-system/PRD.md`](examples/sample-system/PRD.md)) so you can see how the domains overlap, chain, and get synthesised into a final gate decision. Start with [`examples/README.md`](examples/README.md).

---

## Documentation

- [`docs/security-review.md`](docs/security-review.md) — How `/security-review` works, what artifacts it accepts, example output
- [`docs/detection-engineering.md`](docs/detection-engineering.md) — The full rule write → validate → review → coverage pipeline
- [`docs/incident-response.md`](docs/incident-response.md) — How `/triage` works, SIEM MCP setup, example triage output
- [`docs/model-selection.md`](docs/model-selection.md) — Why each agent is on Opus vs. Sonnet, cost analysis

---

## Requirements

- [Claude Code](https://claude.ai/code) — Claude Code CLI or desktop app
- Anthropic API access (Claude Sonnet + Opus)
- For `/triage` with SIEM enrichment: a SIEM with an MCP server (Splunk, Elastic, etc.)
- For detection engineering: Python 3.x with `detection-rules/validators/` scripts (see [`docs/detection-engineering.md`](docs/detection-engineering.md))
