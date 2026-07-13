# Example: DevSecOps on ClariNote PRD

> Agent: `DevSecOps` (Sonnet) · Input: [`../sample-system/PRD.md`](../sample-system/PRD.md)
> Illustrative domain output.

---

## DevSecOps Review

### Critical findings
| # | Area | Finding | Scenario | Fix |
|---|---|---|---|---|
| DS-001 | Secret in image + public registry | The Dockerfile bakes `ENV ANTHROPIC_API_KEY=sk-ant-...` (§4.2), and images are pushed to a **public-read** ECR repo (§4.1). The key is in a layer, in a public registry. This is a live, internet-exposed secret. | Anyone pulls the public image, runs `docker history`/`dive`, extracts the Anthropic key, and bills API usage to ClariNote ("LLMjacking"). The key also grants access to any data reachable via that account. | Remove the secret from the Dockerfile immediately, rotate the key, inject it at runtime from a secrets manager, make ECR private, and scan history to confirm no other secrets leaked. |

### High findings
| # | Area | Finding | Scenario | Fix |
|---|---|---|---|---|
| DS-002 | Registry exposure | ECR is public-read "to simplify partner pulls," with `latest` tags (§4.1). Public images leak internal build details and dependencies; mutable `latest` allows silent swaps. | An attacker studies the public image for vulnerable dependencies and internal structure; a tag push replaces `latest` with a tampered image that deploys unnoticed. | Make ECR private with scoped cross-account pull for the partner only. Pin deployments to immutable digests; enable tag immutability. |
| DS-003 | CI/CD supply chain | GitHub Actions builds the image (§4.1) with no mention of action pinning, OIDC-based cloud auth, or least-privilege. Baked-in secrets (DS-001) suggest secrets flow through the build insecurely. | A compromised or updated third-party action (floating `@v3` tag) exfiltrates build secrets or injects into the image. | Pin all actions to commit SHAs; use GitHub OIDC → short-lived AWS role instead of long-lived keys; minimize `GITHUB_TOKEN` and job permissions. |
| DS-004 | No SCA / image scanning gate | No dependency scanning, image vulnerability scanning, or SBOM in the pipeline. Base is `node:18` (§4.2), pinned only to a major version. | Known-vulnerable dependencies or base-image CVEs ship to production unnoticed; "are we affected by CVE-X?" is unanswerable. | Add SCA (npm audit / Dependabot) and image scanning (Trivy/Grype) as gating steps; generate an SBOM per build; pin and regularly rebuild the base image. |

### Medium / Low findings
| # | Area | Finding | Scenario | Fix |
|---|---|---|---|---|
| DS-005 | Environment parity / prod data in CI-adjacent envs | Staging is refreshed weekly from a production DB dump (§7) — production PHI in a lower-trust environment often wired to CI. | Weaker staging controls expose real PHI (overlaps DataSecurity D-003). | Use masked/synthetic data for staging; never copy raw PHI into non-prod. |
| DS-006 | Branch protection / review gates | No mention of required reviews, signed commits, or protected branches for the repo that builds production images. | A single compromised developer account ships arbitrary code to prod. | Require PR review + status checks on the default branch; consider signed commits and CODEOWNERS. |

### What's done well
- Builds are automated through GitHub Actions and images are centralized — a solid foundation once secrets are externalized and the registry is locked down.

### Verdict
**BLOCK** — DS-001 is a live secret exposure: a production Anthropic API key sits in a public container registry and must be rotated and removed today, independent of the rest of the review. The surrounding supply-chain gaps (public registry, unpinned actions, no scanning) mean the pipeline cannot currently be trusted to ship a clean image.
