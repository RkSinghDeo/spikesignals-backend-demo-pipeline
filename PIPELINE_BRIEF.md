# Pipeline Engineering Brief

This document is the exact specification provided to build this deployment pipeline.
It serves as a reference for what was asked, what decisions were made, and why —
so the client and engineering team have full visibility into the requirements
that shaped the final implementation.

---

## Context Provided to the Engineer

> *The following was the briefing given. Nothing was assumed — every decision in the
> pipeline traces back to a requirement stated here.*

---

### Stack

- Django 6.0.2 with Daphne ASGI server on port 8000
- Dockerfile is named `Dockerfile.api` (not `Dockerfile`)
- Container name in ECS task definition: `spikesignals-api`
- Health check endpoint: `GET /health/` returns `{"status": "ok"}`
- `entrypoint.sh` runs `collectstatic` then starts Daphne
- `entrypoint.sh` does **not** run migrations — migrations must be run separately
- Single AWS account — dev, staging, and production all in same account
- ECR registry already exists
- ECS clusters already exist — no infrastructure is created by the pipeline
- All secrets already in AWS Secrets Manager — nothing is moved
- Redis URL is hardcoded in `settings.py`
- `DJANGO_ALLOWED_HOSTS` needs to include the ALB DNS name

### GitHub Branches

| Branch | Environment |
|--------|-------------|
| `development` | Development |
| `staging` | Staging |
| `production` | Production (live) |

---

## What Was Asked to Build

### Two GitHub Actions workflow files

---

#### File 1 — `deploy-staging.yml`

**Trigger:** push to `staging` branch only

**Steps in exact order:**

1. Checkout code
2. Configure AWS credentials using OIDC
   - Role from secret `STAGING_DEPLOY_ROLE_ARN`
   - Region: `us-east-1`
3. Login to Amazon ECR
4. Build Docker image using `Dockerfile.api`
   - Tag 1: commit SHA (`github.sha`)
   - Tag 2: `staging-latest`
5. Push both tags to ECR
6. Get current ECS task definition
   - Family name from secret `ECS_TASK_DEF_STAGING`
7. Swap the image URI in the task definition to the new SHA-tagged image
   - Use `jq` to do this
   - Remove: `taskDefinitionArn`, `revision`, `status`, `requiresAttributes`, `compatibilities`, `registeredAt`, `registeredBy`
8. Register the updated task definition
9. Run Django migrations as a one-off ECS Fargate task
   - Use the **new** task definition ARN just registered
   - Override the container command to: `python manage.py migrate --noinput`
   - Network configuration from secret `ECS_NETWORK_CONFIG`
   - Wait using `aws ecs wait tasks-stopped`
   - Check exit code — if not `0`, print error and `exit 1`
   - **This blocks the deploy — ECS service is NOT updated if this fails**
10. Update ECS service with new task definition
    - Cluster from secret `ECS_CLUSTER_STAGING`
    - Service from secret `ECS_SERVICE_STAGING`
    - Use `--force-new-deployment`
11. Wait for ECS service to reach steady state using `aws ecs wait services-stable`
12. Smoke test
    - URL from secret `STAGING_URL` appended with `/health/`
    - Retry up to 5 times with 10 second gaps
    - Fail the workflow if all 5 attempts fail
13. Write deployment summary to `GITHUB_STEP_SUMMARY`
    - Show: environment, commit SHA, image URI, task definition ARN, author, status

**Concurrency:** `group: deploy-staging`, `cancel-in-progress: false`

---

#### File 2 — `deploy-production.yml`

**Trigger:** push of tags matching `v*.*.*` pattern only

**Environment:** `production`
- Must be set so GitHub waits for manual approval before the job runs

**Steps:** identical to `deploy-staging.yml` except:

- Use `PRODUCTION_DEPLOY_ROLE_ARN` instead of `STAGING_DEPLOY_ROLE_ARN`
- Use `ECS_TASK_DEF_PRODUCTION` instead of `ECS_TASK_DEF_STAGING`
- Use `ECS_CLUSTER_PRODUCTION` instead of `ECS_CLUSTER_STAGING`
- Use `ECS_SERVICE_PRODUCTION` instead of `ECS_SERVICE_STAGING`
- Tag the image with `github.ref_name` (the version tag e.g. `v1.2.0`) AND `github.sha` AND `production-latest`
- Use `PRODUCTION_URL` instead of `STAGING_URL`
- Summary should clearly say **Production** in bold

**Concurrency:** `group: deploy-production`, `cancel-in-progress: false`

---

### GitHub Secrets — full list

All workflows use these secrets. No values are hardcoded. Every secret must be listed in a comment block at the top of each file.

| Secret | Purpose |
|--------|---------|
| `STAGING_DEPLOY_ROLE_ARN` | IAM role for staging OIDC |
| `PRODUCTION_DEPLOY_ROLE_ARN` | IAM role for production OIDC |
| `ECR_REGISTRY` | ECR registry hostname |
| `ECS_CLUSTER_STAGING` | ECS cluster name — staging |
| `ECS_CLUSTER_PRODUCTION` | ECS cluster name — production |
| `ECS_SERVICE_STAGING` | ECS service name — staging |
| `ECS_SERVICE_PRODUCTION` | ECS service name — production |
| `ECS_TASK_DEF_STAGING` | Task definition family — staging |
| `ECS_TASK_DEF_PRODUCTION` | Task definition family — production |
| `ECS_NETWORK_CONFIG` | JSON network config for one-off Fargate tasks |
| `STAGING_URL` | Base URL for staging smoke test |
| `PRODUCTION_URL` | Base URL for production smoke test |

---

### Technical Requirements

1. Every workflow must declare:
   ```yaml
   permissions:
     id-token: write
     contents: read
   ```

2. Exact `jq` pattern to use for image swap:
   ```bash
   NEW_TASK_DEF=$(echo $TASK_DEF | jq \
     --arg IMAGE "$NEW_IMAGE_URI" \
     '.containerDefinitions[0].image = $IMAGE |
      del(.taskDefinitionArn,
          .revision,
          .status,
          .requiresAttributes,
          .compatibilities,
          .registeredAt,
          .registeredBy)')
   ```

3. Migration one-off task must use `--launch-type FARGATE` and network configuration from `ECS_NETWORK_CONFIG` secret

4. All `echo` statements use emoji prefixes:
   - `✅` success
   - `❌` failure
   - `🔄` in progress

5. Every step must have a clear descriptive name

6. Smoke test must retry — never fail on first attempt

7. If smoke test fails on production, print a clear message telling the team to check CloudWatch logs and manually verify the ECS service

---

### After Creating the Workflow Files

1. Verify YAML syntax is valid for both files
2. Create `GITHUB_SECRETS.md` listing every secret with description and example format
3. Create `OIDC_SETUP.md` with exact IAM trust policy and permissions policy JSON for both roles
4. Cross-check both files for:
   - Correct YAML indentation
   - All secrets referenced exist in the secrets list
   - No hardcoded AWS account IDs or resource names
   - Correct branch and tag trigger patterns
   - Correct environment names

---

## Decisions Made During Build

These were not in the original brief but were resolved during implementation:

| Decision | What was chosen | Why |
|----------|-----------------|-----|
| CI integration | Test job merged into each deploy workflow | One workflow file per environment — full picture in one place. `ci.yml` kept temporarily until validated then deleted. |
| Development pipeline first | Only `deploy-development.yml` built initially | Validate the pattern end-to-end before replicating to staging and production |
| No PR-only pipelines | All workflows trigger on push, not pull_request | Simpler — every push to a branch deploys. PRs don't gate deploys. |
| `workflow_dispatch` added | Manual trigger added to dev workflow | Lets the team re-trigger deploys from the GitHub UI without a dummy push |
| Migration command override | `["python","manage.py","migrate","--noinput"]` passed as array | ECS container override requires array format, not a shell string |
| `iam:PassRole` in IAM policy | Included for both task execution role and task role | Required for `run-task` and `register-task-definition` — most common gotcha that causes silent `AccessDenied` |
| `cancel-in-progress: false` | Never cancel a running deploy | A mid-flight deploy that gets cancelled leaves infrastructure in an unknown state |

---

## What This Pipeline Replaces

The previous deployment process involved:

1. SSHing into an EC2 server
2. Running `deploy_phase3.py` — a Python script that patched files directly on disk
3. Manually running `python manage.py migrate` over SSH
4. Restarting gunicorn and daphne via systemd
5. Running `post_deploy_smoke.sh` by hand

Every step was manual, undocumented at execution time, and dependent on a developer having the right SSH key and IAM permissions on their local machine.

---

*Built by Bright Access Consulting using Claude Code.*
