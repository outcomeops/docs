---
title: GitLab
description: Connect GitLab to OutcomeOps AI Assist via a group-owned OAuth application.
---

# GitLab

OutcomeOps AI Assist connects to GitLab as a full peer to GitHub and Azure DevOps: it ingests source code, ADRs, and docs from your GitLab projects, powers chat with grounded citations across them, and posts merge-request check results back as MR notes.

The auth model is **one OAuth application per deployment, per-user delegated tokens**. A GitLab admin registers a single application; each user clicks "Connect GitLab" once and gets a token scoped to *their own* permissions --- users can only enroll projects they already have access to, and every connection carries per-user attribution. No per-user app registration, no shared group token with org-wide reach.

Time budget: **~15 minutes** for the app registration and handoff.

## What is and isn't supported

- **gitlab.com (SaaS) is the certified target.** Self-managed GitLab is configurable via the `gitlab_base_url` tfvar but not yet certified end-to-end.
- **Nested subgroups are fully supported.** Project paths of any depth (`group/subgroup/subsubgroup/project`) work everywhere --- enrollment, ingest, chat citations, and MR checks.
- **Polling + manual triggers only.** Sync runs hourly, plus manual "Sync Now" from the UI. The platform does not subscribe to GitLab webhooks.
- **Merge-request analysis** posts the same six AI checks as GitHub/ADO PRs, as collapsed MR notes that update in place on every push (no duplicate comments).

## Prerequisites

- **Owner or Maintainer access to a top-level GitLab group** (to register the group-owned OAuth application).
- **Read access to at least one project** to validate the end-to-end connection.
- **Coordination with the operator who runs `terraform apply`.** They need the application ID and secret from you.

## Step 1: Register the OAuth application

1. In GitLab, go to your **top-level group → Settings → Applications**. (Any top-level group works --- the registration location only determines who *manages* the credentials. The token's reach is always the authorizing user's own permissions across the instance. On self-managed GitLab, prefer an instance-wide application under **Admin Area → Applications**.)
2. **Name:** `OutcomeOps AI Assist`.
3. **Redirect URI:** `https://{your-app-fqdn}/api/gitlab/callback` --- exactly this path, no trailing slash. GitLab validates it literally during the OAuth callback.
4. **Confidential:** checked (the platform holds the secret server-side).
5. **Scopes: check `api`.**

    Why `api` and not a read-only scope: merge-request analysis posts and updates MR notes with the delegated token, and GitLab's OAuth scope catalog has no write scope narrower than `api` (`write_repository` covers git-protocol pushes, not the REST notes API). The token's effective reach is still capped by each connecting user's own GitLab permissions. If you enabled the integration before MR analysis existed, see [Upgrading the scope](#upgrading-the-scope-from-a-read-only-install) below.

6. Click **Save application**, then copy the **Application ID** and **Secret**.

## Step 2: Handoff to the platform operator

| What | Goes into |
| --- | --- |
| Application ID | `gitlab_client_id` in tfvars |
| Secret | SSM SecureString (see below) |

The operator populates tfvars:

```hcl
enable_gitlab_integration = true
gitlab_client_id          = "<application-id>"
# Optional overrides:
# gitlab_base_url                = "https://gitlab.example.com"  # self-managed
# gitlab_client_secret_ssm_param = "/custom/path"                # operator-managed secret path
```

By default terraform creates a **placeholder** SSM parameter at `/{env}/outcome-ops-ai-assist/gitlab/client-secret` and never overwrites its value; the operator sets the real secret after the first apply:

```bash
aws ssm put-parameter \
  --name "/prd/outcome-ops-ai-assist/gitlab/client-secret" \
  --value "<the application secret>" \
  --type SecureString \
  --overwrite \
  --region us-west-2
```

Then `terraform apply -var-file=prd.tfvars` --- see the [Deploy guide](../getting-started/deploy.md).

## Step 3: Verify from the UI

1. Open the OutcomeOps UI as a **workspace_admin** on the target workspace.
2. Navigate to **Workspace Settings → Integrations**.
3. Click **Connect GitLab** and authorize on GitLab's consent screen.
4. Back on Integrations, click **Add Repositories**. The picker walks: groups → projects (nested subgroups included; the group list shows groups where you have at least Reporter access).
5. Add a project; sync status runs **pending → queued → in_progress → success**. Use **Sync Now** if it sits pending.

## Merge-request analysis

Once a project is enrolled, MR checks can be triggered from CI or manually. Each check posts a collapsed `<details>` note on the MR; re-running on a new push **updates the same notes in place** rather than stacking duplicates.

Add to your `.gitlab-ci.yml` (requires AWS credentials with `lambda:InvokeFunction` on the `*-analyze-pr` function):

```yaml
outcomeops-mr-check:
  stage: test
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  script:
    - |
      PAYLOAD=$(jq -n \
        --argjson pr_number "$CI_MERGE_REQUEST_IID" \
        --arg repo "$CI_PROJECT_PATH" \
        --arg conn "$OUTCOMEOPS_CONNECTION_ID" \
        '{pr_number: $pr_number, repository: $repo,
          provider: "gitlab", connection_id: $conn}')
      aws lambda invoke \
        --function-name prd-outcome-ops-ai-assist-analyze-pr \
        --payload "$PAYLOAD" \
        --cli-binary-format raw-in-base64-out \
        /tmp/response.json
      cat /tmp/response.json
```

`OUTCOMEOPS_CONNECTION_ID` is the connection ID shown in Workspace Settings → Integrations after connecting. Opaque; not sensitive.

## Upgrading the scope from a read-only install

Deployments connected before MR analysis shipped used `read_api read_repository` scopes. Those tokens can ingest but cannot post MR notes (the API returns `403 Forbidden`). To upgrade:

1. Edit the OAuth application in GitLab and check the **`api`** scope.
2. Deploy the platform version that requests `api` (the authorize URL scope is baked into the code, not configurable).
3. Each connected user clicks **Reconnect GitLab** in Workspace Settings once. Tokens minted under the old scopes stay read-only until re-authorized.

## Common problems

| Error | Cause | Fix |
| --- | --- | --- |
| `redirect_uri mismatch` on callback | Redirect URI on the application doesn't match exactly. | Match scheme, host, and path character-for-character; no trailing slash. |
| `403 Forbidden` posting MR notes | Token minted under the old read-only scopes, or the connecting user lacks permission to comment on that project. | Upgrade the scope (above) and reconnect; confirm the user can comment on the MR in the GitLab UI. |
| Group missing from picker | The connecting user has less than Reporter access to the group. | Reporter or higher is the enrollment floor (read access to code). |
| `Connection lost: reconnect_required` | The stored refresh token was invalidated (e.g., the user revoked the authorization in GitLab). | Any workspace admin clicks **Reconnect GitLab**. |
| Sync stuck after reconnect | GitLab rotates refresh tokens on every refresh; the platform serializes refreshes automatically. If a connection was poisoned by a pre-v1.1 deploy, disconnect and reconnect once. | Disconnect → Connect GitLab. |

## Disconnecting

Workspace Settings → Integrations → **Disconnect GitLab**. The platform removes every project added through the connection, deletes the associated knowledge-base documents, code maps, and symbol-graph entries (irreversible), deletes the stored tokens, and removes the connection record.

To rotate the connecting user, disconnect first, then another workspace admin connects under their own identity and re-adds the projects.
