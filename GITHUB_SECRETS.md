# GitHub Actions — Connecting to AWS

This is the only configuration needed in GitHub after AWS is set up.
No webhooks, no integrations, no default branch changes required.
The pipeline triggers purely based on which branch receives a push —
GitHub Actions handles the rest automatically.

---

## How the Trigger Works

```
push to development  →  deploy-development.yml fires automatically
push to staging      →  deploy-staging.yml fires automatically  (when added)
push tag v*.*.*      →  deploy-production.yml fires automatically (when added)
```

Each workflow watches its own branch. GitHub's default branch setting
is irrelevant — do not change it just for this pipeline.

---

## Step 1 — Add These Secrets to GitHub

Go to your repository:
**Settings → Secrets and variables → Actions → New repository secret**

Add one secret at a time. The table below shows exactly where to find each value in AWS.

> **Note:** Only the development pipeline is active right now.
> Staging and production secrets will be added when those pipelines are enabled.

---

### Development Pipeline Secrets

| Secret | Where to find the value in AWS |
|--------|-------------------------------|
| `DEV_DEPLOY_ROLE_ARN` | AWS Console → IAM → Roles → find `spikesignals-github-deploy-dev` → copy the ARN at the top of the summary page. Format: `arn:aws:iam::123456789012:role/spikesignals-github-deploy-dev` |
| `ECR_REGISTRY` | AWS Console → ECR → Repositories → copy the URI from the top — **hostname only**, no repository name, no trailing slash. Format: `123456789012.dkr.ecr.us-east-1.amazonaws.com` |
| `ECS_CLUSTER_DEV` | AWS Console → ECS → Clusters → copy the cluster **name** (not the ARN). Example: `spikesignals-dev` |
| `ECS_SERVICE_DEV` | AWS Console → ECS → Clusters → click your cluster → Services tab → copy the service **name**. Example: `spikesignals-api-dev` |
| `ECS_TASK_DEF_DEV` | AWS Console → ECS → Task definitions → copy the **family name only** — no colon, no revision number. Example: `spikesignals-api-dev` not `spikesignals-api-dev:3` |
| `ECS_NETWORK_CONFIG` | See format below — built from your VPC subnet IDs and security group ID |
| `DEV_URL` | Your dev app's base URL with no trailing slash. Example: `https://dev.spikesignals.com` |

---

### ECS_NETWORK_CONFIG Format

Paste as a **single-line JSON string** with no line breaks.
Get the subnet IDs and security group ID from:
**AWS Console → VPC → Subnets** and **AWS Console → VPC → Security groups**

```json
{"awsvpcConfiguration":{"subnets":["subnet-xxxxxxxxxxxxxxxxx","subnet-yyyyyyyyyyyyyyyyy"],"securityGroups":["sg-xxxxxxxxxxxxxxxxx"],"assignPublicIp":"DISABLED"}}
```

- **`subnets`** — use private subnets that route outbound traffic through a NAT Gateway (the migration task needs to reach AWS Secrets Manager)
- **`securityGroups`** — use the same security group as your main ECS tasks
- **`assignPublicIp`** — `"DISABLED"` for private subnets with NAT, `"ENABLED"` for public subnets

---

## That's It

Once the 7 secrets above are saved, the pipeline is fully connected to AWS.
The next push to the `development` branch will trigger a full deploy automatically.

No webhooks to configure. No integrations to install. No default branch to change.

---

## Secrets to Add Later (Staging & Production)

When the staging and production pipelines are enabled, add these using the same process:

| Secret | Where to find the value in AWS |
|--------|-------------------------------|
| `STAGING_DEPLOY_ROLE_ARN` | IAM → Roles → `spikesignals-github-deploy-staging` → ARN |
| `PRODUCTION_DEPLOY_ROLE_ARN` | IAM → Roles → `spikesignals-github-deploy-production` → ARN |
| `ECS_CLUSTER_STAGING` | ECS → Clusters → staging cluster name |
| `ECS_CLUSTER_PRODUCTION` | ECS → Clusters → production cluster name |
| `ECS_SERVICE_STAGING` | ECS → Clusters → staging cluster → service name |
| `ECS_SERVICE_PRODUCTION` | ECS → Clusters → production cluster → service name |
| `ECS_TASK_DEF_STAGING` | ECS → Task definitions → staging family name (no revision) |
| `ECS_TASK_DEF_PRODUCTION` | ECS → Task definitions → production family name (no revision) |
| `STAGING_URL` | Staging app base URL, no trailing slash |
| `PRODUCTION_URL` | Production app base URL, no trailing slash |
