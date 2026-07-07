---
title: ServiceNow
description: Ingest published ServiceNow Knowledge Base articles + attachments as chat context, without leaving the customer's AWS account.
---

# ServiceNow

The ServiceNow integration ingests **published Knowledge Base articles + their attachments** from your ServiceNow instance and makes them retrievable in chat. Ask "what's our password reset process for cloud-only accounts?" or "which article covers the office VPN client on macOS?" and the platform answers grounded in the article your service desk already maintains --- cited by article number, not paraphrased from a stale copy.

Time budget: **~5 minutes** for someone with ServiceNow admin + AWS admin access.

## What this connects

- **`kb_knowledge` records** with `workflow_state = published`. Draft, review, and retired articles are excluded.
- **Attachments** on those articles, filtered to a file-type allowlist (Office docs, PDF, Markdown, plaintext, CSV, HTML, JSON, YAML).
- **Delta-only re-syncs** every hour. The scheduler tracks each connection's `sys_updated_on` high-water mark so only articles that changed since last sync are re-walked.
- **Orphan cleanup.** Articles moved out of `published` or deleted on the ServiceNow side are removed from the workspace's KB on the next full walk.
- **NOT ingested:** incidents, service catalog tasks, problem records, CMDB, article ratings, article revision history.

## Design principles

- **Per-workspace OAuth.** Every customer's ServiceNow instance requires an OAuth app registered on their own instance. There is no shared "OutcomeOps ServiceNow app" to consent to --- the customer's SSO admin owns and can rotate the OAuth app entirely from their side.
- **KMS-encrypted secrets in DDB, not env-level SSM.** Because each workspace's OAuth app is distinct, the `client_secret` lives on the connection row encrypted with the customer's OAuth-tokens CMK. Nothing about ServiceNow lives at the deploy level.
- **KB read is user-scoped.** ServiceNow doesn't take a scope parameter on the authorize URL; the OAuth app inherits the requesting user's ACLs. To constrain what the platform can see, connect as a dedicated read-only user rather than a full ServiceNow admin.
- **HTML wrapped, not scraped.** The platform stores the article body verbatim (in a minimal `<html>` envelope with number + short description), not a summarized re-render. Retrieval hits the source ServiceNow copy.

## Prerequisites

- **`enable_servicenow_integration = true`** in your tfvars. That's the entire deploy step --- no SSM parameters to pre-populate.
- **ServiceNow admin access** on the target instance, specifically the ability to register an OAuth Application Registry entry. A ServiceNow "admin" role or equivalent.
- **A workspace admin** logged into the OutcomeOps UI. (This can be the same person as the ServiceNow admin.)

## Step 1: Enable the flag and deploy

```hcl
enable_servicenow_integration = true
```

```bash
cd terraform
terraform apply -var-file=prd.tfvars
```

This provisions the ServiceNow integration + sync + scheduler Lambdas, the sync SQS queue with a DLQ, an hourly EventBridge cron, and CloudWatch alarms wired to your existing alerts SNS topic. Both regions come up in parallel under the active/passive HA topology.

See the [Deploy guide](../getting-started/deploy.md).

## Step 2: Register an OAuth Application Registry entry

On the ServiceNow instance you want to connect, as a ServiceNow admin:

1. Navigate to **System OAuth → Application Registry**.
2. Click **New**. ServiceNow shows the **Select your application connection type** picker. Pick **OAuth - Authorization code grant**.
    - The other options --- Client credentials grant, JWT bearer grant, Resource owner password credential grant, and Third-party ID token --- are all machine-to-machine or password flows and won't produce the redirect back to the OutcomeOps UI that the platform's per-workspace OAuth needs.
3. Fill in:
    - **Name:** anything descriptive, e.g. `OutcomeOps AI Assist (prd)`.
    - **Client ID:** ServiceNow generates this.
    - **Client Secret:** ServiceNow generates this. This is the last time ServiceNow will show it in plaintext --- copy it now.
    - **Redirect URL:** `https://<your-outcomeops-fqdn>/api/servicenow/callback`. Exact match, no trailing slash, no query string.
    - **Refresh token lifespan:** default (100 days) is fine. The platform auto-rotates on each refresh where the instance supports it.
    - **Access token lifespan:** 30 minutes is fine; the sync worker refreshes on every invocation.
4. Save.

**Scopes.** ServiceNow's OAuth app inherits the requesting user's ACLs at authorize time, not a scope parameter. If you want the platform limited to KB read only, either register the OAuth app while signed in as a dedicated read-only user, or ask the workspace admin to authorize as one. A full ServiceNow admin authorizing will grant the OAuth app admin-level ACLs (which is more than the KB flow uses, but works fine).

## Step 3: Connect from the OutcomeOps UI

As a workspace admin:

1. Open the OutcomeOps UI and navigate to **Workspace Settings → Integrations**.
2. Find the **ServiceNow Knowledge Base** section. Click **Connect**.
3. Fill in the three fields:
    - **Instance URL:** `https://<subdomain>.service-now.com`. No trailing slash. Legacy `.servicenow.com` (without the hyphen) is accepted with a warning.
    - **Client ID:** paste from Step 2.
    - **Client Secret:** paste from Step 2.
4. Click **Save & Authorize.**

The platform KMS-encrypts the Client Secret with the workspace's OAuth-tokens CMK and stores it in DynamoDB. It then redirects your browser to ServiceNow's OAuth consent screen.

## Step 4: Authorize

On the ServiceNow OAuth consent screen, sign in (as a KB-reader user or your admin, depending on how you scoped Step 2) and click **Allow.**

ServiceNow redirects you back to the OutcomeOps UI, which:

- Exchanges the authorization code for access + refresh tokens.
- Captures the ServiceNow user identity so a later reconnect against a different user account is detectable.
- Enqueues an immediate sync so KB articles appear in chat without waiting for the top-of-hour scheduler tick.

Refresh the workspace page after ~30 seconds. The connection card shows `Status: success` when the first walk lands.

## Sync cadence

- **Hourly at :45 UTC** on the scheduler tick.
- **Delta-only after the first sync.** The scheduler passes the connection's stored `sys_updated_on` cursor to the sync worker, which walks only articles updated since.
- **Manual sync** from the workspace settings card (**Sync now**) starts from the current cursor.
- **Orphan cleanup** runs after a complete enumeration. Articles that disappeared from `published` are removed from the workspace's KB on the next full walk.

## Rate limiting

ServiceNow rate-limits per instance. The sync worker retries `429` responses with `Retry-After` backoff up to 3 attempts. If the instance sustains rate-limiting harder than that, the sync marks the run partial-success (or failed) and defers to the next scheduler tick with a fresh cursor. If it keeps happening, raise the instance's Table API rate limit (usually configurable per role) or contact ServiceNow support.

## Common problems

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Redirect URI mismatch` from ServiceNow | The Redirect URL on the Application Registry entry doesn't match `https://<your-fqdn>/api/servicenow/callback` byte-for-byte. | Verify: no trailing slash, no query string, case-sensitive subdomain. Update in **System OAuth → Application Registry** on ServiceNow. |
| Connection card says `reauth_required` | Refresh token was rejected. Usually the OAuth app was revoked/regenerated, or the user's ACLs changed. | Click **Connect** again, re-run the setup form. Existing article-mapped chat citations survive. |
| Connection card says `failed` with "ServiceNow rate limited" | Instance rate-limiting exceeds the sync worker's 3-attempt backoff. | Raise the Table API rate limit on the ServiceNow side, or wait for the next scheduler tick. |
| The ServiceNow section doesn't appear in Workspace Settings | Either the tfvar is off, or the UI Fargate task didn't refresh after the last apply. | Verify `enable_servicenow_integration = true` in your tfvars, apply, then force-redeploy the UI service. |
| Articles missing from chat | The user who authorized doesn't have KB read ACLs on those categories on ServiceNow. | Grant KB read to that user, or re-authorize as a different user with broader ACLs. |

## Rotating the Client Secret

1. On ServiceNow, regenerate the Client Secret on the Application Registry entry.
2. In the OutcomeOps UI, open the workspace's ServiceNow connection card and click **Connect** to re-run the setup form with the new value.

The same `connection_id` is reused (idempotent per instance URL), so no history loss.

## Disconnecting

Workspace Settings → Integrations → ServiceNow connection card → **Disconnect.** The connection record is deleted, and a fan-out delete purges all `workspaces/{ws}/servicenow/{connection_id}/*` S3 objects + vectors. The OAuth Application Registry entry on the ServiceNow side is untouched --- delete it from ServiceNow if you want to revoke the platform's grant fully.

## What the flag gates

Setting `enable_servicenow_integration = false` provisions no ServiceNow Lambdas, no SQS queues, no scheduler, no alarms, no UI section. SecOps can look at your tfvars and know exactly which surfaces exist. See [Integration Gating](../administration/integration-gating.md).

## Why this doesn't require SOC 2 from OutcomeOps

The platform runs entirely in your AWS account, and the ServiceNow OAuth app is registered on your ServiceNow instance. The Client Secret, refresh token, article contents, and article attachments never leave your control plane. There is no vendor-hosted control path in the ingest flow --- so your existing SOC 2 (or HIPAA, or FedRAMP) posture applies to the ServiceNow integration the same way it applies to the rest of your AWS estate. See the [FAQ](../faq.md) for the full compliance-posture explanation.
