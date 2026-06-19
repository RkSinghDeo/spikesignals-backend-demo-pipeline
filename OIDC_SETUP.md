# OIDC Setup — GitHub Actions IAM Roles

GitHub Actions deploys to AWS without long-lived access keys by using OIDC.
The GitHub Actions runner assumes an IAM role via a short-lived token that
is scoped to a specific branch or tag — it cannot be used by any other workflow.

> **Only the development role is needed right now.**
> Staging and production role configs are included at the bottom for when those pipelines are enabled.

---

## Step 1 — Create the GitHub OIDC Identity Provider (once per AWS account)

This only needs to be done **once**, regardless of how many roles you create.

**AWS Console:** IAM → Identity providers → Add provider

| Field | Value |
|-------|-------|
| Provider type | OpenID Connect |
| Provider URL | `https://token.actions.githubusercontent.com` |
| Audience | `sts.amazonaws.com` |

**Or via CLI:**
```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

---

## Step 2 — Create the Development Deploy Role

**Suggested role name:** `spikesignals-github-deploy-dev`

Replace every `UPPER_CASE` placeholder before pasting.

### Trust Policy

Locks this role to pushes on the `development` branch only.
A workflow on any other branch, tag, or PR cannot assume it.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:Bright-Access-Consulting-LLC/spikesignals-backend:ref:refs/heads/development"
        }
      }
    }
  ]
}
```

> **Confirm the repo slug casing.** The `sub` condition is case-sensitive.
> If the GitHub org or repo name differs, update the string above before creating the role.

### Permissions Policy

Replace `ACCOUNT_ID`, `CLUSTER_NAME`, `SERVICE_NAME`, `TASK_EXECUTION_ROLE_NAME`,
and `TASK_ROLE_NAME` with your actual values.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ECRAuth",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ECRPush",
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "arn:aws:ecr:us-east-1:ACCOUNT_ID:repository/spikesignals-api"
    },
    {
      "Sid": "ECSTaskDefinition",
      "Effect": "Allow",
      "Action": [
        "ecs:DescribeTaskDefinition",
        "ecs:RegisterTaskDefinition"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ECSRunAndUpdate",
      "Effect": "Allow",
      "Action": [
        "ecs:RunTask",
        "ecs:DescribeTasks",
        "ecs:UpdateService",
        "ecs:DescribeServices"
      ],
      "Resource": [
        "arn:aws:ecs:us-east-1:ACCOUNT_ID:cluster/CLUSTER_NAME",
        "arn:aws:ecs:us-east-1:ACCOUNT_ID:service/CLUSTER_NAME/SERVICE_NAME",
        "arn:aws:ecs:us-east-1:ACCOUNT_ID:task/CLUSTER_NAME/*",
        "arn:aws:ecs:us-east-1:ACCOUNT_ID:task-definition/spikesignals-api-dev:*"
      ]
    },
    {
      "Sid": "PassRoleForECS",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": [
        "arn:aws:iam::ACCOUNT_ID:role/TASK_EXECUTION_ROLE_NAME",
        "arn:aws:iam::ACCOUNT_ID:role/TASK_ROLE_NAME"
      ]
    }
  ]
}
```

> **Why `iam:PassRole` is required:** `aws ecs run-task` and `aws ecs register-task-definition`
> both require the caller to have permission to "pass" the task execution role and task role
> to the ECS service. Without this, the API returns `AccessDenied` even when all other
> ECS permissions are present — and it is one of the most common gotchas when setting this up.

---

## Step 3 — Pre-deploy Checklist

Before pushing to `development` for the first time:

- [ ] OIDC provider created in AWS IAM (Step 1)
- [ ] Dev deploy role created with the trust policy above
- [ ] `DEV_DEPLOY_ROLE_ARN` secret set in GitHub
- [ ] All 7 secrets from `GITHUB_SECRETS.md` configured in GitHub
- [ ] ECR repository `spikesignals-api` exists in the account
- [ ] Dev ECS cluster, service, and task definition exist (pipeline does not create infrastructure)
- [ ] `ECS_NETWORK_CONFIG` subnets have outbound internet access via NAT Gateway

---

## Staging & Production Roles (add when pipelines are enabled)

### Staging Trust Policy

Same as dev but scoped to the `staging` branch:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:Bright-Access-Consulting-LLC/spikesignals-backend:ref:refs/heads/staging"
        }
      }
    }
  ]
}
```

### Production Trust Policy

Scoped to semver tags only — no branch push can assume this role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:Bright-Access-Consulting-LLC/spikesignals-backend:ref:refs/tags/v*"
        }
      }
    }
  ]
}
```

The permissions policy for staging and production is identical in structure to the dev policy above — substitute the appropriate cluster name, service name, and task definition family name.

> **Production note:** When enabling the production pipeline, also create a `production`
> GitHub environment (Settings → Environments) with required reviewers configured.
> This provides the manual approval gate before any production deploy runs.
