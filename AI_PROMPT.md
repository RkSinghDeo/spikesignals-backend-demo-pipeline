# AI Prompt — GitHub Actions ECS Deployment Pipeline

Copy and paste the prompt below into any AI coding assistant to generate this pipeline for your own Django/ECS project. Replace every value inside `< >` with your own.

---

```
I have a Django backend application deployed on AWS ECS using Fargate.
I need you to implement a GitHub Actions deployment pipeline.

I will handle the OIDC provider creation and IAM role creation in AWS
myself. You just need to build the GitHub Actions workflows.

---

CONTEXT

Our stack:
- Django <VERSION> with Daphne ASGI server on port 8000
- Dockerfile is named <DOCKERFILE_NAME> (e.g. Dockerfile.api)
- Container name in ECS task definition is: <CONTAINER_NAME>
- Health check endpoint: GET /health/ returns {"status": "ok"}
- entrypoint.sh runs collectstatic then starts Daphne
- entrypoint.sh does NOT run migrations — migrations must be run separately
- Single AWS account — dev, staging, production all in same account
- ECR registry already exists
- ECS clusters already exist — we are not creating any infrastructure
- All secrets already in AWS Secrets Manager — we are not moving them

Our GitHub branches:
- development — feature work
- staging — pre-production
- production — live

---

WHAT TO BUILD

Create exactly two files in .github/workflows/

---

FILE 1 — deploy-staging.yml

Trigger: push to staging branch only

Steps in exact order:
1. Checkout code
2. Configure AWS credentials using OIDC
   - role-to-assume comes from secret STAGING_DEPLOY_ROLE_ARN
   - region: <AWS_REGION>
3. Login to Amazon ECR
4. Build Docker image using <DOCKERFILE_NAME>
   - Tag 1: commit SHA (github.sha)
   - Tag 2: staging-latest
5. Push both tags to ECR
6. Get current ECS task definition
   - Family name comes from secret ECS_TASK_DEF_STAGING
7. Swap the image URI in the task definition to the new SHA-tagged image
   - Use jq to do this
   - Remove these fields before registering:
     taskDefinitionArn, revision, status,
     requiresAttributes, compatibilities,
     registeredAt, registeredBy
8. Register the updated task definition
9. Run Django migrations as a one-off ECS Fargate task
   - Use the NEW task definition ARN just registered
   - Override the container command to:
     python manage.py migrate --noinput
   - Network configuration comes from secret ECS_NETWORK_CONFIG
   - Wait for the task to stop using aws ecs wait tasks-stopped
   - Check the exit code of the container
   - If exit code is not 0 — print error and exit 1
   - This blocks the deploy — ECS service is NOT updated if this fails
10. Update ECS service with new task definition
    - Cluster from secret ECS_CLUSTER_STAGING
    - Service from secret ECS_SERVICE_STAGING
    - Use --force-new-deployment
11. Wait for ECS service to reach steady state
    - Use aws ecs wait services-stable
12. Smoke test
    - URL from secret STAGING_URL appended with /health/
    - Retry up to 5 times with 10 second gaps
    - Fail the workflow if all 5 attempts fail
13. Write deployment summary to GITHUB_STEP_SUMMARY
    - Show environment, commit SHA, image URI,
      task definition ARN, author, status

Concurrency:
- group: deploy-staging
- cancel-in-progress: false

---

FILE 2 — deploy-production.yml

Trigger: push of tags matching v*.*.* pattern only

Environment: production
- This must be set so GitHub waits for manual approval
  before the job runs

Steps: identical to deploy-staging.yml except:
- Use PRODUCTION_DEPLOY_ROLE_ARN instead of STAGING_DEPLOY_ROLE_ARN
- Use ECS_TASK_DEF_PRODUCTION instead of ECS_TASK_DEF_STAGING
- Use ECS_CLUSTER_PRODUCTION instead of ECS_CLUSTER_STAGING
- Use ECS_SERVICE_PRODUCTION instead of ECS_SERVICE_STAGING
- Tag the image with github.ref_name (the version tag e.g. v1.2.0)
  AND github.sha AND production-latest
- Use PRODUCTION_URL instead of STAGING_URL
- Summary should clearly say Production in bold

Concurrency:
- group: deploy-production
- cancel-in-progress: false

---

GITHUB SECRETS REFERENCE

All workflows use these secrets.
Do not hardcode any values.
List every secret used in a comment block at the top of each file.

Secrets:
- STAGING_DEPLOY_ROLE_ARN
- PRODUCTION_DEPLOY_ROLE_ARN
- ECR_REGISTRY
- ECS_CLUSTER_STAGING
- ECS_CLUSTER_PRODUCTION
- ECS_SERVICE_STAGING
- ECS_SERVICE_PRODUCTION
- ECS_TASK_DEF_STAGING
- ECS_TASK_DEF_PRODUCTION
- ECS_NETWORK_CONFIG
- STAGING_URL
- PRODUCTION_URL

---

IMPORTANT TECHNICAL REQUIREMENTS

1. Every workflow must have these permissions:
   permissions:
     id-token: write
     contents: read

2. Use this exact jq pattern to swap the image:

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

3. Migration one-off task must use --launch-type FARGATE
   and network configuration from ECS_NETWORK_CONFIG secret

4. All echo statements use emoji prefixes:
   ✅ success
   ❌ failure
   🔄 in progress

5. Every step must have a clear descriptive name

6. Smoke test must retry — never fail on first attempt

7. If smoke test fails on production print a clear
   message telling the team to check CloudWatch logs
   and manually verify the ECS service

---

AFTER CREATING THE FILES

1. Verify YAML syntax is valid for both files

2. Create GITHUB_SECRETS.md listing every secret
   that needs to be configured in GitHub with:
   - Description of what each value should be
   - Example format where applicable

3. Create OIDC_SETUP.md documenting exactly what
   IAM trust policy and permissions policy the client
   needs to create for both staging and production
   deploy roles with exact JSON they can copy paste

4. Check both workflow files for:
   - Correct YAML indentation
   - All secrets referenced exist in the secrets list
   - No hardcoded AWS account IDs or resource names
   - Correct branch and tag trigger patterns
   - Correct environment names

5. Tell me if anything needs to be confirmed
   from the client before these workflows will work
```

---

## Values to Replace

| Placeholder | What to put |
|-------------|-------------|
| `<VERSION>` | Your Django version e.g. `6.0.2` |
| `<DOCKERFILE_NAME>` | Your Dockerfile name e.g. `Dockerfile.api` or `Dockerfile` |
| `<CONTAINER_NAME>` | The container name in your ECS task definition |
| `<AWS_REGION>` | Your AWS region e.g. `us-east-1` |

Everything else — cluster names, service names, ECR registry, role ARNs — is kept out of the prompt intentionally and passed as GitHub secrets at runtime.

---

## What the AI Will Produce

- `.github/workflows/deploy-staging.yml` — full staging pipeline
- `.github/workflows/deploy-production.yml` — production pipeline with manual approval gate
- `GITHUB_SECRETS.md` — every secret documented with example values
- `OIDC_SETUP.md` — copy-paste IAM JSON for both deploy roles

---

## Follow-Up Prompts

After the initial build, these follow-up prompts were used to refine the pipeline:

**To merge CI tests into the deploy workflow:**
```
Merge the existing ci.yml test steps into the deploy workflow
as a first job called test. The deploy job should only run if
test passes. Keep ci.yml alongside for now — we will delete it
once the new pipeline is validated.
```

**To add a development environment pipeline:**
```
We are missing the development branch. Add a deploy-development.yml
that triggers on push to the development branch with the same
test → deploy flow. Use DEV_DEPLOY_ROLE_ARN, ECS_CLUSTER_DEV,
ECS_SERVICE_DEV, ECS_TASK_DEF_DEV, and DEV_URL as secrets.
Only build the development pipeline for now — staging and
production will be added once this is validated.
```

**To enable manual triggering for demos:**
```
Add workflow_dispatch as an additional trigger to
deploy-development.yml so the workflow can be manually
triggered from the GitHub Actions UI.
```
