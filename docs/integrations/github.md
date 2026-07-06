---
title: GitHub
description: Connect GitHub to OutcomeOps AI Assist via a GitHub App for repository ingest and PR checks.
---

# GitHub

The GitHub integration connects OutcomeOps AI Assist to your GitHub organization via a **GitHub App** (not personal OAuth), so access is org-scoped and revocable in one place. It ingests repository content for chat context, watches for issue and PR events, and posts PR-check results back as comments.

Time budget: **~20 minutes** if you have GitHub admin access on your org.

## What this connects

- **Repository content.** Files in default branches, ingested on a hourly sync cadence plus manual "Sync now" triggers.
- **Issues.** Optional. GitHub Issues can trigger code generation via labels similar to Jira Automation (see [Code Generation](../features/code-generation.md)).
- **Pull requests.** The platform can post PR-check results as thread comments (ADR compliance, README freshness, test coverage, breaking changes, architectural duplication, license compliance).

The GitHub App does **not**:

- Access repositories you don't explicitly install it on.
- Modify code without being asked (all writes happen through PRs that require your review).
- Access GitHub Actions secrets or organization-level settings.

## Prerequisites

- **GitHub organization admin** access (owner role) to create + install the GitHub App.
- **Coordination with the platform operator** who runs `terraform apply`. They need the App ID, client ID, private key, and webhook secret from you.
- **`enable_github_integration = true`** in the tfvars for the deploy (see [Integration Gating](../administration/integration-gating.md)).

## Public GitHub vs. GitHub Enterprise

Both are supported. The setup steps below reference `github.com` URLs; for GitHub Enterprise Server, substitute your instance's URL throughout (`https://ghe.your-company.com/organizations/...`), and set the corresponding tfvar so the platform's Lambdas talk to your GHES API instead of `api.github.com`.

## Step 1: Create the GitHub App

1. Sign in to GitHub as an organization owner.
2. Navigate to **your org → Settings → Developer settings → GitHub Apps → New GitHub App**.
3. Fill in:
    - **GitHub App name:** `OutcomeOps AI Assist - <yourorg>`. Names are globally unique across GitHub; suffix with your org name if needed.
    - **Homepage URL:** your OutcomeOps deploy's FQDN (e.g., `https://outcomeops.example.com`).
    - **Callback URL:** `https://<your-app-fqdn>/api/github/callback`.
    - **Webhook URL:** `https://<your-app-fqdn>/api/github/webhook`.
    - **Webhook secret:** generate a random string (the operator will store it in SSM). Save it --- GitHub only shows it once.
4. **Repository permissions** (set each to the listed level):
    - **Contents:** Read & write (needed to clone repos + open PRs)
    - **Issues:** Read & write (if you want issue-driven code generation; otherwise Read)
    - **Pull requests:** Read & write (post check comments)
    - **Metadata:** Read (auto-included)
    - **Actions:** Read (optional; enables PR-check workflow correlation)
5. **Subscribe to events** (checkbox list): Issues, Issue comment, Pull request, Pull request review comment, Push.
6. **Where can this GitHub App be installed?** Select **Only on this account** (your org only).
7. Click **Create GitHub App**.
8. On the resulting App settings page:
    - Copy the **App ID** (numeric).
    - Copy the **Client ID** (starts with `Iv23...`).
    - Click **Generate a private key**. GitHub downloads a `.pem` file. Save it --- you can generate more if lost, but each is one-shot.

## Step 2: Handoff to the platform operator

The operator needs these values:

| What | Where you got it | Goes into |
| --- | --- | --- |
| App ID (numeric) | Step 1.8 | `github_app_id` tfvar |
| App slug (URL name) | App settings page URL | `github_app_slug` tfvar |
| Client ID | Step 1.8 | `github_client_id` tfvar |
| Private key `.pem` | Step 1.8 | SSM SecureString |
| Webhook secret | Step 1.3 | SSM SecureString |

Tfvars:

```hcl
enable_github_integration = true
github_app_id             = "1234567"
github_app_slug           = "outcomeops-ai-assist-yourorg"
github_client_id          = "Iv23liXXXXXXXXXXXXXX"
```

The operator creates two SSM SecureString parameters:

```bash
aws ssm put-parameter \
  --name "/prd/outcome-ops-ai-assist/github/app-private-key" \
  --value "$(cat outcomeops-ai-assist.private-key.pem)" \
  --type SecureString \
  --key-id alias/aws/ssm \
  --region us-west-2

aws ssm put-parameter \
  --name "/prd/outcome-ops-ai-assist/github/webhook-secret" \
  --value "<the webhook secret from Step 1.3>" \
  --type SecureString \
  --key-id alias/aws/ssm \
  --region us-west-2
```

Then `terraform apply` --- see the [Deploy guide](../getting-started/deploy.md).

## Step 3: Install the App on your org

Once the operator confirms the deploy is live:

1. Go back to the App settings page in GitHub → **Install App** → your org.
2. Choose **All repositories** or **Only select repositories**. If your platform will only ever ingest a subset, select-only is cleaner from a SecOps perspective.
3. Click **Install**.

GitHub redirects to `https://<your-app-fqdn>/api/github/callback`, and the platform records the installation.

## Step 4: Add repositories to a workspace

1. Open the OutcomeOps UI as a **workspace admin**.
2. Navigate to **Workspace Settings → Integrations**.
3. Under GitHub, click **Add repositories**.
4. Pick from the list of repositories the App is installed on. Only repositories in the installation's allowed set appear.
5. Each added repo goes through **pending → queued → in_progress → success** as it clones, chunks, and embeds.

Small repos are queryable in 2-5 minutes. Large monorepos take longer; watch the Integrations panel for progress.

## Common problems

| Error | Cause | Fix |
| --- | --- | --- |
| `Webhook delivery failed: 401` | Webhook secret mismatch --- the value stored in SSM doesn't match what you set in the GitHub App. | Copy the webhook secret from GitHub App settings (or regenerate it), re-run the SSM `put-parameter` command with `--overwrite`. |
| `Installation not found` when adding repos | The App was installed on your org but the installation record didn't propagate to the platform (webhook delivery failure). | Go to the GitHub App's Advanced tab, find the failed `installation.created` delivery, click **Redeliver**. |
| `Repo missing from picker` | Repository isn't in the App installation's allowed set. | GitHub org owner: edit the installation → add the repo, or switch to **All repositories**. |
| `Rate limited` errors in sync | Large monorepo hits GitHub's REST API rate limits. | The App has a higher rate limit than personal tokens. If you're still hitting it, contact OutcomeOps to enable GitHub Apps concurrency tuning. |
| `Signature verification failed` on webhook | Old webhook secret cached in a Lambda cold-start. | Wait ~15 minutes for the next cold start, or force a Lambda redeploy via `terraform apply` (env-var change forces new deploy). |

## Rotating the private key

GitHub Apps support multiple active private keys concurrently, so rotation is safe:

1. GitHub App settings → **Generate a private key** → save the new `.pem`.
2. Hand the new key to the operator; they run `aws ssm put-parameter --overwrite` on the same parameter name.
3. Verify the integration still works (open a repo picker; confirm repos load).
4. **Then** delete the old key on the App settings page.

Rotating the webhook secret follows the same pattern --- regenerate in GitHub App settings, update SSM, verify, done. No `terraform apply` needed.

## Disconnecting

Two levels:

- **Remove a repo from a workspace:** Workspace Settings → Integrations → find the repo → **Remove**. This deletes the repo's ingested content from the workspace's KB. Reversible: add the repo back and it re-ingests.
- **Uninstall the App from your org:** GitHub org → Settings → GitHub Apps → uninstall. This immediately revokes all access. Existing ingested content stays in the platform's KB until the workspace admin removes it individually.
