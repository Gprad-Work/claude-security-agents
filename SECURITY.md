# Security Policy

## Reporting a Vulnerability

If you find a security issue in this repository — including prompt injection risks in agent definitions, logic flaws in detection rules, or anything else — please **do not open a public GitHub issue**.

Instead, use **GitHub private vulnerability reporting** — click the "Report a vulnerability" button on the [Security tab](../../security/advisories/new) of this repo.

Please include:
- A description of the issue
- Steps to reproduce or a proof-of-concept
- The potential impact

You can expect an acknowledgement within 48 hours and a resolution or status update within 7 days.

## Scope

This repository contains Claude Code agent definitions and slash commands (Markdown files). There is no deployed service, API, or running infrastructure associated with this repo.

Security issues in scope:
- Agent prompts that could be exploited via prompt injection to cause harmful outputs
- Detection rules with logic errors that would cause systematic false negatives on real attacks
- Documentation examples that contain or encourage insecure patterns

Out of scope:
- Issues in Claude itself — report those to [Anthropic](https://www.anthropic.com/security)
- Issues in Claude Code — report those via the [Claude Code issue tracker](https://github.com/anthropics/claude-code/issues)

## Using These Agents Safely

- Never paste real credentials, API keys, or PII into agent prompts — even as examples
- Agent outputs are AI-generated and should be reviewed by a human before acting on them
- Detection rules produced by these agents should be validated in a test environment before production deployment
