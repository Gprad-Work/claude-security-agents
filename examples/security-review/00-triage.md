# Example: SecurityTriage on ClariNote PRD

> Agent: `SecurityTriage` · Input: [`../sample-system/PRD.md`](../sample-system/PRD.md) · Phase 1 of `/security-review`
> Illustrative output showing the dispatch decision. No findings — that's the domain agents' job.

---

## SECURITY TRIAGE

### System Summary
ClariNote is a multi-tenant B2B healthcare SaaS that ingests patient documents from outpatient clinics and uses the Anthropic Claude API to generate structured clinical summaries. It has a React web app and a React Native mobile app, a Node/Express API and summarization workers on AWS EKS, Postgres (RDS) and S3 for storage, and Pinecone as a vector store for retrieval-augmented context. It processes PHI for US and EU customers and integrates seven third-party vendors. Auth is email/password + optional Google SSO, with HS256 JWTs carrying `clinic_id` and `role`.

### Threat Model
The crown jewels are PHI (patient documents, demographics, clinical summaries) across many clinic tenants sharing one database. The most valuable attacker outcomes are cross-tenant PHI theft, bulk exfiltration of the full patient corpus, and abuse of the Claude API key. Healthcare SaaS is a top ransomware and data-extortion target, and the design concentrates risk: application-layer-only tenant isolation, sequential patient IDs, an LLM fed untrusted document text, a root container mounting the Docker socket, and vendors integrated with no review. Realistic attackers range from opportunistic ransomware affiliates to an authenticated user in one clinic pivoting to another clinic's records.

### Selected Domain Agents

**AISecurity**
- Surface area: Claude API summarization fed untrusted uploaded document text, plus a RAG step pulling prior summaries into context.
- Key questions:
  1. Uploaded documents are attacker-controllable and passed directly into the summarization prompt — what stops injected instructions in a document from steering the model or exfiltrating retrieved context?
  2. The RAG step retrieves "the patient's 3 most recent prior summaries" — is that retrieval scoped to the requesting tenant, or can it cross `clinic_id`?
  3. Is there any output handling on the generated summary before it's stored and rendered?

**ProductAppSecurity**
- Surface area: `GET /api/patients/{patient_id}` with sequential integer IDs; `PATCH /api/summaries/{id}` accepting the full client object.
- Key questions:
  1. Is object-level authorization enforced on `/api/patients/{id}` and `/api/summaries/{id}`, or only tenant-scoping in some queries?
  2. Sequential patient IDs + any missing check = enumeration of all patients — is there a per-object owner check?
  3. `PATCH` saves the full summary object from the client — can a caller set fields like `signed_by`, `clinic_id`, or `status` via mass assignment?

**GRCSecurity**
- Surface area: PHI, US+EU users, HIPAA and GDPR both in scope, seven vendors, soft-delete-only account deletion.
- Key questions:
  1. Are BAAs in place with every vendor that receives PHI (Anthropic, AWS, Twilio, Sentry at minimum)?
  2. Soft delete (`deleted_at`) does not satisfy GDPR erasure or HIPAA disposal — what is the hard-delete/anonymization path, and does it reach backups?
  3. What is the EU data-residency story given a single AWS account and vendors with US subprocessors?

**DataSecurity**
- Surface area: PHI in Postgres/S3, full request bodies logged on error, staging refreshed from prod, backups on the same KMS key.
- Key questions:
  1. Full request bodies (containing PHI) logged to CloudWatch and Sentry — what controls access to those pipelines?
  2. Staging refreshed from a production dump — is PHI masked, or is prod PHI sitting in a lower-trust environment?
  3. Is any field-level encryption applied, or is protection limited to disk/TLS?

**TPRMSecurity**
- Surface area: seven vendors, no onboarding review, long-lived keys, a Google OAuth scope requesting `drive.readonly`.
- Key questions:
  1. Why does SSO request `drive.readonly`? What's the blast radius of that grant if the token or ClariNote is compromised?
  2. What does each vendor's compromise yield — especially Anthropic (document text), Twilio (patient contact), Sentry/Segment (request context)?
  3. Is there any offboarding path for long-lived vendor keys?

**ContainerSecurity**
- Surface area: root container mounting the Docker socket, `latest` tags, public-read ECR, no admission controller, one namespace.
- Key questions:
  1. The API container runs as root and mounts the node's Docker socket — this is node compromise on any RCE. Is it truly required?
  2. Public-read ECR and `latest` tags — what stops image tampering or an unintended-public image?
  3. Workers share the API image with elevated S3/vector permissions — what's the blast radius of a worker compromise?

**CloudSecurity**
- Surface area: S3 upload bucket, RDS, IAM for workers with "elevated permissions," CloudTrail in one region only.
- Key questions:
  1. Is `s3://clarinote-uploads` private with least-privilege access, and is object-level ownership enforced beyond the path convention?
  2. What exactly are the workers' "elevated permissions" — wildcard IAM?
  3. CloudTrail in the primary region only leaves other regions blind — is that a gap for forensics?

**PlatformSecurity**
- Surface area: HS256 JWT with a shared secret across services, `role`/`clinic_id` claims trusted downstream, Google SSO, EKS RBAC.
- Key questions:
  1. HS256 with one shared secret across all services means any service (or a leak from any) can mint valid tokens for any role — is asymmetric signing considered?
  2. Downstream services trust `role: admin` from the JWT without re-verification — where is authorization actually enforced?
  3. What is the EKS RBAC and service-account posture given one namespace for everything?

**SecOps**
- Surface area: no alerting on PHI access volume or failed-authz spikes; logs to CloudWatch/Sentry.
- Key questions:
  1. With sequential IDs and thin authz, bulk patient enumeration is a top risk — is there any detection for anomalous PHI access volume?
  2. Are failed-authorization spikes alerted, or would enumeration be silent?
  3. Is there forensic-grade audit logging of PHI access (who read which patient, when)?

**ThreatIntel**
- Surface area: healthcare sector, internet-facing, named stack (EKS, Node, Claude API) to map to real TTPs.
- Key questions:
  1. Which active healthcare-targeting campaigns and TTPs map onto this design's weakest points?
  2. Are any named components KEV-exposed or approaching EOL (e.g., node:18 base)?
  3. Is the Claude API key exposure (in the Dockerfile) on a known "LLMjacking" abuse path?

**RedTeam**
- Surface area: enough cross-domain weakness (IDOR + injection + root container + shared JWT secret + flat network) to chain.
- Key questions:
  1. Can a single authenticated front-desk user in one clinic reach another clinic's full PHI, and by what chain?
  2. What does one popped worker container yield given Docker-socket mount + elevated IAM + flat namespace?
  3. Assume one clinician's session is phished — how far does it reach?

**MobileSecurity**
- Surface area: React Native clinician app that uploads and displays PHI on personal/mobile devices.
- Key questions:
  1. Is PHI cached or stored locally on-device, and is it encrypted / excluded from OS backups?
  2. Is certificate pinning implemented for the API connection?
  3. Are there deep links that could expose `patient_id`/`summary_id` to other apps?

**DevSecOps**
- Surface area: GitHub Actions image build, `ANTHROPIC_API_KEY` baked into the Dockerfile, public ECR, `latest` tags.
- Key questions:
  1. The Anthropic key is hard-coded in the Dockerfile — it's in every image layer and the public registry. Confirm and scope the blast radius.
  2. Are GitHub Actions pinned to commit SHAs, and are build secrets handled via OIDC or baked in?
  3. Is any SCA/image scanning gating the pipeline?

**APISecurity**
- Surface area: object-referencing endpoints with sequential IDs, a cost-bearing upload→LLM path, no versioning/inventory.
- Key questions:
  1. Is BOLA (object-level authz) enforced per endpoint at the data layer across the whole surface?
  2. Are cost-bearing endpoints rate-limited and quota'd per tenant?
  3. Is there an API inventory/versioning story, or shadow/staging endpoints reachable in prod?

**PrivacyEngineering**
- Surface area: special-category health data → LLM + embeddings, soft-delete-only erasure, prod-cloned staging, analytics URLs.
- Key questions:
  1. Does the erasure *mechanism* reach S3, Pinecone, logs, and backups — or only the clinic row?
  2. Is PHI use limited to the clinical purpose, and are embeddings classified/protected as PHI?
  3. Is "realistic test data" actually de-identified, or raw production PHI?

**FraudAbuse**
- Surface area: B2B contracted model (limited consumer-abuse surface), but high-value clinician accounts with no MFA and an unmetered LLM pipeline.
- Key questions:
  1. What is a taken-over clinician account worth, and what raises the attacker's cost to obtain one?
  2. Can an authorized/compromised account run up Anthropic spend via unmetered summarization?
  3. Is there any velocity/breadth control on abnormal-scale record access?

### Agents Not Called

| Agent | Reason |
|---|---|
| `InfraSecurity` | TLS/cert and host-hardening detail is thin in the PRD; the sharpest infra-adjacent risks (secrets at rest, KMS key reuse) are covered by DataSecurity and CloudSecurity here. Call it once infra configs exist. |
| `NetworkSecurity` | The PRD notes "one namespace, flat" but has no VPC/subnet/firewall detail to review yet; ContainerSecurity and RedTeam cover the flat-network blast radius at the level the artifact supports. Revisit at the infra-design stage. |

### Lead Hypotheses
1. There is at least one CRITICAL cross-tenant PHI path: sequential patient IDs plus application-only tenant isolation will produce an IDOR, and the RAG retrieval may cross tenants independently.
2. The Anthropic API key in the Dockerfile pushed to a public-read ECR is a live secret exposure — likely the single most urgent finding.
3. The root-plus-Docker-socket container turns any RCE (including one reached via prompt injection → worker) into node/cluster compromise; RedTeam will chain these into one path.
