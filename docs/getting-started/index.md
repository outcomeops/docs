---
title: Getting Started
description: Deploy OutcomeOps AI Assist into your AWS account and connect your first integration.
---

# Getting Started

OutcomeOps AI Assist deploys as a set of Terraform modules that stand up Lambdas, DynamoDB tables, KMS keys, and a Fargate UI --- entirely in your AWS account. This section walks you from zero to a working deploy with your first workspace connected.

Read in order:

1. **[Deploy](deploy.md)** --- provision the platform with `terraform apply`. 30-60 minutes.
2. **[First Workspace](first-workspace.md)** --- create a workspace, connect the GitHub integration, ask your first question. 10-15 minutes.

## Before you start

You'll need:

- An AWS account with permission to create IAM roles, KMS keys, Lambdas, DynamoDB tables, S3 buckets, Fargate services, and Route 53 records
- Terraform 1.5+ locally or in CI
- AWS CLI authenticated to the target account
- Docker on the machine that builds the layers and container images
- Bedrock model access in your target region (Claude Sonnet + Haiku by default, or your compliance-approved substitutes)
- A license key from OutcomeOps ([procure via outcomeops.ai](https://outcomeops.ai))
- An Azure AD tenant for OIDC authentication on the UI
- A domain name you control for the UI subdomain

The [Deploy](deploy.md) page covers each of these in detail.

## After you finish

Once your platform is running:

- Enable more integrations from the [Integrations](../integrations/index.md) section
- Configure [integration gating](../administration/integration-gating.md) so SecOps can review exactly what's provisioned
- Explore [Features](../features/index.md) --- chat, workspaces, MCP servers, code generation
