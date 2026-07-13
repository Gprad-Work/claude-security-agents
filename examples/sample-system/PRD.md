# PRD: ClariNote — Clinical Document Summarization

> **Status:** Draft for security review
> **Author:** Platform Team
> **Audience:** Engineering, Security, Compliance

> ⚠️ This is a **fictional sample artifact** used to demonstrate the security agents in this repo. It intentionally contains realistic security gaps. Do not use it as a reference design.

---

## 1. Overview

ClariNote is a B2B SaaS product for outpatient clinics. Clinic staff upload patient documents (referral letters, lab PDFs, prior visit notes), and ClariNote uses an LLM to produce a structured clinical summary that clinicians review and sign off. The goal is to cut chart-prep time before appointments.

Customers are clinics (tenants). Each clinic has clinicians, front-desk staff, and one clinic admin. ClariNote handles **Protected Health Information (PHI)** and serves customers in the US and EU.

---

## 2. Users and tenancy

- **Clinician** — uploads documents, reviews and signs summaries, views patient history
- **Front-desk** — uploads documents, cannot view signed summaries
- **Clinic admin** — manages users within their clinic, views billing
- **ClariNote support** — internal staff who can access clinic data to troubleshoot

Multi-tenant: one shared Postgres database, tenant rows keyed by `clinic_id`. Isolation is enforced in the application service layer by adding `WHERE clinic_id = ?` to queries.

---

## 3. Core flows

### 3.1 Document upload
1. Staff upload a PDF/image via the web or mobile app.
2. File is stored in S3 at `s3://clarinote-uploads/{clinic_id}/{patient_id}/{doc_id}`.
3. A summarization job is queued.

### 3.2 Summarization
1. Worker pulls the document, extracts text (OCR for images).
2. Worker calls the Anthropic Claude API with a prompt:
   `"You are a clinical scribe. Summarize the following patient document into SOAP format. Document: {extracted_text}"`
3. To improve quality, the worker also retrieves the patient's 3 most recent prior summaries from a vector store and includes them as context.
4. The generated summary is stored in Postgres (`summaries` table) and shown to the clinician.

### 3.3 Review and sign-off
1. Clinician opens the summary at `GET /api/summaries/{summary_id}`.
2. Clinician edits if needed and clicks Sign. `PATCH /api/summaries/{summary_id}` accepts the full summary object from the client and saves it.

### 3.4 Patient record access
- `GET /api/patients/{patient_id}` returns the patient's demographics and full document/summary history.
- Patient IDs are sequential integers.

---

## 4. Architecture

- **Frontend:** React SPA (web) + React Native app (iOS/Android) for clinicians on the go.
- **API:** Node/Express service, containerized, running on AWS EKS.
- **Workers:** summarization workers, same container image as the API, run with elevated permissions to reach S3 and the vector store.
- **Data:** Postgres (RDS) for structured data; S3 for documents; Pinecone (managed) for the vector store.
- **Auth:** Clinicians log in with email + password. Clinic admins can optionally enable Google SSO. Sessions are JWTs signed with HS256; the signing secret is shared across all services via an env var. JWTs carry `clinic_id` and `role` claims and are trusted by downstream services.

### 4.1 Deployment
- Single EKS cluster, one namespace for all services. Images built in GitHub Actions and pushed to a public-read ECR repo (to simplify pulls from a partner's cluster). Images tagged `latest`.
- The API container runs as root and mounts the node's Docker socket to build preview containers on the fly.
- No admission controller; developers set their own pod specs.

### 4.2 Dockerfile (API)
```dockerfile
FROM node:18
COPY . .
RUN npm install
ENV ANTHROPIC_API_KEY=sk-ant-REDACTED_IN_REAL_LIFE
CMD ["node", "server.js"]
```

---

## 5. Third-party vendors

| Vendor | Use | Access |
|---|---|---|
| Anthropic (Claude API) | Summarization | Receives extracted document text |
| AWS (RDS, S3, EKS) | Hosting | All data |
| Pinecone | Vector store for prior-summary retrieval | Stores summary embeddings |
| Twilio | SMS appointment reminders to patients | Receives patient phone + name |
| Segment | Product analytics | Receives page views, includes URL paths |
| Sentry | Error tracking | Receives stack traces and request context |
| Google Workspace | SSO (optional per clinic) | OAuth: requests `openid email profile https://www.googleapis.com/auth/drive.readonly` |

Vendors are integrated as needed by engineers. There is no formal vendor onboarding review. API keys are long-lived.

---

## 6. Logging and monitoring

- Application logs go to CloudWatch. On error, the full request (including body) is logged to help debugging.
- Sentry captures exceptions with request context attached.
- No specific alerting on PHI access volume or failed-authorization spikes today. CloudTrail is enabled in the primary region only.

---

## 7. Data lifecycle

- Documents and summaries are retained indefinitely.
- Account deletion sets `deleted_at` on the clinic row (soft delete).
- Nightly RDS snapshots and an S3 backup bucket, both in the same AWS account, using the same KMS key as production.
- Staging environment is refreshed weekly from a production database dump for realistic test data.

---

## 8. Non-goals (for this version)

- On-prem deployment
- Direct EHR integration
- Patient-facing access

---

## 9. Open questions

- Do we need a BAA with every vendor, or only those that "store" PHI?
- Is Google SSO's requested scope set appropriate?
- How do we handle EU clinics' data residency?
