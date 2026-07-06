---
title: Azure DevOps
description: Connect Azure DevOps Repos and Boards to OutcomeOps AI Assist via Microsoft Entra ID OAuth.
---

# Azure DevOps

OutcomeOps AI Assist connects to Azure DevOps via Microsoft Entra ID OAuth. Two integrations ship independently but share the same Entra ID app registration:

- **Azure DevOps (Repos)** --- ingests source code from repos in your ADO organizations. Enables chat with grounded citations across your ADO codebases and posts PR-check results back as thread comments.
- **Azure DevOps (Boards)** --- ingests work items (User Stories, Bugs, Tasks, Epics, Features) as retrievable chat context.

Time budget: **~30 minutes** for an admin who has done one Entra ID app registration before, plus **~15 minutes** if you also want Boards.

## What is and isn't supported

- **Cloud-hosted `dev.azure.com/{org}` only.** Azure DevOps Server (on-prem TFS) is not supported.
- **Polling + manual triggers only.** The platform does not subscribe to ADO Service Hooks. Sync runs hourly, plus manual "Sync now" from the UI.
- **Repos + Boards.** ADO Wiki, Test Plans, and Pipelines are not ingested (except for the optional PR-check pipeline integration described below).

## Prerequisites

- **Azure tenant admin access.** You need to register an app and grant admin consent in your Entra ID directory.
- **Azure DevOps organization access.** To validate the end-to-end connection, you need read access to at least one project + repo in the ADO org you want the platform to see.
- **Coordination with the operator who runs `terraform apply`.** They need the tenant ID, client ID, redirect URI, and client secret from you.

## Step 1: Register an Entra ID application

1. Sign in to the **[Microsoft Entra admin center](https://entra.microsoft.com)** with an account that has `Application Administrator` or `Cloud Application Administrator` role.
2. Navigate to **Applications → App registrations → New registration**.
3. **Name:** `OutcomeOps AI Assist - Azure DevOps` (the exact name doesn't matter --- this is just how you'll find it later).
4. **Supported account types:** `Accounts in this organizational directory only (Single tenant)`. **Do not pick multi-tenant** --- the integration serves your users only.
5. **Redirect URI:**
    - Platform: **Web**
    - Value: `https://{your-app-fqdn}/api/azure-devops/callback`

    Replace `{your-app-fqdn}` with the FQDN of your OutcomeOps deploy. The path is exactly `/api/azure-devops/callback` --- no trailing slash, no query string. Entra ID does an exact-match validation on this URL during the OAuth callback; a trailing slash, `http://` instead of `https://`, or a host typo all produce a `redirect_uri mismatch` error.
6. Click **Register**.
7. On the resulting Overview page, copy these two values:
    - **Application (client) ID** --- a UUID
    - **Directory (tenant) ID** --- a UUID

## Step 2: Grant the Azure DevOps API permission

The platform calls Azure DevOps REST APIs on the connected user's behalf. This requires the `user_impersonation` delegated permission on the Azure DevOps resource (**not** Microsoft Graph --- they're separate permission surfaces).

1. In your app registration, navigate to **API permissions → Add a permission**.
2. Click the **APIs my organization uses** tab.
3. Search for `Azure DevOps`. You're looking for the entry with the exact name **Azure DevOps** and resource ID `499b84ac-1321-427f-aa17-267ca6975798`. This is Microsoft's well-known resource ID; it's the same value across every Entra ID tenant.

    If the "Azure DevOps" entry doesn't appear, someone in your tenant needs to sign in to Azure DevOps once for the service principal to provision in your tenant. A one-time visit to `https://dev.azure.com/{any-org-your-tenant-has}` from any account in your tenant resolves it.

4. Select **Delegated permissions** → check **`user_impersonation`** → click **Add permissions**.
5. Back on the API permissions list, click **Grant admin consent for {tenant name}** and confirm. The status column for `user_impersonation` should now read **Granted for {tenant name}**.

Without admin consent, users hitting the "Connect Azure DevOps" flow will see a consent prompt they cannot satisfy themselves --- they'd have to ask an admin, and that's per-user friction the integration is designed to avoid.

## Step 3: Create a client secret

1. Navigate to **Certificates & secrets → Client secrets → New client secret**.
2. **Description:** `OutcomeOps AI Assist`.
3. **Expires:** pick a value per your tenant's secret-rotation policy. Default Entra options are 6, 12, or 24 months. The platform reads the secret from SSM at request time; if the secret expires the integration stops working until you generate a new one, so **longer = fewer interruptions, shorter = more secure**. A 24-month secret with a calendar reminder at month 22 is a reasonable default.
4. Click **Add**.
5. **Immediately copy the Value column** --- not the Secret ID. Entra ID only shows the value once on this page; reload and it's gone forever.
6. Hand the value off to your platform operator via a secure channel (shared 1Password / Bitwarden item, sealed-secrets workflow). **Never email or DM the secret value.**

## Step 4: Handoff to the platform operator

The operator needs four values from you:

| What | From step | Goes into |
| --- | --- | --- |
| Tenant ID | 1.7 | `azure_devops_tenant_id` in tfvars |
| Client ID | 1.7 | `azure_devops_client_id` in tfvars |
| Redirect URI | 1.5 | `azure_devops_redirect_uri` in tfvars |
| Client secret value | 3.5 | SSM SecureString (see below) |

The operator populates the tfvars file (`prd.tfvars` for production, or the equivalent for your environment):

```hcl
enable_azure_devops_integration      = true
azure_devops_tenant_id               = "<tenant-id-uuid>"
azure_devops_client_id               = "<client-id-uuid>"
azure_devops_client_secret_ssm_param = "/prd/outcome-ops-ai-assist/azure-devops/client-secret"
azure_devops_redirect_uri            = "https://<your-app-fqdn>/api/azure-devops/callback"
```

The `enable_azure_devops_integration` flag gates the entire Azure DevOps Repos backend surface. See [Integration Gating](../administration/integration-gating.md) for the full flag list.

The SSM parameter name is the path that holds the client secret; the operator creates it manually. Terraform does not create or manage the secret value:

```bash
aws ssm put-parameter \
  --name "/prd/outcome-ops-ai-assist/azure-devops/client-secret" \
  --value "<the secret value you handed off>" \
  --type SecureString \
  --key-id alias/aws/ssm \
  --region us-west-2
```

Then the operator runs `terraform apply -var-file=prd.tfvars` --- see the [Deploy guide](../getting-started/deploy.md).

## Step 5: Verify from the UI

Once the deploy is live:

1. Open the OutcomeOps UI as a user who has **workspace_admin** role on the workspace you want to connect Azure DevOps to.
2. Navigate to **Workspace Settings → Integrations**.
3. Click **Connect Azure DevOps**.
4. The browser opens Microsoft's consent screen for the Entra ID app you registered. The user accepts; admin consent (granted in Step 2) means no second consent prompt.
5. The browser returns to Integrations with a green banner showing the connected user's display name and email.
6. Click **Add Repositories**. The picker walks: orgs → projects → repos. Pick a repo and click **Add**.
7. The repo appears in the workspace's repo list with sync status **pending → queued → in_progress → success**. Transitions take ~30 seconds for a small repo; several minutes for large monorepos.

    If sync stays `pending` for more than 5 minutes, click **Sync Now** to trigger immediately.

## Enabling Boards on an existing Repos integration

The Boards integration ingests work items (User Stories, Bugs, Tasks, Epics, Features) as searchable chat context --- the ADO equivalent of the Jira integration. It's a **separate integration** with independent OAuth consent, its own workspace-settings tab, and its own disconnect flow. It reuses the Entra ID app, client secret, and tfvars you already set up for Repos --- two new steps: add the `vso.work` API permission, and register a second redirect URI.

Time budget: **~15 minutes**.

### Add the `vso.work` API permission

1. In the same Entra ID app registration, go to **API permissions → Add a permission → APIs my organization uses → Azure DevOps**.
2. Choose **Delegated permissions** → check **`vso.work`** (Read your organization's work items and their metadata) → click **Add permissions**.
3. Back on the API permissions list, click **Grant admin consent for {tenant name}** → confirm.

### Add the second redirect URI

Entra ID validates redirect URIs literally; the Boards OAuth callback lands on a different path than Repos.

1. Same app → **Authentication → Redirect URIs → Add URI**.
2. Enter `https://{your-app-fqdn}/api/azure-devops-boards/callback` (mirrors the Repos redirect but with `-boards` in the path).
3. **Save**.

### Enable the Boards flag

In your tfvars, add:

```hcl
enable_azure_devops_boards_integration = true
```

That's the only new line. Tenant, client ID, and client-secret SSM param reuse the Repos values. The redirect URI env var the Lambda consumes is derived from your deploy's FQDN inside the Terraform module --- you don't type it in tfvars. The value you registered in Entra ID has to match exactly: `https://{your-app-fqdn}/api/azure-devops-boards/callback`.

Run `terraform apply -var-file=prd.tfvars`. The apply creates the Boards integration Lambda + Function URL, sync Lambda + SQS queue + DLQ, and scheduler Lambda + EventBridge rule.

The UI's Integrations tab now shows an "Azure DevOps (Boards)" section next to "Azure DevOps (Repos)". Click **Connect Azure DevOps Boards**, complete the OAuth consent screen (Microsoft asks for `vso.work` only, so the consent copy is scoped correctly), and pick projects from the dropdown.

The initial WIQL sync fires immediately; subsequent syncs run hourly. Work items become retrievable in chat within one scheduler tick.

## What Boards syncs vs. doesn't

- **Only open work items are re-synced each cycle.** Filter is `[System.State] NOT IN ('Closed', 'Done', 'Removed', 'Resolved')`. Once ingested, closed items stay in the KB indefinitely --- a bug you fix today stays retrievable ("what was the fix for X?").
- **Work items closed before Boards was connected are not ingested.** No retroactive backfill.
- **Initial sync is capped at the 5000 most recently changed work items per project.** Larger projects require an operator-run backfill.
- **v1 field set is fixed:** Title, State, WorkItemType, AssignedTo, Description, Repro Steps, Acceptance Criteria, Priority, Severity, Tags, Area, Iteration, Created, Changed, Reason, Id. Custom fields are a v2 feature.
- **Comments are included inline** on each work item's document.
- **Attachments are not ingested.**

## Optional: post PR-check results from an Azure Pipeline

If your team uses Azure Pipelines and wants PR-check results to post automatically on every PR build, add this step to your `azure-pipelines.yml`. The step invokes the platform's `analyze-pr` Lambda directly via the AWS CLI:

```yaml
- task: AWSShellScript@1
  displayName: 'Trigger OutcomeOps PR-check'
  condition: eq(variables['Build.Reason'], 'PullRequest')
  inputs:
    awsCredentials: 'OutcomeOps-PRCheck'
    regionName: 'us-west-2'
    scriptType: 'inline'
    inlineScript: |
      PAYLOAD=$(jq -n \
        --argjson pr_number "$(System.PullRequest.PullRequestId)" \
        --arg org  "$(OUTCOMEOPS_ADO_ORG)" \
        --arg proj "$(System.TeamProject)" \
        --arg repo "$(Build.Repository.Name)" \
        --arg conn "$(OUTCOMEOPS_CONNECTION_ID)" \
        '{
          pr_number: $pr_number,
          repository: ($org + "/" + $proj + "/" + $repo),
          provider: "azure_devops",
          connection_id: $conn,
          ado_org: $org,
          ado_project: $proj,
          ado_repo: $repo
        }')
      aws lambda invoke \
        --function-name prd-outcome-ops-ai-assist-analyze-pr \
        --payload "$PAYLOAD" \
        --cli-binary-format raw-in-base64-out \
        /tmp/response.json > /dev/null
      cat /tmp/response.json | jq .
```

Pipeline prerequisites:

- An `OutcomeOps-PRCheck` AWS service connection in your Azure DevOps project (Project Settings → Service connections → New → AWS), configured with an IAM identity that has `lambda:InvokeFunction` on the `*-analyze-pr` function ARN.
- The **AWS Toolkit for Azure DevOps** extension installed in your ADO organization (adds the `AWSShellScript@1` task; free from the Marketplace).

Pipeline variables:

- `OUTCOMEOPS_ADO_ORG` --- your ADO organization slug (the `{org}` in `dev.azure.com/{org}`).
- `OUTCOMEOPS_CONNECTION_ID` --- the connection ID visible in Workspace Settings → Integrations after Step 5. Opaque; not sensitive.

If you don't want to wire up Pipelines, users can trigger a PR check manually from the workspace's PR-check UI page.

## Common problems

| Error | Cause | Fix |
| --- | --- | --- |
| `OAuth callback failed: redirect_uri mismatch` | The redirect URI on the Entra ID app doesn't match exactly. | Re-check the app registration's redirect URI. Match scheme (`https`), host, and path character-for-character. No trailing slash. |
| `Approval required` / `Need admin approval` consent screen | Admin consent (Step 2.5) was not granted. | Go back to API permissions and click **Grant admin consent**. |
| `invalid_resource` from Microsoft | Wrong API was added in Step 2. | Re-check API permissions --- should be **Azure DevOps**, not **Microsoft Graph**. |
| `Connection lost: reconnect_required` | The user who connected hasn't used the integration in ~90 days; Entra ID expired the refresh token. | Any workspace admin clicks **Reconnect Azure DevOps** in Workspace Settings. |
| `Pull request comment not posting` | The connected user lacks PR-comment permission in the target repo. | Verify the user has Contributor on the project, then have an ADO admin re-run admin consent. |
| `Repo missing from picker` | The connected user lacks read access to that repo in ADO. | Grant the user read on the repo; the picker will see it on the next refresh. |
| `Client secret authentication failed` | Secret expired, or was pasted with leading/trailing whitespace into SSM. | Regenerate per Step 3 and re-run the SSM `put-parameter` command. |
| `AADSTS65001: The user or administrator has not consented` (Boards) | The `vso.work` permission wasn't granted admin consent. | Go back to the app registration and grant admin consent for `vso.work`. |

## Rotating the client secret

Calendar a reminder ~30 days before the client secret expires:

1. Generate a new secret in the Entra app (repeat Step 3). Don't delete the old one yet.
2. Hand the new value to the operator.
3. Operator runs the SSM `put-parameter --overwrite` command. The integration Lambda picks up the new value on its next cold start (within ~15 minutes of normal traffic) --- no `terraform apply` needed.
4. Confirm the integration still works: open the connection picker in the UI, confirm orgs load.
5. **Then** delete the old secret in the Entra app.

Don't delete the old secret before confirming the new one works. Worst case during rotation: SSM updates between two Lambda invocations and you see a few seconds of "old cached secret" --- harmless because both are valid.

## Disconnecting

Workspace Settings → Integrations → **Disconnect Azure DevOps**. The platform:

- Removes every repo (or Boards project) that was added through this connection. **The associated KB documents are also cleaned up --- this is irreversible.**
- Deletes the refresh + access tokens.
- Removes the connection record.

To rotate the connecting user (the original connector left the company), disconnect first, then any other workspace admin clicks **Connect Azure DevOps** and walks the OAuth flow under their own identity. Repos must be re-added against the new connection --- repos are tied to the connecting user's token.
