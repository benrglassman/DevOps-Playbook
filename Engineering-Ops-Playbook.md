
# Engineering Ops Playbook (Non‑Regulated SaaS)
*A step‑by‑step guide to Branching & Versioning, Builds & Environments, Backups & DR, and Security & Privacy.*

**Audience:** Engineering + Ops teams (web, API, mobile).  
**Applies to:** Most SaaS that are not handling regulated financial or health data.  
**Defaults:** GitHub, GitHub Actions, Docker, GCP (Cloud Run, Cloud SQL Postgres, GCS), React/React Native, Flask/FastAPI.

---

## Table of Contents

1. [Branching & Versioning](#1-branching--versioning)  
   1.1 Goals & Definitions  
   1.2 One‑Time Setup  
   1.3 Day‑to‑Day Flow (Feature → PR → Merge)  
   1.4 Release Process (SemVer)  
   1.5 Hotfix Process  
   1.6 Definition of Done (DoD)  
   1.7 Quick References & Templates

2. [Environments, Builds & CI/CD](#2-environments-builds--cicd)  
   2.1 Environment Matrix  
   2.2 One‑Time Setup  
   2.3 PR Pipeline (Checks)  
   2.4 Deploy to Staging  
   2.5 Production Release (Canary/Blue‑Green)  
   2.6 Database Migrations  
   2.7 Mobile App Releases  
   2.8 Rollback Playbook  
   2.9 CI/CD Examples (GitHub Actions)

3. [Backups & Disaster Recovery](#3-backups--disaster-recovery)  
   3.1 RPO/RTO Targets  
   3.2 One‑Time Setup  
   3.3 Automated Backups & Monitoring  
   3.4 Quarterly Restore Drill (Step‑by‑Step)  
   3.5 DR Runbook Template  
   3.6 Verification & Reporting

4. [Security & Privacy](#4-security--privacy)  
   4.1 Identity & Access (SSO/MFA)  
   4.2 Secrets & Keys (Storage, Rotation)  
   4.3 App & API Security Controls  
   4.4 Data Classification, Retention & DSRs  
   4.5 Supply‑Chain Security  
   4.6 Cloud & Network Hardening  
   4.7 Monitoring, Alerting & Incident Response  
   4.8 Minimal Compliance Docs to Maintain  
   4.9 Cadence Calendar

5. [Appendix: Starter Docs](#5-appendix-starter-docs)  
   5.1 `SECURITY.md` (skeleton)  
   5.2 `RELEASE.md` (skeleton)  
   5.3 DR Runbook (skeleton)  
   5.4 Data Retention Policy (skeleton)

---

## 1) Branching & Versioning

### 1.1 Goals & Definitions
- **Goal:** Keep `main` always deployable; ship small, frequent, reversible changes.
- **Definitions:**
  - **Trunk‑based development:** Short‑lived feature branches, frequent merges to `main`.
  - **SemVer:** `MAJOR.MINOR.PATCH` (breaking / features / fixes). Pre‑releases like `-rc.1`, `-beta.2`.
  - **Conventional Commits:** `feat:`, `fix:`, `perf:`, `refactor:`, `docs:`, `chore:`, etc.

### 1.2 One‑Time Setup
1. **Protect `main`:** Require PRs, 1–2 code reviews, status checks, linear history, signed commits (optional).
2. **(Optional) `develop`:** If your team is large or releases are orchestrated, add `develop` as an integration branch.
3. **Branch naming:** Enforce via repo rules or bot.
   - `feat/<scope>`, `fix/<ticket>`, `chore/<task>`, `docs/<page>`.
4. **PR template:** Include summary, screenshots, test plan, rollout/rollback notes.
5. **Changelog automation:** Use commit conventions (e.g., `release-please`/`semantic‑release`).

### 1.3 Day‑to‑Day Flow (Feature → PR → Merge)
1. **Create a branch:**  
   ```bash
   git checkout -b feat/onboarding-flow
   ```
2. **Write code + tests.**
3. **Commit using Conventional Commits:**  
   ```bash
   git commit -m "feat(onboarding): add email capture step"
   git commit -m "fix(onboarding): correct validation error message"
   ```
4. **Open PR → CI runs checks.**
5. **Review & update:** Address comments; ensure “green” checks.
6. **Merge to `main`:** Squash‑merge keeps history clean; delete branch.

### 1.4 Release Process (SemVer)
1. **Auto‑version bump** based on commit types or manually decide.
2. **Tag release:** `v1.8.0`.
3. **Generate changelog** (automated).
4. **CI builds artifact** (Docker image, app bundle) and publishes to registry/store.
5. **Deploy to staging** → smoke tests → **promote to prod** (manual approval recommended).

### 1.5 Hotfix Process
1. Branch from `main`: `hotfix/1.8.1-fix-500s`.
2. Implement + tests → PR → expedited review.
3. Merge to `main` → tag `v1.8.1` → deploy.
4. Cherry‑pick to `develop` if used.

### 1.6 Definition of Done (DoD)
- All CI checks pass (lint, types, unit, integration, basic e2e).
- Code reviewed and approved.
- Feature flags in place (if risky).
- DB migrations are forward‑compatible (and reversible plan documented).
- Observability added (logs/metrics).
- Security checklist touched (input validation, access controls).
- Release notes drafted.

### 1.7 Quick References & Templates
- **Conventional Commit examples:**  
  - `feat(auth): add SSO via Google`  
  - `fix(api): handle null user agent`  
  - `chore(deps): bump flask to 3.0.2`
- **Release tags:** `vMAJOR.MINOR.PATCH`, pre‑release `v1.9.0-rc.1`.

---

## 2) Environments, Builds & CI/CD

### 2.1 Environment Matrix
| Env | Purpose | Access | Data | Deploy Style |
|---|---|---|---|---|
| Dev | Local + PR previews | Engineers | Seed/scrubbed | Auto on PR |
| Staging | Prod‑like testing | Eng + QA + PM | Scrubbed subset | Auto from `main` |
| Prod | Customer‑facing | On‑call + Leads | Live | Manual promote (canary/blue‑green) |

### 2.2 One‑Time Setup
1. **IaC:** Terraform/Pulumi for Cloud Run, Cloud SQL (Postgres), VPC, GCS buckets, IAM.
2. **Secrets:** Cloud Secret Manager; wire into GitHub Actions via OIDC → short‑lived tokens.
3. **Artifact registry:** Configure Docker registry; image naming `service:gitsha` and `service:semver` tags.
4. **Feature flags:** e.g., LaunchDarkly or OSS (Unleash).

### 2.3 PR Pipeline (Checks)
1. **Static checks:** lint, typecheck.
2. **Unit tests** with coverage gate (e.g., ≥80% lines on changed files).
3. **Integration tests** (spin ephemeral services via docker‑compose).
4. **Security scans:** SAST (code) + dependency vulnerability check.
5. **Build** (Docker image) + **store** as ephemeral artifact.
6. **(Optional) PR preview** deploy (temporary URL).

### 2.4 Deploy to Staging
1. Merge to `main` triggers **build + push** (Docker image `:v1.8.0`).  
2. **Database migrations** run in “online” mode.  
3. **Smoke tests** (health checks, key flows).  
4. Notify channel with build metadata and changelog.

### 2.5 Production Release (Canary/Blue‑Green)
1. **Manual approval** in CI to promote staging image to prod.
2. **Canary:** 10% traffic → observe metrics/logs → 50% → 100%.  
   - Roll back if error rate/latency exceeds SLOs.
3. **Blue‑green:** Stand up “green” stack → cut over DNS/traffic → retire “blue” after bake period.

### 2.6 Database Migrations
1. **Design migrations forward‑compatible:** no dropping columns in same release as code stop using them.
2. **Order of operations:** add new columns → dual‑write → backfill → flip reads → remove old.
3. **Tools:** Alembic (Python), Flyway/Liquibase (polyglot).
4. **Rollback plan:** scripted reversal or compensating migration.

### 2.7 Mobile App Releases
1. **Schemes/flavors:** Debug/Staging/Release with separate bundle IDs.
2. **Versioning:** `versionName`/`versionCode` (Android) & `CFBundleShortVersionString`/`CFBundleVersion` (iOS).
3. **Tracks:** Internal → TestFlight/Closed track → Production (phased rollout).
4. **Crash & perf:** Sentry/Crashlytics enabled before widening rollout.

### 2.8 Rollback Playbook
1. Identify bad release (tag/sha).  
2. `git revert` or redeploy previous image tag (kept for N releases).  
3. If DB schema incompatible, run rollback migration or restore to PITR timestamp (coordinate carefully).  
4. Post‑incident: root cause, action items, safeguards.

### 2.9 CI/CD Examples (GitHub Actions)
**Build & Test (backend example):**
```yaml
name: ci-backend
on:
  pull_request:
    paths: [ "api/**" ]
  push:
    branches: [ "main" ]
jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }
      - run: pip install -r api/requirements.txt
      - run: ruff api && mypy api
      - run: pytest -q --maxfail=1 --disable-warnings
      - name: SAST & deps scan
        uses: github/codeql-action/init@v3
        with: { languages: python }
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ${{ secrets.REGISTRY }}
          username: ${{ secrets.REGISTRY_USER }}
          password: ${{ secrets.REGISTRY_PASS }}
      - name: Build image
        run: |
          docker build -t $REGISTRY/myapp-api:${{ github.sha }} -f api/Dockerfile ./api
          docker push $REGISTRY/myapp-api:${{ github.sha }}
```

**Promote to Staging/Prod (Cloud Run, canary):**
```yaml
name: deploy-backend
on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [ staging, production ]
        required: true
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - name: Auth to GCP via OIDC
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.GCP_WIF_PROVIDER }}
          service_account: ${{ secrets.GCP_SA_EMAIL }}
      - uses: google-github-actions/setup-gcloud@v2
      - name: Deploy to Cloud Run
        run: |
          IMAGE="${{ secrets.REGISTRY }}/myapp-api:${{ github.sha }}"
          gcloud run deploy myapp-api-${{ inputs.environment }} \
            --image=$IMAGE \
            --region=${{ vars.GCP_REGION }} \
            --platform=managed \
            --set-secrets=DATABASE_URL=projects/.../secrets/DB_URL:latest \
            --min-instances=1 --max-instances=20
      - name: Canary traffic (prod only)
        if: inputs.environment == 'production'
        run: |
          SVC="myapp-api-production"
          gcloud run services update-traffic $SVC --to-latest --region=${{ vars.GCP_REGION }} --splits LATEST=10
          sleep 300
          # TODO: insert health-check script here; on success:
          gcloud run services update-traffic $SVC --to-latest --region=${{ vars.GCP_REGION }} --splits LATEST=100
```

---

## 3) Backups & Disaster Recovery

### 3.1 RPO/RTO Targets
- **Primary DB (Postgres):** RPO ≤ 15 minutes (PITR), RTO 2–8 hours.  
- **Object storage:** RPO ≤ 24 hours (versioning & lifecycle), RTO ≤ 24 hours.

### 3.2 One‑Time Setup
1. **Cloud SQL Postgres:**
   - Enable **Automated Backups** + **Point‑In‑Time Recovery (PITR/WAL)**.
   - Set retention: 35 days (or per your risk appetite).
   - Cross‑region read replica (optional for HA).
2. **GCS Buckets:**
   - Turn **Object Versioning** ON.
   - Add **Lifecycle rules**: delete noncurrent versions after 90 days (or as policy dictates).
3. **IaC & State:**
   - Store Terraform state in remote backend (GCS) with versioning + KMS encryption.
4. **3‑2‑1 Rule:**
   - 3 copies (primary + backup + DR copy), 2 different storage types/locations, 1 off‑site/cross‑region.
5. **Secrets:**
   - Back up **secret versions** (Cloud Secret Manager has built‑in versioning; export periodically to secure vault).

### 3.3 Automated Backups & Monitoring
1. **Schedule exports** (daily full) to a dedicated, locked GCS bucket.  
2. **Enable alerts**:
   - Backup failure warnings.
   - Replication lag (if using replicas).
   - Storage growth anomalies.
3. **Checksum/verify** DB exports (e.g., `pg_dump` + verify size/hash).

### 3.4 Quarterly Restore Drill (Step‑by‑Step)
1. **Select timestamp** (random within last 30 days).  
2. **Provision temp DB** in an isolated project/env.  
3. **Restore** via PITR or from export.  
4. **Run migrations** to app’s current schema if needed.  
5. **Boot a staging app** pointing to restored DB.  
6. **Run validation tests:** user auth, CRUD flows, reports.  
7. **Record timings** vs RTO; note any data gaps vs RPO.  
8. **Tear down** temporary resources; file post‑drill notes + improvements.

### 3.5 DR Runbook Template
- **Trigger Conditions:** region outage, data corruption, security incident.  
- **Contacts:** on‑call rotation, platform lead, security lead.  
- **Decision Tree:** failover vs restore vs wait.  
- **Steps:** (a) freeze deploys, (b) revoke risky credentials, (c) promote replica or restore to timestamp, (d) update DNS/traffic, (e) verify health checks & SLOs.  
- **Comms:** internal Slack channel, status page, customer email template.  
- **After Action:** incident review, action items, docs update.

### 3.6 Verification & Reporting
- **Monthly report:** backup success rate, average sizes, restore test result.  
- **KPIs:** % successful backups, mean restore time, data loss window.  
- **Evidence:** screenshots/logs of backup jobs, drill docs.

---

## 4) Security & Privacy

### 4.1 Identity & Access (SSO/MFA)
1. **SSO everywhere** (Google/Microsoft Okta style).  
2. **MFA required** for all staff and production consoles.  
3. **Least‑privilege IAM:** role per service; deny by default.  
4. **Quarterly access review** and deprovisioning checklist.

### 4.2 Secrets & Keys (Storage, Rotation)
1. **Use Cloud Secret Manager**; no plaintext in repos/CI vars.  
2. **Rotate** API keys, DB passwords, signing keys ≤90 days or on staff change.  
3. **Short‑lived workload identities** (OIDC) from CI to cloud provider (no static keys).  
4. **Encrypt at rest & in transit** with managed KMS; document key rotation.

### 4.3 App & API Security Controls
1. **TLS 1.2+**, HSTS, secure cookies, CSRF where relevant.  
2. **AuthZ:** RBAC/ABAC; server‑side checks for every sensitive action.  
3. **Input validation & output encoding** to prevent injection/XSS.  
4. **Rate limiting & anti‑abuse** (per‑IP/user, CAPTCHA when abused).  
5. **Multi‑tenant isolation:** tenant ID scoping at query/service layer; add assertions/tests.  
6. **Security headers:** Content‑Security‑Policy, X‑Frame‑Options, etc.  
7. **Logging:** no secrets/PII; include request IDs; audit sensitive events.

### 4.4 Data Classification, Retention & DSRs
1. **Classify data:** PII, SPI, telemetry, public.  
2. **Minimize:** collect only what’s needed; redact before log/analytics.  
3. **Retention:** define tables/buckets lifetimes; auto‑purge via lifecycle or scheduled tasks.  
4. **Data Subject Requests (DSRs):** export/delete flow; 30‑day target; track in ticketing system.

### 4.5 Supply‑Chain Security
1. **Pin dependencies** via lockfiles; automated PRs to update (Dependabot/Renovate).  
2. **SAST** on PR; **dependency** vuln scans in CI.  
3. **Container hygiene:** slim base images, weekly rebuilds, non‑root user.  
4. **SBOM** generation (CycloneDX) for distributed artifacts; sign images (cosign).

### 4.6 Cloud & Network Hardening
1. **Private subnets** for DBs; restrict egress; least‑open security groups.  
2. **WAF/CDN** in front of public apps; basic DDoS protections.  
3. **CIS benchmarks** where feasible; auto‑remediation for drift.  
4. **Logging:** Cloud audit logs on; alert on anomalous IAM events.

### 4.7 Monitoring, Alerting & Incident Response
1. **Golden signals:** uptime, error rate, latency p95, saturation per service.  
2. **Alerting:** on SLO breaches and auth anomalies.  
3. **Runbooks:** link every alert to a playbook.  
4. **Incident process:** sev levels, commander, comms, postmortem in 5 business days.

### 4.8 Minimal Compliance Docs to Maintain
- `SECURITY.md` (public), `PRIVACY.md`, `DATA-RETENTION.md`, `SUBPROCESSORS.md`, `IR-PLAN.md`, `ACCESS-REVIEW.md` checklist.

### 4.9 Cadence Calendar
- **Weekly:** patch dependencies, triage security issues.  
- **Monthly:** review alerts & dashboards.  
- **Quarterly:** access review, secret rotation check, restore drill.  
- **Annually:** IR tabletop exercise, policy review.

---

## 5) Appendix: Starter Docs

### 5.1 `SECURITY.md` (skeleton)
```
# Security Overview
We practice least-privilege access, MFA, encryption in transit and at rest, dependency scanning, and regular backups.

## Reporting a Vulnerability
Email security@yourdomain.com. We triage within 3 business days.

## Key Practices
- SSO + MFA for all staff.
- Secrets in Secret Manager; rotated ≤90 days.
- Dependency scanning on every PR.
- Backups: daily full + PITR; quarterly restore drills.
```

### 5.2 `RELEASE.md` (skeleton)
```
# Release Process
- Versioning: SemVer (MAJOR.MINOR.PATCH). RC tags for staging.
- Branching: feature branches → PR → main; hotfix from main.

## Checklist
- [ ] CI green (lint, tests, scans)
- [ ] Migrations safe & documented
- [ ] Feature flags toggled
- [ ] Staging smoke tests passed
- [ ] Canary 10% → 50% → 100%
- [ ] Rollback plan verified
```

### 5.3 DR Runbook (skeleton)
```
# Disaster Recovery
## Triggers
- Region outage, data corruption, security event.

## Steps
1. Freeze deploys. Assemble incident channel.
2. Decide: failover vs restore.
3. Promote replica or restore to timestamp.
4. Reroute traffic; validate health & SLOs.
5. Communicate to customers; postmortem.
```

### 5.4 Data Retention Policy (skeleton)
```
# Data Retention
- PII: 24 months post-account deletion.
- Logs: 30 days (no PII).
- Backups: 35 days rolling.
- Deletion: automated pipelines with audit logs.
```
