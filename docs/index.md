---
title: Home
description: Self-hosted, regulated-grade AI code assistant. Deploys entirely into your AWS account.
---

# OutcomeOps AI Assist

**Self-hosted, regulated-grade AI code assistant.** Runs entirely in your AWS account. Integrates with GitHub, Azure DevOps, Jira, Confluence, Microsoft 365, and your databases.

Your data never leaves your AWS account. Your KMS keys encrypt everything. Your IAM controls access. Your Bedrock models power the LLM calls. OutcomeOps ships you Terraform modules; you run them.

## Where to start

<div class="grid cards" markdown>

- :material-rocket-launch: **[Deploy](getting-started/deploy.md)**

    Terraform-based install into your own AWS account. 30-60 min for a first deploy.

- :material-account-group: **[First Workspace](getting-started/first-workspace.md)**

    Create a workspace, connect GitHub, ask your first question. 10-15 min.

- :material-shield-lock: **[Integration Gating](administration/integration-gating.md)**

    Every integration is one tfvar, default false. Nothing you don't enable gets provisioned. The story SecOps needs.

- :material-server: **[MCP Servers](features/mcp-servers.md)**

    Extend the platform with your own tools or point at existing MCP endpoints (SonarQube, AWS, custom).

- :material-chat: **[Chat](features/chat.md)**

    Streaming, workspace-scoped conversations with source citations for every claim.

- :material-source-pull: **[Code Generation](features/code-generation.md)**

    Turn Jira issues or GitHub Issues into pull requests. Plan-first, then generate.

</div>

## What OutcomeOps AI Assist is

An AI assistant that reads your organization's code, tickets, documents, and databases and answers questions grounded in what it finds. Every response cites the specific file, page, or ticket it drew from --- so users can inspect the source and reviewers can audit the chain.

Runs as a set of Lambda functions + a Fargate UI + DynamoDB tables + S3 vectors, all in your AWS account. Access is controlled by your Azure AD tenant via OIDC. Multi-region HA is supported.

## Key differentiators

- **You own the deploy.** Terraform apply, in your AWS account, with your KMS keys.
- **You own the audit trail.** Every query, retrieval, and tool call lands in your CloudWatch + DynamoDB. Optional Kinesis stream to your SIEM.
- **You own the model choice.** Bring your compliance-approved Bedrock model, or the defaults (Claude Sonnet + Haiku).
- **Integrations gate individually.** Each is one tfvar; SecOps reviews exactly what you provision.
- **Regulated-grade by design.** SOC 2 / HIPAA / FedRAMP-friendly (compliance is a property of your deploy; the platform's design supports it).

## Integrations

- [GitHub](integrations/github.md) --- repo content, issues, PR checks
- [Azure DevOps](integrations/azure-devops.md) --- Repos and Boards
- [Jira](integrations/jira.md) --- issue context + code generation
- [Confluence](integrations/confluence.md) --- space pages
- [Microsoft 365](integrations/microsoft-365.md) --- Outlook, Teams, SharePoint, OneDrive, OneNote
- [Database](integrations/database.md) --- PostgreSQL, MySQL, SQL Server

Every integration ships behind a `enable_<name>_integration` tfvar, default `false`. See [Integration Gating](administration/integration-gating.md).

## Get help

- **Documentation questions:** you're in the right place. Use the search in the top bar, or start with [Getting Started](getting-started/index.md).
- **Product support (deployment issues, bugs, feature requests):** open a ticket at the [OutcomeOps Support Portal](https://outcomeops.atlassian.net/servicedesk/customer/portal/1). Your account team is on the other side.
- **Documentation errors:** [file an issue](https://github.com/outcomeops/docs/issues/new). Include the page URL and what's confusing.
