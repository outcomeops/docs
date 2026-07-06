---
title: Integration Gating
description: Every integration is a single tfvar. Nothing you don't enable exists in your AWS account.
---

# Integration Gating

Every workspace integration in OutcomeOps AI Assist ships behind a **platform-level tfvar** that defaults to `false`. Setting the flag to `true` provisions the entire backend surface for that integration --- Lambdas, Function URLs, SQS queues, dead-letter queues, EventBridge rules, SSM parameters, IAM policies. Setting it to `false` (or leaving the default) provisions nothing.

This is the SecOps story:

> **Nothing you don't enable exists in your AWS account.** A reviewer can look at your tfvars and know exactly which surface has been provisioned. Terraform apply provisions the shared core (workspace management, chat, UI) plus whatever integration flags are true --- and only those.

## Why this matters

Before per-integration gating landed, the platform provisioned every integration when `enable_workspaces = true` fired. Regulated buyers pushed back: "we don't use Confluence; why does its Lambda exist in our account?" The answer used to be "because the platform is monolithic." The answer now is "it doesn't --- flip `enable_confluence_integration = false` and it's gone."

Regulated customers, air-gapped deploys, and SecOps reviewers all benefit from the same property: **zero cost, zero attack surface, zero audit noise** for integrations you don't use.

## Flags

| Flag | Default | What provisions when true |
| --- | --- | --- |
| `enable_azure_devops_integration` | `false` | [Azure DevOps (Repos)](../integrations/azure-devops.md) --- integration Lambda + Function URL + sync + scheduler + SQS + DLQ + EventBridge |
| `enable_azure_devops_boards_integration` | `false` | [Azure DevOps (Boards)](../integrations/azure-devops.md#enabling-boards-on-an-existing-repos-integration) --- work items ingestion |
| `enable_github_integration` | `false` | [GitHub App](../integrations/github.md) --- integration Lambda + Function URL + sync + scheduler + SQS + DLQ + App-credential SSM parameters |
| `enable_github_issue_integration` | `true` | Code-generation API Gateway v2 (public HTTP endpoint that receives GitHub webhooks) + Lambda trigger permission. Enables the GitHub-Issues code-generation trigger. |
| `enable_confluence_integration` | `false` | [Confluence](../integrations/confluence.md) --- integration Lambda + Function URL + sync + scheduler + SQS + DLQ + Confluence callback SSM |
| `enable_jira_integration` | `false` | [Jira](../integrations/jira.md) --- workspace Jira Lambdas + Function URL + sync + scheduler + SQS + DLQ + Jira callback SSM + SSM Automation cross-account role + document |
| `enable_outlook_integration` | `false` | [Outlook](../integrations/microsoft-365.md) --- integration Lambda + Function URL + sync + scheduler + SQS + DLQ + Outlook callback SSM |
| `enable_teams_integration` | `false` | [Teams](../integrations/microsoft-365.md) --- same surface, Teams callback SSM |
| `enable_sharepoint_integration` | `false` | [SharePoint](../integrations/microsoft-365.md) --- same surface, SharePoint callback SSM |
| `enable_onedrive_integration` | `false` | [OneDrive](../integrations/microsoft-365.md) --- same surface, OneDrive callback SSM |
| `enable_onenote_integration` | `false` | [OneNote](../integrations/microsoft-365.md) --- same surface, OneNote callback SSM |
| `enable_database_integration` | `false` | [Database](../integrations/database.md) --- integration Lambda (container image) + Function URL + sync + scheduler + SQS + DLQ + drift-notification SNS topic |

## Shared SSM parameters

Some SSM parameters back multiple integrations. They provision when **any** of the sharing integrations is enabled:

- **Atlassian OAuth (`atlassian_oauth_client_id` + SSM secret)** --- provisioned when `enable_confluence_integration` OR `enable_jira_integration` is `true`.
- **Microsoft OAuth (`microsoft_oauth_client_id` + SSM secret)** --- provisioned when any of `enable_outlook_integration`, `enable_teams_integration`, `enable_sharepoint_integration`, `enable_onedrive_integration`, or `enable_onenote_integration` is `true`.

So you don't populate the shared credentials again when adding a second Atlassian-family or Microsoft-family integration.

## To enable an integration

1. Add `enable_<name>_integration = true` to your `tfvars` file.
2. Populate any integration-specific tfvars and SSM parameters (see the per-integration setup guide under [Integrations](../integrations/index.md)).
3. Run `terraform apply -var-file=<env>.tfvars` via your CI/CD path.
4. **Verify:** the workspace-management Lambda's `GET /api/config/integrations` manifest returns the flag as `true`. The UI's Workspace Settings → Integrations tab renders the integration's section once the manifest reload completes.

## To disable an integration

1. Set `enable_<name>_integration = false` (or remove the line --- the default is `false`).
2. Run `terraform apply`. The plan shows destruction of the integration's Lambdas, SQS queues, EventBridge rules, and SSM parameters. Shared OAuth parameters stay if another sharing integration is still enabled.
3. **Verify:** the manifest returns `false` and the UI section is hidden.

**Existing workspace connections are NOT auto-deleted** when the flag flips off. The connection rows stay in DynamoDB until a workspace admin manually disconnects each one. Users see the connection state in the UI but can't ingest new content because the backend Lambdas are gone.

## Checking current state on a live deploy

```bash
curl -sS https://<your-deploy-fqdn>/api/config/integrations | jq
```

Returns:

```json
{
  "azure_devops": true,
  "azure_devops_boards": true,
  "github": true,
  "github_issues": true,
  "confluence": false,
  "jira": true,
  "outlook": false,
  "teams": false,
  "sharepoint": false,
  "onedrive": false,
  "onenote": false,
  "database": true
}
```

The endpoint is unauthenticated --- SecOps can check the manifest from anywhere. It only reveals which integrations are provisioned, not any credentials or content.

## SecOps review checklist

When your SecOps reviewer opens your tfvars, they should be able to confirm:

- [ ] Each `enable_*_integration` flag matches a documented business need.
- [ ] Flags they don't recognize are set to `false` (or absent, since the default is `false`).
- [ ] SSM parameter names for OAuth credentials are documented.
- [ ] The list of enabled integrations matches `curl .../api/config/integrations` on the live deploy.

Any drift between tfvars and the live manifest is a bug --- either the tfvars weren't applied, or the manifest env vars weren't wired correctly. File a support issue.

## FAQ

**Q: What happens if I flip a flag off that has existing workspace connections?**
The backend Lambdas are destroyed. Existing DynamoDB connection rows survive but become inert. Users see the connection state in the UI but can't ingest new content, and background sync stops running. Delete the connections from Workspace Settings when you're ready.

**Q: Can I re-enable an integration I previously disabled?**
Yes. Flip the flag back to `true`, apply Terraform. The Lambdas re-provision, sync resumes for any surviving connections, and existing workspace repos + projects resume being queryable.

**Q: Does disabling `enable_github_issue_integration` break the workspace GitHub App?**
No. The two flags gate different surfaces. `enable_github_integration` gates the workspace GitHub App (repo ingestion + PR checks + workspace connections). `enable_github_issue_integration` gates only the API Gateway v2 that receives GitHub webhooks for issue-driven code generation. Disable the latter if you don't want a public HTTP endpoint but still want repo ingestion.

**Q: Are there costs for provisioned-but-unused integrations?**
Yes --- Lambda functions incur a minimal storage cost, SQS queues have no idle cost, but scheduler EventBridge rules fire hourly (which fires Lambda invocations that immediately no-op when there's no work). Total cost of a provisioned-but-unused integration is a few dollars per month. Not zero, but not a factor in real deploys.

## Related

- [Deploy](../getting-started/deploy.md) --- where the tfvars go.
- Per-integration setup guides --- [Azure DevOps](../integrations/azure-devops.md), [GitHub](../integrations/github.md), [Jira](../integrations/jira.md), [Confluence](../integrations/confluence.md), [Microsoft 365](../integrations/microsoft-365.md), [Database](../integrations/database.md).
