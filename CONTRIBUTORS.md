# Contributors

Thank you for being here. This project gets better every time someone brings their security experience to it — whether that's a sharper detection rule, a domain agent that covers a gap, or a doc fix that makes things clearer for the next person.

Every contribution counts, no matter the size.

---

## Ways to contribute

### Add a new security agent
Know a domain that isn't covered? Write an agent for it. Good candidates:
- Data security / DLP
- Container security (beyond what InfraSecurity covers)
- Threat intelligence
- Red team / offensive security

Follow the frontmatter format in any existing agent under `.claude/agents/` and open a PR.

### Improve an existing agent
Found a check that's too generic, missing a key attack pattern, or wrong for a specific stack? Edit the agent and explain what you changed and why in the PR description.

### Add a detection rule
Rules live in `detection-rules/rules/` (subdirectories by category). Use the `/rule-write` command or write one from scratch following the Sigma format described in `docs/detection-engineering.md`. The `/rule-review` command will validate it before you open the PR.

### Improve the docs
Spotted something unclear in `docs/`? Fixed a broken example? Go for it — docs PRs are always welcome.

### Report a bug or gap
Open an issue if you find:
- An agent that gives consistently bad advice on a real-world pattern
- A detection rule with a logic error or high false-positive rate
- A gap in coverage you don't have time to fill yourself (so someone else can pick it up)

---

## Getting started

```bash
git clone https://github.com/Gprad-Work/claude-security-agents.git
cd claude-security-agents
```

Open the repo as a project in Claude Code. All agents and commands are immediately available — no install step needed.

Test your changes by running the relevant slash command locally before opening a PR.

---

## Guidelines

- **No real credentials in examples** — use `your-token`, `<YOUR_KEY>`, or `...` as placeholders
- **Agent prompts should be specific, not generic** — "check for SSRF" is weak; "check whether the URL-fetching endpoint validates the scheme and blocks RFC 1918 ranges" is strong
- **Detection rules must pass the efficacy check** — run `/rule-review` on your rule before submitting
- **One logical change per PR** — a new agent, an improved agent, a new rule. Mix of unrelated changes makes review harder

---

## Recognition

All merged contributors are listed below. If you've contributed and aren't listed, open a PR to add yourself.

| Contributor | Contribution |
|---|---|
| [Geet Pradhan](https://github.com/Gprad-Work) | Initial agent library |

---

If you're unsure whether an idea fits, open an issue first and ask. The worst that can happen is a conversation.
