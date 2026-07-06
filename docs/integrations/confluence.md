---
title: Confluence
description: Connect Confluence Cloud to OutcomeOps AI Assist for space page ingest.
---

# Confluence

The Confluence integration reads pages from the Atlassian spaces you connect and makes them retrievable in chat. Use it for design docs, ADR spaces, runbooks, onboarding wikis --- anything your team writes down that should be queryable alongside code.

Confluence and Jira share the same Atlassian OAuth app registration on your side. **If you already set up Jira**, most of this is done --- you just add Confluence-specific scopes and pick spaces.

Time budget: **~15 minutes**.

## What this connects

- **Confluence pages** in the spaces the connecting user has access to.
- **Page hierarchy is preserved** in the retrieval index so the platform can cite "under Space X → parent page Y → your page."
- **Comments are not ingested** in v1.
- **Attachments are not ingested** in v1 (linked docs from other integrations show up separately).

**Confluence Cloud only.** Confluence Server / Data Center (self-hosted) is not supported in v1.

## Prerequisites

- **Atlassian admin access** on the Atlassian org that owns your Confluence workspace.
- **Coordination with the platform operator.**
- **`enable_confluence_integration = true`** in the tfvars.

## Step 1: Register or reuse the Atlassian OAuth app

If you already have an Atlassian OAuth app registered for Jira, skip to Step 2 --- the same app works for both.

Otherwise:

1. Go to the **[Atlassian Developer Console](https://developer.atlassian.com/console/myapps/)**.
2. Click **Create** → **OAuth 2.0 integration**.
3. Name it `OutcomeOps AI Assist` (or similar).
4. On the app page, note:
    - **Client ID** (top of the "Settings" tab)
    - **Client Secret** (in the "Settings" tab; regenerate if needed)

## Step 2: Add Confluence scopes to the app

1. In the app, navigate to **Permissions** → **Confluence API** → **Add**.
2. Choose these scopes:
    - `read:page:confluence`
    - `read:space:confluence`
    - `read:content:confluence`
    - `read:user:confluence`
3. Save.

## Step 3: Configure the OAuth callback URL

1. In the app, navigate to **Authorization** → **OAuth 2.0 (3LO)** → **Add**.
2. Callback URL: `https://<your-app-fqdn>/api/confluence/callback`.
3. Save.

## Step 4: Handoff to the platform operator

The operator populates:

```hcl
enable_confluence_integration = true
atlassian_oauth_client_id     = "<the Client ID>"
```

And creates the SSM SecureString for the client secret:

```bash
aws ssm put-parameter \
  --name "/prd/outcome-ops-ai-assist/oauth/atlassian/client-secret" \
  --value "<the Client Secret>" \
  --type SecureString \
  --key-id alias/aws/ssm \
  --region us-west-2
```

Note the SSM parameter path is `/oauth/atlassian/` (shared with Jira), not Confluence-specific. If you already populated it for Jira, you don't populate it again.

Then `terraform apply` --- see the [Deploy guide](../getting-started/deploy.md).

## Step 5: Verify from the UI

1. Open the OutcomeOps UI as a **workspace admin**.
2. Navigate to **Workspace Settings → Integrations**.
3. Under Confluence, click **Connect**.
4. The browser opens Atlassian's consent screen. The user accepts.
5. The browser returns to Integrations with a green banner showing the connected user's display name.
6. Click **Add spaces**. The picker shows every Confluence space the connecting user can see.
7. Pick spaces and click **Add**. Sync status goes **pending → in_progress → success**.

Small spaces sync in a few minutes; large spaces (thousands of pages) take longer.

## Common problems

| Error | Cause | Fix |
| --- | --- | --- |
| `Redirect URI mismatch` | Callback URL in the Atlassian app doesn't match the deploy's FQDN exactly. | Re-check the callback URL in the Developer Console. Must be `https://<your-app-fqdn>/api/confluence/callback` --- no trailing slash. |
| `insufficient_scope` after consent | The Confluence scopes weren't added in Step 2. | Add the four Confluence scopes, save, and try connecting again. |
| `Space missing from picker` | Connecting user doesn't have permission on the space. | Give the user permission in Confluence, then click **Refresh** on the picker. |
| Content is stale hours after edit | Confluence sync runs hourly. | Click **Sync now** for that space, or wait for the next hourly tick. |

## Disconnecting

Workspace Settings → Integrations → **Disconnect Confluence**. This removes the connection + the tokens; ingested space content is deleted from the workspace's KB.
