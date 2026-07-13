# Example: ContainerSecurity on ClariNote PRD

> Agent: `ContainerSecurity` (Sonnet) · Input: [`../sample-system/PRD.md`](../sample-system/PRD.md)
> Illustrative domain output — one of the newly added agents.

---

## Container Security Review

### Workload inventory
| Workload / Image | Base image | Runs as | Notable runtime settings | Escape-relevant exposure |
|---|---|---|---|---|
| API (§4.2) | `node:18` (major tag only) | **root** | Mounts node **Docker socket**; no seccomp/caps described | Docker-socket mount = full node compromise on any RCE |
| Summarization workers | Same image as API | root (inherited) | Elevated S3 + vector IAM | Worker RCE (reachable via prompt injection) → node + all-tenant S3 |
| Cluster | Single EKS, one namespace, no admission controller | — | Developers set own pod specs | No guardrail prevents any of the above |

### Critical findings
| # | Workload | Finding | Escape / compromise scenario | Fix |
|---|---|---|---|---|
| CN-001 | API container (§4.1) | The API container **runs as root and mounts the node's Docker socket** to build preview containers. Mounting `/var/run/docker.sock` grants control of the container runtime — equivalent to root on the node. | Any RCE in the API (or a worker sharing the image) uses the Docker socket to launch a privileged container mounting the host filesystem, escaping to the node, then to other pods and cluster credentials. This converts an app-level bug into full cluster compromise. | Remove the Docker-socket mount. Build preview containers out-of-band via a dedicated, isolated builder (BuildKit rootless, or a separate hardened build service) — never from the request-serving container. Run as a non-root `USER`. |
| CN-002 | Workers (§4) | Workers share the API image and run with elevated S3/vector IAM as root. The summarization worker processes attacker-controllable document text (see AISecurity AI-001), making it the most likely RCE target — with the most dangerous permissions. | Prompt-injection or a parsing/OCR exploit yields code execution in a root worker holding broad S3 access on a node with a mounted Docker socket: read all tenants' documents, then escape the node. | Separate the worker image and identity from the API; drop to non-root; scope IAM to least privilege (see CloudSecurity C-001); no Docker socket. |

### High findings
| # | Workload | Finding | Escape / compromise scenario | Fix |
|---|---|---|---|---|
| CN-003 | Image build (§4.2) | Dockerfile runs as root, no `USER`, `COPY . .` (ships source, `.git`, possibly secrets), `npm install` (not `ci`, unpinned), and a hard-coded secret (see DevSecOps DS-001). Base pinned only to `node:18`. | Fat, root, unpinned image with a baked secret; layer history leaks the key and source. | Multi-stage build, non-root `USER`, `.dockerignore`, `npm ci` against a lockfile, distroless/slim runtime base pinned by digest, secret injected at runtime. |
| CN-004 | Cluster policy (§4.1) | No admission controller; developers set their own pod specs; single namespace. Nothing enforces non-root, dropped capabilities, no host mounts, or signed images. | A single insecure pod spec (privileged, hostPath, automounted token) ships unchecked; blast radius is the whole namespace. | Enforce Pod Security Standards `restricted` and an admission policy (Kyverno/Gatekeeper) rejecting privileged pods, host mounts, and unsigned images. Namespace per trust boundary. |

### Medium / Low findings
| # | Workload | Finding | Escape / compromise scenario | Fix |
|---|---|---|---|---|
| CN-005 | Runtime hardening | No `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`, seccomp `RuntimeDefault`, or dropped capabilities described. | Standard container-hardening baseline is absent, easing persistence and kernel-exploit paths. | Apply a restricted securityContext to all workloads. |
| CN-006 | Image provenance | Public-read ECR, `latest` tags, no signing/scanning (overlaps DevSecOps DS-002/004). | Tampered or vulnerable images deploy silently. | Private registry, digest pinning, cosign signing verified at admission, scan gate. |

### What's done well
- Running on EKS with a container build pipeline means all of the above are enforceable centrally (securityContext defaults + admission policy) rather than per-team — the platform supports a strong fix.

### Verdict
**BLOCK** — CN-001/CN-002 are a container-escape path directly downstream of the AI attack surface: the worker that ingests untrusted document text runs as root with broad cloud permissions on a node whose Docker socket it can reach. Any code execution becomes cluster compromise and all-tenant PHI exposure. Remove the Docker-socket mount, split and de-privilege the worker, and add admission enforcement before production.
