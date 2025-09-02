
# Engineering Ops Deep-Dive Manual
*A full “how-to” guide with detailed step-by-step instructions + AI agent prompts.*

---

## Table of Contents
1. Branching & Versioning  
   - Setup protected branches  
   - Create feature branch  
   - Commit with Conventional Commits  
   - Open PR and run CI checks  
   - Merge to `main`  
   - Tag and release (SemVer)  
   - Hotfix process  
   - AI prompts for GitOps automation  

2. Environments, Builds & CI/CD  
   - Provision dev/staging/prod environments  
   - Connect GitHub Actions to GCP via OIDC  
   - Configure PR pipelines (lint/tests/scans)  
   - Build Docker images  
   - Deploy to staging  
   - Canary deploy to production  
   - Rollback flow  
   - AI prompts for CI/CD automation  

3. Backups & Disaster Recovery  
   - Enable automated backups (Cloud SQL)  
   - Setup PITR  
   - Backup verification alerts  
   - Quarterly restore drill (hands-on)  
   - DR runbook execution  
   - AI prompts for backup verification  

4. Security & Privacy  
   - Enforce MFA/SSO  
   - Store secrets in Secret Manager  
   - Rotate keys/secrets  
   - Configure IAM least-privilege  
   - Add dependency scanning  
   - Configure logging and audit trails  
   - Incident response process  
   - AI prompts for security review  

---

## 1. Branching & Versioning (Step-by-Step)

### Setup Protected Branches
- **GitHub UI:**  
  1. Repo → *Settings* → *Branches*.  
  2. Add branch protection for `main`.  
  3. Require pull request reviews (min 1).  
  4. Require status checks to pass before merge.  
  5. Require signed commits (optional).  

**AI Agent Prompt:**  
*"Write me a GitHub CLI command that enforces branch protection on `main` with required PR reviews and CI checks in repo `<repo-name>`."*

---

### Create Feature Branch
```bash
git checkout -b feat/onboarding-flow
```

**AI Agent Prompt:**  
*"Create a new Git branch named `feat/<feature-name>` based on main and push it to GitHub."*

---

### Commit with Conventional Commits
```bash
git add .
git commit -m "feat(auth): add Google SSO login"
```

**AI Agent Prompt:**  
*"Suggest a proper Conventional Commit message for adding password reset to the user API."*

---

### Open PR and Run CI Checks
- **GitHub UI:**  
  1. Click *Compare & Pull Request*.  
  2. Fill out PR template (summary, screenshots, test plan).  
  3. CI pipeline auto-runs (lint/tests/scans).  

**AI Agent Prompt:**  
*"Draft a PR description for adding email capture in onboarding, including what was changed, screenshots, and how to test."*

---

### Merge to `main`
- After review + green checks → *Squash and merge* → delete branch.

**AI Agent Prompt:**  
*"Summarize the key changes in this PR into a concise squash commit message using Conventional Commits."*

---

### Tag and Release (SemVer)
```bash
git tag -a v1.2.0 -m "Release v1.2.0"
git push origin v1.2.0
```

**AI Agent Prompt:**  
*"Generate a changelog entry for version 1.2.0 based on these commit messages."*

---

### Hotfix Process
```bash
git checkout main
git pull
git checkout -b hotfix/1.2.1-fix-500s
# make fix
git commit -m "fix(api): resolve 500 error on login"
git push origin hotfix/1.2.1-fix-500s
```

**AI Agent Prompt:**  
*"Write a GitHub Actions workflow to automatically bump PATCH version and deploy hotfix branches once merged."*

---

## 2. Environments, Builds & CI/CD (Step-by-Step)

### Provision Environments (GCP Example)
1. **Dev/Staging/Prod Projects** in GCP.  
2. **Cloud Run services**: `myapp-dev`, `myapp-staging`, `myapp-prod`.  
3. **Cloud SQL Postgres** instances per env.  
4. **GCS buckets** for backups + artifacts.  

**AI Agent Prompt:**  
*"Generate Terraform to provision a Cloud Run service with a Cloud SQL Postgres instance and connect them securely."*

---

### Connect GitHub Actions to GCP via OIDC
1. Create workload identity pool in GCP.  
2. Allow GitHub Actions service account.  
3. Add GitHub Actions `auth@v2` step with OIDC.  

**AI Agent Prompt:**  
*"Write me GitHub Actions config to authenticate to GCP via OIDC for repo `<repo>` and service account `<sa-email>`."*

---

### Build & Deploy
```bash
docker build -t gcr.io/<project>/myapp:${GITHUB_SHA} .
docker push gcr.io/<project>/myapp:${GITHUB_SHA}
gcloud run deploy myapp-staging \
  --image gcr.io/<project>/myapp:${GITHUB_SHA} \
  --region us-central1
```

**AI Agent Prompt:**  
*"Write a shell script to build and push a Docker image to GCP Artifact Registry and deploy to Cloud Run staging."*

---

### Canary Deploy to Production
```bash
gcloud run services update-traffic myapp-prod --to-latest --splits LATEST=10
```

Monitor → then:
```bash
gcloud run services update-traffic myapp-prod --to-latest --splits LATEST=100
```

**AI Agent Prompt:**  
*"Generate a health check script to run during a 10% canary rollout that verifies error rates and latency."*

---

### Rollback
```bash
gcloud run services update-traffic myapp-prod --to-revisions old-revision=100
```

**AI Agent Prompt:**  
*"Write a GitHub Actions rollback job that reverts to the previous revision if error rate >5%."*

---

## 3. Backups & DR (Step-by-Step)

### Enable PITR (Postgres Cloud SQL)
- GCP Console → SQL → Instance → *Backups* → Enable Automated Backups + PITR.

**AI Agent Prompt:**  
*"Write a gcloud CLI command to enable point-in-time recovery on a Cloud SQL Postgres instance."*

---

### Quarterly Restore Drill
1. Pick random timestamp.  
2. Create new Cloud SQL instance.  
3. Restore from backup to that time.  
4. Connect staging app to it.  
5. Run validation tests.  

**AI Agent Prompt:**  
*"Generate a checklist for restoring Cloud SQL Postgres from a PITR backup into a temporary instance and validating it."*

---

## 4. Security & Privacy (Step-by-Step)

### Enforce MFA/SSO
- **Google Workspace/Okta:** Require MFA.  
- **GitHub:** Settings → Security → Require 2FA.  

**AI Agent Prompt:**  
*"Generate instructions for enforcing mandatory MFA in GitHub for all org members."*

---

### Store Secrets
```bash
gcloud secrets create DB_URL --replication-policy="automatic"
echo "postgres://..." | gcloud secrets versions add DB_URL --data-file=-
```

**AI Agent Prompt:**  
*"Write Terraform to store an API key in Secret Manager and inject it into Cloud Run service."*

---

### Rotate Keys
- Create new key → deploy → revoke old.  
- Automate with CI job every 90 days.

**AI Agent Prompt:**  
*"Draft a GitHub Actions workflow to rotate a Cloud SQL password every 90 days and update dependent services."*

---

### Incident Response
- **Slack channel** `#incident-<id>`.  
- **Roles:** Incident Commander, Comms, Ops.  
- **Steps:** freeze deploys → assess impact → mitigate → comms → postmortem.

**AI Agent Prompt:**  
*"Generate a status page incident update for users about elevated error rates in the API, with ETA for resolution."*

---
