---
title: ServiceNow
description: Ingest published ServiceNow Knowledge Base articles + attachments as chat context, without leaving the customer's AWS account.
---

# ServiceNow

The ServiceNow integration ingests **published Knowledge Base articles + their attachments** from your ServiceNow instance and makes them retrievable in chat. Ask "what's our password reset process for cloud-only accounts?" or "which article covers the office VPN client on macOS?" and the platform answers grounded in the article your service desk already maintains --- cited by article number, not paraphrased from a stale copy.

Time budget: **~10 minutes** for someone with ServiceNow admin + AWS admin access.

## What this connects

- **`kb_knowledge` records** with `workflow_state = published`. Draft, review, and retired articles are excluded.
- **Attachments** on those articles, filtered to a file-type allowlist (Office docs, PDF, Markdown, plaintext, CSV, HTML, JSON, YAML).
- **Delta-only re-syncs** every hour. The scheduler tracks each connection's `sys_updated_on` high-water mark so only articles that changed since last sync are re-walked.
- **Orphan cleanup.** Articles moved out of `published` or deleted on the ServiceNow side are removed from the workspace's KB on the next full walk.
- **NOT ingested:** incidents, service catalog tasks, problem records, CMDB, article ratings, article revision history.

## Design principles

- **One OAuth app per deployment.** Same pattern as every other OAuth integration --- register the app on ServiceNow once, drop the client ID in tfvars and the client secret in SSM, then every workspace on the deploy connects against it.
- **Read-only in effect.** ServiceNow's OAuth app inherits the requesting user's ACLs. Connect as a dedicated read-only ServiceNow user to constrain what the platform can see; or connect as an admin if you want the KB write-audit trail to just show up as one identity.
- **HTML wrapped, not scraped.** The platform stores the article body verbatim (in a minimal `<html>` envelope with number + short description), not a summarized re-render. Retrieval hits the source ServiceNow copy.

## Prerequisites

- **`enable_servicenow_integration = true`** in your tfvars, plus the two configuration variables below.
- **ServiceNow admin access** on the target instance, specifically the ability to register an OAuth Application Registry entry.
- **A workspace admin** logged into the OutcomeOps UI. (Can be the same person as the ServiceNow admin.)

## Step 1: Register an OAuth Application Registry entry (ServiceNow admin)

On the ServiceNow instance you want to connect:

1. Navigate to **System OAuth → Application Registry**.
2. Click **New**. ServiceNow shows the **Select your application connection type** picker. Pick **OAuth - Authorization code grant**.
    - The other options --- Client credentials grant, JWT bearer grant, Resource owner password credential grant, and Third-party ID token --- are all machine-to-machine or password flows and won't produce the redirect back to the OutcomeOps UI.
3. Fill in:
    - **Name:** anything descriptive, e.g. `OutcomeOps AI Assist (prd)`.
    - **Client ID:** ServiceNow generates this.
    - **Client Secret:** ServiceNow generates this. This is the last time ServiceNow will show it in plaintext --- copy it now.
    - **Redirect URL:** `https://<your-outcomeops-fqdn>/api/servicenow/callback`. Exact match, no trailing slash, no query string.
    - **Refresh token lifespan:** default (100 days) is fine.
    - **Access token lifespan:** 30 minutes is fine.
4. Save.

## Step 2: Put the client secret in SSM

The platform reads the client_secret from SSM SecureString at cold start. Put it in there:

```bash
aws ssm put-parameter \
  --name "/prd/outcome-ops-ai-assist/oauth/servicenow/client-secret" \
  --value "<the client_secret from Step 1>" \
  --type SecureString \
  --overwrite \
  --region us-west-2
```

If you're running HA (secondary region), mirror the value there too (or use `scripts/ha-sync-secrets`).

## Step 3: Set the tfvars

In your `terraform/prd.tfvars`:

```hcl
enable_servicenow_integration = true

# Your ServiceNow instance URL (no trailing slash)
servicenow_instance_url = "https://acme.service-now.com"

# The Client ID from Step 1 (public, not a secret)
servicenow_oauth_client_id = "..."
```

## Step 4: Enable and deploy

```bash
cd terraform
terraform apply -var-file=prd.tfvars
```

This provisions the ServiceNow integration + sync + scheduler Lambdas, the sync SQS queue with a DLQ, an hourly EventBridge cron, and CloudWatch alarms. Both regions come up in parallel under the active/passive HA topology.

See the [Deploy guide](../getting-started/deploy.md).

## Step 5: Connect from the UI (workspace admin)

Because the OAuth app is env-configured, the workspace admin's job is a single click:

1. Open the OutcomeOps UI and navigate to **Workspace Settings → Integrations**.
2. Find the **ServiceNow Knowledge Base** section. Click **Connect**.
3. Your browser lands on the ServiceNow OAuth consent screen. Sign in (as a KB-reader user or your admin) and click **Allow**.
4. ServiceNow redirects you back to the OutcomeOps UI. The connection card shows `Status: queued`, then flips to `success` once the first sync completes (~1 minute).

Same UX pattern as Confluence, Jira, and Microsoft 365 --- no separate setup form.

## Sync cadence

- **Hourly at :45 UTC** on the scheduler tick.
- **Delta-only after the first sync.** The scheduler passes the connection's stored `sys_updated_on` cursor to the sync worker.
- **Manual sync** from the workspace settings card (**Sync now**) starts from the current cursor.
- **Orphan cleanup** runs after a complete enumeration. Articles that disappeared from `published` are removed from the workspace's KB.

## Rate limiting

ServiceNow rate-limits per instance. The sync worker retries `429` responses with `Retry-After` backoff up to 3 attempts. If the instance sustains rate-limiting harder than that, the sync marks the run partial-success (or failed) and defers to the next scheduler tick with a fresh cursor. If it keeps happening, raise the instance's Table API rate limit (usually configurable per role) or contact ServiceNow support.

## Common problems

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Redirect URI mismatch` from ServiceNow | The Redirect URL on the Application Registry entry doesn't match `https://<your-fqdn>/api/servicenow/callback` byte-for-byte. | Verify: no trailing slash, no query string, case-sensitive subdomain. Update in **System OAuth → Application Registry** on ServiceNow. |
| Integration Lambda returns 500 "ServiceNow integration not fully configured" | One of `servicenow_instance_url` / `servicenow_oauth_client_id` is missing from tfvars, or the client-secret SSM param is still `PLACEHOLDER_CONFIGURE_ME`. | Verify tfvars are populated, `terraform apply`, and re-run the `aws ssm put-parameter` from Step 2. |
| Connection card says `reauth_required` | Refresh token was rejected. Usually the OAuth app was revoked/regenerated, or the user's ACLs changed. | Click **Connect** again. Existing article-mapped chat citations survive. |
| Connection card says `failed` with "ServiceNow rate limited" | Instance rate-limiting exceeds the sync worker's 3-attempt backoff. | Raise the Table API rate limit on the ServiceNow side, or wait for the next scheduler tick. |
| The ServiceNow section doesn't appear in Workspace Settings | Either the tfvar is off, or the UI Fargate task didn't refresh after the last apply. | Verify `enable_servicenow_integration = true` in your tfvars, apply, then force-redeploy the UI service. |
| Articles missing from chat | The user who authorized doesn't have KB read ACLs on those categories on ServiceNow. | Grant KB read to that user, or re-authorize as a different user with broader ACLs. |

## Rotating the Client Secret

1. On ServiceNow, regenerate the Client Secret on the Application Registry entry.
2. Run `aws ssm put-parameter --overwrite` on `/prd/outcome-ops-ai-assist/oauth/servicenow/client-secret` with the new value. Mirror to your secondary region.
3. Wait for the next Lambda cold start (~5 minutes idle) or invoke the integration Lambda once to force one. No `terraform apply` needed --- the SSM parameter uses `lifecycle.ignore_changes = [value]`.

## Disconnecting

Workspace Settings → Integrations → ServiceNow connection card → **Disconnect.** The connection record is deleted and a fan-out delete purges all `workspaces/{ws}/servicenow/{connection_id}/*` S3 objects + vectors. The OAuth Application Registry entry on the ServiceNow side is untouched.

## What the flag gates

Setting `enable_servicenow_integration = false` provisions no ServiceNow Lambdas, no SQS queues, no scheduler, no alarms, no UI section, no SSM parameters. SecOps can look at your tfvars and know exactly which surfaces exist. See [Integration Gating](../administration/integration-gating.md).
