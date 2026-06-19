# OIDC Setup — One-Time AWS Configuration

All environments (development, staging, production) live in the same AWS account.
This means the OIDC provider is created **once**, and the same IAM policy template
is reused for each environment role — just with different resource names.

---

## Step 1 — Create the OIDC Provider (once, never again)

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

Done. Never touch this again.

---

## Step 2 — Create One Role Per Environment

Repeat this step 3 times — once for dev, staging, and production.
The only things that change between roles are highlighted with `← change this`.

### Trust Policy

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
          "token.actions.githubusercontent.com:sub": "repo:ORG/REPO:ref:REF_PATTERN"
        }
      }
    }
  ]
}
```

Fill in `REF_PATTERN` per environment:

| Environment | Role name | `REF_PATTERN` |
|-------------|-----------|---------------|
| Development | `spikesignals-github-deploy-dev` | `refs/heads/development` |
| Staging | `spikesignals-github-deploy-staging` | `refs/heads/staging` |
| Production | `spikesignals-github-deploy-production` | `refs/tags/v*` |

> Each role can only be assumed by its own branch or tag.
> A workflow on `development` cannot assume the staging or production role.

---

### Permissions Policy (same for all three roles)

Replace `ACCOUNT_ID`, `CLUSTER_NAME`, `SERVICE_NAME`,
`TASK_EXECUTION_ROLE_NAME`, and `TASK_ROLE_NAME` with the values
for that environment before attaching.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ECRAuth",
      "Effect": "Allow",
      "Action": ["ecr:GetAuthorizationToken"],
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
        "arn:aws:ecs:us-east-1:ACCOUNT_ID:task-definition/*"
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

---

## Step 3 — Add Role ARNs as GitHub Secrets

Once each role is created, copy its ARN into GitHub:
**Settings → Secrets and variables → Actions**

| Secret name | Value |
|-------------|-------|
| `DEV_DEPLOY_ROLE_ARN` | ARN of `spikesignals-github-deploy-dev` |
| `STAGING_DEPLOY_ROLE_ARN` | ARN of `spikesignals-github-deploy-staging` |
| `PRODUCTION_DEPLOY_ROLE_ARN` | ARN of `spikesignals-github-deploy-production` |

---

## Full Checklist

- [ ] OIDC provider created in IAM (Step 1 — done once)
- [ ] Dev role created — trust policy with `refs/heads/development`
- [ ] Staging role created — trust policy with `refs/heads/staging`
- [ ] Production role created — trust policy with `refs/tags/v*`
- [ ] All 3 role ARNs added as GitHub secrets
- [ ] Remaining secrets from `GITHUB_SECRETS.md` configured
- [ ] GitHub `production` environment created with required reviewers
