---
title: Microsoft 365
description: Connect Outlook, Teams, SharePoint, OneDrive, and OneNote via a single Entra ID app.
---

# Microsoft 365

OutcomeOps AI Assist connects to five Microsoft 365 surfaces --- **Outlook**, **Teams**, **SharePoint**, **OneDrive**, and **OneNote** --- through a single Entra ID app registration. Each surface has its own tfvar flag; you enable only the ones you actually use. All five share the same OAuth client ID and client secret.

Time budget: **~30 minutes** for the initial Entra ID setup, plus ~5 minutes per surface you enable.

## What each surface ingests

| Surface | Content |
| --- | --- |
| **Outlook** | Mail folders the connecting user picks (typically shared team mailboxes, not personal inbox) |
| **Teams** | Channel messages from the Teams + channels the user picks (not private chats) |
| **SharePoint** | Document library items from the sites the user picks |
| **OneDrive** | Files from the folders the user picks (personal or shared) |
| **OneNote** | Notebooks the user picks |

## Prerequisites

- **Azure tenant admin access** for the Entra ID app registration + admin consent.
- **Coordination with the platform operator.**
- **The tfvar flag** for each surface you want to enable (see [Integration Gating](../administration/integration-gating.md)).

## Step 1: Register a single Entra ID app for all M365 surfaces

1. Sign in to the **[Microsoft Entra admin center](https://entra.microsoft.com)** as an application administrator.
2. Navigate to **Applications → App registrations → New registration**.
3. **Name:** `OutcomeOps AI Assist - Microsoft 365`.
4. **Supported account types:** `Accounts in this organizational directory only (Single tenant)`.
5. **Redirect URIs:** you'll add these in Step 3 (there's one per surface). For now, leave blank.
6. Click **Register**.
7. On the Overview page, copy:
    - **Application (client) ID**
    - **Directory (tenant) ID**

## Step 2: Grant the Microsoft Graph API permissions

The M365 integrations use Microsoft Graph delegated permissions. Add whichever ones you need based on the surfaces you plan to enable:

| Surface | Required delegated permissions (Microsoft Graph) |
| --- | --- |
| **Outlook** | `Mail.Read`, `Mail.Read.Shared` |
| **Teams** | `ChannelMessage.Read.All`, `Team.ReadBasic.All`, `Channel.ReadBasic.All` |
| **SharePoint** | `Sites.Read.All`, `Files.Read.All` |
| **OneDrive** | `Files.Read.All`, `Files.Read.Selected` |
| **OneNote** | `Notes.Read.All` |

1. In your app registration → **API permissions → Add a permission → Microsoft Graph → Delegated permissions**.
2. Search for each permission from the table, check it, click **Add permissions**.
3. Repeat for each surface.
4. Click **Grant admin consent for {tenant name}** and confirm. All permissions should show **Granted for {tenant name}**.

If you skip admin consent, users hit a per-user consent prompt on their first connect --- SecOps friction you probably don't want.

## Step 3: Register redirect URIs for each enabled surface

Each M365 surface has a distinct OAuth callback URL. Add every URI for the surfaces you plan to enable:

| Surface | Redirect URI |
| --- | --- |
| Outlook | `https://<your-app-fqdn>/api/outlook/callback` |
| Teams | `https://<your-app-fqdn>/api/teams/callback` |
| SharePoint | `https://<your-app-fqdn>/api/sharepoint/callback` |
| OneDrive | `https://<your-app-fqdn>/api/onedrive/callback` |
| OneNote | `https://<your-app-fqdn>/api/onenote/callback` |

In the app registration → **Authentication → Add a platform → Web → Redirect URIs** → add each URL that applies.

## Step 4: Create a client secret

1. **Certificates & secrets → Client secrets → New client secret**.
2. Expires: 24 months is a reasonable default.
3. Copy the **Value** immediately (only shown once).
4. Hand off to the operator via a secure channel.

## Step 5: Handoff to the platform operator

Tfvars:

```hcl
# Shared: apply to all five M365 surfaces
microsoft_oauth_client_id = "<Client ID from Step 1.7>"

# Per-surface flags: enable only what you need
enable_outlook_integration    = true
enable_teams_integration      = true
enable_sharepoint_integration = false
enable_onedrive_integration   = false
enable_onenote_integration    = false
```

The tenant ID and client secret land in SSM:

```bash
aws ssm put-parameter \
  --name "/prd/outcome-ops-ai-assist/oauth/microsoft/client-secret" \
  --value "<the Client Secret from Step 4>" \
  --type SecureString \
  --key-id alias/aws/ssm \
  --region us-west-2
```

The SSM path is `/oauth/microsoft/` (shared across all five surfaces) --- you populate it once regardless of how many surfaces you enable.

Then `terraform apply` --- see the [Deploy guide](../getting-started/deploy.md).

## Step 6: Verify from the UI (per enabled surface)

For each surface you enabled:

1. Open the OutcomeOps UI as a **workspace admin**.
2. Navigate to **Workspace Settings → Integrations**.
3. Under the surface (e.g., "Outlook"), click **Connect**.
4. The browser opens Microsoft's consent screen scoped to just that surface's permissions --- e.g., Outlook shows mail access only, Teams shows channel access only. This scoped consent is why we register one Entra ID app but grant one permission set per surface.
5. Complete the consent, land back on Integrations with a green banner.
6. Click **Add ...** (folders / teams / sites / notebooks / mailboxes depending on the surface) and pick what to ingest.
7. Sync status transitions **pending → in_progress → success**.

## Common problems

| Error | Cause | Fix |
| --- | --- | --- |
| `Need admin approval` consent screen for a surface | Admin consent (Step 2.4) wasn't granted for that surface's permissions. | Go back to the app registration → API permissions → Grant admin consent. |
| `Redirect URI mismatch` on one surface but not others | You didn't register that surface's redirect URI in Step 3. | Add the URI, save, retry. |
| Only some content syncs (e.g., team channels missing) | The connecting user isn't a member of that team / doesn't have permission on that site / folder. | Grant the user permission in the M365 admin center, click Refresh in the picker. |
| SharePoint sync succeeds but 0 documents | The picker showed sites, but each site has no docs the user can read. | Verify site permissions --- delegated access means the user's read scope determines what the platform sees. |

## What each flag gates

Each flag independently provisions its own Lambda + SQS + scheduler surface. The shared Microsoft OAuth SSM params (`client_id` + `client_secret`) stay as long as **any** of the five flags is `true`. Set them all to `false` and the platform provisions nothing M365-related. See [Integration Gating](../administration/integration-gating.md).

## Disconnecting

Per surface: Workspace Settings → Integrations → **Disconnect \<surface\>**. Removes the connection, tokens, and ingested content for that surface only. The other four remain untouched.

Full uninstall: from the Entra admin center, delete the app registration. This revokes all consent across the tenant.
