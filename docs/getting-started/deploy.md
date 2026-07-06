---
title: Deploy
description: Deploy OutcomeOps AI Assist into your own AWS account using Terraform.
---

# Deploy

OutcomeOps AI Assist runs entirely in your AWS account. You deploy it with Terraform, using tfvars you control, backed by KMS keys you own. Nothing about the deploy involves an external OutcomeOps-hosted control plane --- your data never leaves your AWS account.

This page covers a first deploy end-to-end. Time budget: **30-60 minutes** for someone comfortable with AWS + Terraform.

## What gets provisioned

At a high level, `terraform apply` provisions:

- A set of **Lambda functions** (workspace management, chat, integrations, code generation) behind Function URLs with IAM authentication
- **DynamoDB tables** for workspaces, connections, audit logs, symbol graph, and analytics --- all with your KMS keys
- **S3 buckets + S3 Vectors** for the knowledge base + retrieval index
- **A Fargate service** for the chat UI (Application Load Balancer secured by Azure AD OIDC)
- **A Fargate service** for the [MCP catalog](../features/mcp-servers.md) (SonarQube, Snyk, and OutcomeOps-native MCP servers)
- **CloudWatch log groups** and metric filters
- **SNS topics** for cost alerts, database drift, and audit-critical events

Only the integrations you explicitly enable get provisioned. Every workspace integration (GitHub, Azure DevOps, Jira, Confluence, Microsoft 365, Database) is gated by its own `enable_<name>_integration` tfvar --- default `false`. Setting all of them to `false` provisions the shared core (workspace management + chat + UI) and nothing else. See [Integration Gating](../administration/integration-gating.md) for the full list.

## Prerequisites

Before you run `terraform apply` you need:

1. **An AWS account** with permissions to create IAM roles, KMS keys, Lambda functions, DynamoDB tables, S3 buckets, ECS services, ALBs, VPC resources, Route 53 records, and SSM parameters.
2. **Terraform 1.5+** on your local machine (or your CI runner).
3. **AWS CLI** authenticated to the target account (`aws sts get-caller-identity` should succeed).
4. **Docker** on the machine that runs `make pre-deploy` --- Lambda layers and Fargate container images are built locally and pushed to ECR before Terraform references them.
5. **Bedrock model access** enabled in your target AWS region for the models you plan to use. The defaults are Claude Sonnet 4.6 and Claude Haiku 4.5; you can override via tfvars for a compliance-driven model choice.
6. **A license key** issued by OutcomeOps. Contact [outcomeops.ai](https://outcomeops.ai) to procure one for your organization. The key is a signed JWT that the platform validates at Lambda cold start.
7. **An Azure AD tenant** for OIDC user authentication on the UI ALB. Any tenant works; the platform is compatible with Microsoft Entra ID (both single-tenant and multi-tenant app registrations).
8. **A domain name** you control (for the UI subdomain) and DNS access to add records.

## Step 1: Create a Terraform state bucket

Terraform stores its state in S3. Create the bucket manually before the first apply --- Terraform can't manage the bucket that stores its own state.

**One bucket per app + region.** Terraform workspaces (Step 4) namespace state within the bucket via `env:/dev/...` and `env:/prd/...` prefixes, so you don't need a separate bucket per environment.

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION="us-west-2"
APP="outcome-ops-ai-assist"
BUCKET="terraform-state-${ACCOUNT_ID}-${REGION}-${APP}"

aws s3api create-bucket --bucket "${BUCKET}" --region "${REGION}" \
  --create-bucket-configuration LocationConstraint="${REGION}"

aws s3api put-bucket-versioning --bucket "${BUCKET}" \
  --versioning-configuration Status=Enabled

aws s3api put-public-access-block --bucket "${BUCKET}" \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

## Step 2: Store the license key in SSM

The license key is sensitive. Put it in Systems Manager Parameter Store as a `SecureString`; Terraform reads only the parameter name, never the value.

```bash
aws ssm put-parameter \
  --name "/prd/outcome-ops-ai-assist/license/key" \
  --type SecureString \
  --value "<paste the JWT here>" \
  --region us-west-2
```

## Step 3: Prepare your `prd.tfvars`

Copy `terraform/prd.tfvars.example` to `terraform/prd.tfvars` and fill it in. A minimal file looks like:

```hcl
environment = "prd"
aws_region  = "us-west-2"

# License
outcomeops_license_layer_arn  = "arn:aws:lambda:us-west-2:<layer-account>:layer:prd-outcomeops-license-validator-v2-0-0:4"
outcomeops_license_ssm_param  = "/prd/outcome-ops-ai-assist/license/key"
outcomeops_license_server_url = "https://license.outcomeops.ai"

# Core
enable_workspaces       = true
initial_org_admin_email = "you@yourdomain.com"

# UI
deploy_ui   = true
ui_vpc_id   = "vpc-xxxxxxxxxxxxxxxxx"
ui_private_subnet_ids = [
  "subnet-xxxxxxxxxxxxxxxxx",
  "subnet-xxxxxxxxxxxxxxxxx",
  "subnet-xxxxxxxxxxxxxxxxx",
]
ui_container_image = "<account>.dkr.ecr.us-west-2.amazonaws.com/prd-outcome-ops-ai-assist-chat-ui:latest"

# OIDC
domain          = "example.com"
oidc_enabled    = true
oidc_client_id  = "<Azure AD app client ID>"
oidc_tenant_id  = "<Azure AD tenant ID>"

# Integrations (all default false -- opt in only what you use)
enable_github_integration     = true
enable_jira_integration       = false
enable_confluence_integration = false
enable_azure_devops_integration = false
```

The complete tfvars reference lives in `terraform/prd.tfvars.example` in your source distribution.

**Committing `prd.tfvars` is your call.** Many teams commit tfvars to a private ops repo alongside the OutcomeOps source --- it gives you PR review on infra changes and a clear audit trail of who changed what. Just make sure the repo is private and access-controlled, and keep the license JWT + any secrets out of it (those live in SSM, referenced by parameter name). The `.example` file is what OutcomeOps ships to seed your `prd.tfvars`.

## Step 4: Set up Terraform workspaces

Terraform's built-in workspace feature lets you keep one Terraform state per environment (dev, prd) using the same code + module tree. This is the cleanest way to test changes in a lower environment before rolling them to production.

```bash
cd terraform
terraform init

terraform workspace new dev
terraform workspace new prd

# Confirm both exist:
terraform workspace list
```

Each workspace gets its own state file in the S3 backend, keyed by workspace name. You select which one is active with `terraform workspace select <name>` before running plan or apply.

Recommended pattern:

- **dev** --- deploy first with `terraform apply -var-file=dev.tfvars`. Iterate here on tfvars changes, module upgrades, integration-flag flips. Lower-stakes for outages.
- **prd** --- roll validated changes forward with `terraform apply -var-file=prd.tfvars` after dev looks good.

The tfvars files reference the workspace name via `environment = "dev"` and `environment = "prd"`, which flows through resource names and IAM policies so nothing collides between the two.

Some teams instead run dev + prd in separate AWS accounts entirely (Control Tower / Organizations pattern) --- Terraform workspaces work fine there too; each account gets its own backend + workspace.

## Step 5: Build the Lambda layers and container images

Terraform references pre-built Lambda layers and Fargate container images by ECR tag. Build them before `terraform apply` or the apply will fail on missing artifacts.

One catch-all target builds everything (layers + images) and provisions its own ECR repositories on first run:

```bash
ENVIRONMENT=prd make pre-deploy
```

`make pre-deploy` runs `build-all-layers` (Lambda runtime, Terraform helper, platform-CA) followed by `build-all-images` (chat UI, MCP server, run-tests, Snyk MCP, database integration containers). The step is **idempotent** --- unchanged layers and images short-circuit, so it's safe to run before every apply.

First run takes 10-30 minutes depending on your network + Docker cache. Subsequent runs (on unchanged code) finish in under a minute.

Substitute `dev` or your environment name for `prd` if you're building for a non-production workspace.

## Step 6: Apply

```bash
cd terraform
terraform workspace select prd    # or dev
terraform apply -var-file=prd.tfvars
```

The first apply takes 10-20 minutes. Subsequent applies are typically 1-5 minutes.

## Step 7: Verify

Confirm the deploy is healthy with three checks:

```bash
# Manifest of enabled integrations -- unauthenticated read
curl -sS https://<your-ui-fqdn>/api/config/integrations | jq

# The Lambda-URL-backed workspace management is reachable
curl -sSI https://<your-ui-fqdn>/api/region/role

# Sign in to the UI in a browser
open https://<your-ui-fqdn>/
```

If all three succeed, you have a working deploy. Move on to [First Workspace](first-workspace.md) to create your first workspace and connect an integration.

## Bring your own certificate and DNS {#byo-cert}

Two DNS models are supported:

- **Terraform-managed Route 53.** If you own the parent zone (`example.com`) in Route 53 and want Terraform to manage the UI subdomain, set `domain = "example.com"` and Terraform will create an A record aliased at the UI ALB.
- **External DNS.** If your DNS is managed elsewhere (Cloudflare, Azure DNS, on-prem BIND), skip the Route 53 automation and create a `CNAME` from `<subdomain>.<your-domain>` to the ACM-verified ALB hostname yourself.

Bring-your-own-certificate is supported via ACM certificate ARN --- set the `ui_certificate_arn` tfvar to skip Terraform's ACM issuance step and use a pre-provisioned certificate.

## Common problems

- **Bedrock `AccessDeniedException` on first chat query.** Confirm model access is granted in the Bedrock console for the specific model IDs referenced in your tfvars (`bedrock_advanced_model_id`, `bedrock_basic_model_id`, `bedrock_embedding_model_id`). Model access is a per-region, per-model opt-in in AWS.
- **Fargate task stuck in `PROVISIONING`.** Usually a subnet capacity or ENI-limit issue. Check the ECS event log; expand the subnet's available IPs or move to a different subnet.
- **OIDC redirect loop on sign-in.** The Azure AD app registration's redirect URI must exactly match `https://<your-ui-fqdn>/oauth2/idpresponse`. Trailing slashes and `http://` vs `https://` matter.

If you hit a problem not covered here, [open an issue](https://github.com/outcomeops/docs/issues/new) and we'll extend this page.

## Next steps

- [Create your first workspace](first-workspace.md)
- [Enable your first integration](../integrations/index.md)
- [Configure integration gating](../administration/integration-gating.md) for SecOps review
