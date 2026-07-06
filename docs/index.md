# OutcomeOps AI Assist

Self-hosted, regulated-grade AI code assistant. Runs entirely in your AWS account. Integrates with GitHub, Azure DevOps, Jira, Confluence, Microsoft 365, and your databases.

## Where to start

<div class="grid cards" markdown>

- :material-rocket-launch: **[Deploy](getting-started/deploy.md)**

    Terraform-based install into your own AWS account. 30 min for a first deploy.

- :material-account-group: **[First Workspace](getting-started/first-workspace.md)**

    Create a workspace, connect an integration, ask the platform your first question.

- :material-shield-lock: **[Integration Gating](administration/integration-gating.md)**

    Turn integrations on and off per deploy. Nothing you don't use gets provisioned.

- :material-server: **[MCP Servers](features/mcp-servers.md)**

    Extend the platform with your own tools or point at existing MCP catalogs.

</div>

## What OutcomeOps AI Assist is

An AI assistant that reads your organization's code, tickets, documents, and databases and answers questions grounded in what it finds. It runs in your AWS account -- your data never leaves your control.

**Key differences from SaaS AI assistants:**

- **You own the deploy.** Terraform apply, in your AWS account, with your KMS keys.
- **You own the audit trail.** Every query, retrieval, and tool call lands in your CloudWatch + DynamoDB.
- **You own the model choice.** Bring your own Bedrock model, or bring Anthropic direct.
- **Integrations gate individually.** Each is one tfvar; SecOps reviews what you provision.

## Get help

- Ask questions in the OutcomeOps Help workspace inside your own deploy (once you seed this content into it).
- File issues at [github.com/outcomeops/docs](https://github.com/outcomeops/docs).
