# SpikeSignals — Automated Deployment Pipeline

This repository demonstrates the GitHub Actions CI/CD pipeline built for the SpikeSignals Django/ECS backend. Every push to the `development` branch automatically runs tests and deploys to AWS — no manual steps, no SSH, no scripts.

---

## How It Works

```
Developer pushes code
        │
        ▼
┌─────────────────────────────────────────────┐
│           GitHub Actions triggers            │
│         CI/CD — Development workflow         │
└─────────────────────────────────────────────┘
        │
        ▼
╔═════════════════════════════════════════════╗
║  JOB 1 — Tests & Static Checks             ║
║  (no AWS credentials, runs immediately)     ║
╠═════════════════════════════════════════════╣
║  ✦ Python syntax verification               ║
║  ✦ Django system checks                     ║
║  ✦ Migration dry-run (catches missing       ║
║    migrations before they hit production)   ║
║  ✦ Full pytest suite                        ║
╚═════════════════════════════════════════════╝
        │
        │  If any test fails — pipeline stops here.
        │  Deploy job never runs. AWS is never touched.
        │
        ▼  Only continues if ALL tests pass ✅
╔═════════════════════════════════════════════╗
║  JOB 2 — Deploy to ECS                     ║
║  (assumes AWS role via OIDC, no keys)       ║
╠═════════════════════════════════════════════╣
║  ✦ Build Docker image (Dockerfile.api)      ║
║  ✦ Push to Amazon ECR (2 tags)              ║
║    · :abc1234  ← immutable commit SHA       ║
║    · :dev-latest                            ║
║  ✦ Register new ECS task definition         ║
║  ✦ Run database migrations                  ║
║    (one-off Fargate task — blocks if fail)  ║
║  ✦ Update ECS service                       ║
║  ✦ Wait for service to stabilise            ║
║  ✦ Smoke test /health/ (5 retries)          ║
║  ✦ Write deployment summary                 ║
╚═════════════════════════════════════════════╝
        │
        ▼
   Deploy complete ✅
```

---

## Branch → Environment Map

| Branch | Environment | How it deploys | Approval |
|--------|-------------|----------------|----------|
| `development` | Dev (AWS ECS) | Automatic on every push | None — instant |
| `staging` | Staging (AWS ECS) | Automatic on every push | None — instant |
| `production` | Production (AWS ECS) | Tag `v1.2.3` push | **Manual approval required** |

---

## The Migration Safety Gate

One of the most critical parts of the pipeline is how database migrations are handled. This is a common failure point in manual deployments.

```
New task definition registered
           │
           ▼
  ┌─────────────────────┐
  │  One-off Fargate    │
  │  task launches with │
  │  the NEW image      │
  │                     │
  │  python manage.py   │
  │  migrate --noinput  │
  └─────────────────────┘
           │
     ┌─────┴──────┐
     │            │
  Exit 0       Exit 1
  (success)   (failure)
     │            │
     ▼            ▼
  ECS service   Pipeline
  updated   ←✗  stops.
  ✅            ECS service
               is NOT updated.
               Old version
               keeps running.
```

If a migration fails, the old version of the app stays live. The broken code never reaches the ECS service. This replaces the previous approach of running migrations manually over SSH before a deploy.

---

## Security — No Long-Lived AWS Keys

The pipeline uses **OpenID Connect (OIDC)** to authenticate with AWS. There are no AWS access keys stored anywhere.

```
GitHub Actions runner
        │
        │  "I am a workflow running on
        │   the development branch of
        │   Bright-Access-Consulting-LLC/spikesignals-backend"
        │
        ▼
AWS IAM verifies the signed token
        │
        ▼
Issues a short-lived session token
(expires when the workflow finishes)
        │
        ▼
Runner assumes the deploy role
and performs only the actions
that role is allowed to do
```

Each environment (dev / staging / production) has its own IAM role. The production role can only be assumed by a `v*.*.*` tag push — not by any branch push, PR, or manual trigger.

---

## What a Deployment Looks Like

After every successful deploy, GitHub writes a summary directly into the Actions run:

```
┌────────────────────┬──────────────────────────────────────────────┐
│ Environment        │ Development                                   │
│ Status             │ ✅ Success                                    │
│ Commit SHA         │ abc1234def5678...                             │
│ Author             │ ramneek                                       │
│ Image URI          │ 123456789.dkr.ecr.us-east-1.amazonaws.com/   │
│                    │ spikesignals-api:abc1234def5678               │
│ Task Definition    │ arn:aws:ecs:us-east-1:...:task-definition/   │
│                    │ spikesignals-api-dev:42                       │
│ Branch             │ development                                   │
└────────────────────┴──────────────────────────────────────────────┘
```

Every deploy is fully traceable — commit SHA → Docker image → ECS task definition → running containers.

---

## What Was Replaced

| Before | After |
|--------|-------|
| SSH into EC2 server | No server access needed |
| Run `deploy_phase3.py` manually | Triggered automatically on push |
| Manually run `python manage.py migrate` over SSH | Automated Fargate task, blocks deploy on failure |
| Restart gunicorn/daphne via systemd | ECS rolling deployment with health checks |
| Run `post_deploy_smoke.sh` by hand | Automated smoke test with retry logic |
| Hope the deploy worked | `/health/` verified after every deploy |

---

## Rollout Plan

```
Phase 1 — Development  ◄── You are here
    Validate full pipeline end-to-end
    Confirm migrations, smoke test, summaries all work
          │
          ▼
Phase 2 — Staging
    Copy workflow, swap secrets
    Mirror production infra at smaller scale
          │
          ▼
Phase 3 — Production
    Tag-based trigger (v1.0.0, v1.1.0, ...)
    Manual approval gate before deploy runs
    Full CloudWatch alerting
```

---

## Files in This Repository

| File | Purpose |
|------|---------|
| `.github/workflows/deploy-development.yml` | The full CI/CD pipeline |
| `GITHUB_SECRETS.md` | All secrets that need to be configured in GitHub |
| `OIDC_SETUP.md` | IAM role setup — copy-paste trust and permissions policies |
