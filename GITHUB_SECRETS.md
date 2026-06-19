# GitHub Actions Secrets Reference

Configure all of these in your GitHub repository under:
**Settings → Secrets and variables → Actions → Repository secrets**

> **Note:** Only the development pipeline is active right now.
> Staging and production secrets will be added when those pipelines are enabled.

---

## Secrets Required — Development Pipeline

| Secret | Description | Example value / format |
|--------|-------------|------------------------|
| `DEV_DEPLOY_ROLE_ARN` | ARN of the IAM role GitHub Actions assumes via OIDC to deploy to development. See `OIDC_SETUP.md`. | `arn:aws:iam::123456789012:role/spikesignals-github-deploy-dev` |
| `ECR_REGISTRY` | ECR registry hostname — no trailing slash, no repository name, no image tag. | `123456789012.dkr.ecr.us-east-1.amazonaws.com` |
| `ECS_CLUSTER_DEV` | Name of the ECS cluster that runs the development service. | `spikesignals-dev` |
| `ECS_SERVICE_DEV` | Name of the ECS service to update for development. | `spikesignals-api-dev` |
| `ECS_TASK_DEF_DEV` | ECS task definition **family name** — no revision number. The workflow fetches the latest active revision automatically. | `spikesignals-api-dev` |
| `ECS_NETWORK_CONFIG` | Full JSON string for `--network-configuration` used by the migration one-off Fargate task. See format below. | See below |
| `DEV_URL` | Base URL for the dev environment — **no trailing slash**. Used for smoke test: `{DEV_URL}/health/` | `https://dev.spikesignals.com` |

---

## ECS_NETWORK_CONFIG Format

Paste as a single-line JSON string with no line breaks:

```json
{"awsvpcConfiguration":{"subnets":["subnet-xxxxxxxxxxxxxxxxx","subnet-yyyyyyyyyyyyyyyyy"],"securityGroups":["sg-xxxxxxxxxxxxxxxxx"],"assignPublicIp":"DISABLED"}}
```

- **`subnets`** — private subnets with a NAT Gateway (required so the task can reach AWS Secrets Manager during `collectstatic`)
- **`securityGroups`** — same security group as your main ECS tasks
- **`assignPublicIp`** — `"DISABLED"` for private subnets, `"ENABLED"` if using public subnets without NAT

---

## Secrets to Add Later (Staging & Production)

When the staging and production pipelines are enabled, add these:

| Secret | Description |
|--------|-------------|
| `STAGING_DEPLOY_ROLE_ARN` | IAM role ARN for staging deploys |
| `PRODUCTION_DEPLOY_ROLE_ARN` | IAM role ARN for production deploys |
| `ECS_CLUSTER_STAGING` | ECS cluster name for staging |
| `ECS_CLUSTER_PRODUCTION` | ECS cluster name for production |
| `ECS_SERVICE_STAGING` | ECS service name for staging |
| `ECS_SERVICE_PRODUCTION` | ECS service name for production |
| `ECS_TASK_DEF_STAGING` | Task definition family name for staging |
| `ECS_TASK_DEF_PRODUCTION` | Task definition family name for production |
| `STAGING_URL` | Base URL for staging smoke test |
| `PRODUCTION_URL` | Base URL for production smoke test |
