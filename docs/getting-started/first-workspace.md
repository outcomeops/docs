---
title: First Workspace
description: Create your first workspace, connect an integration, and ask your first question.
---

# First Workspace

After [deploying OutcomeOps AI Assist](deploy.md) you have a running platform but no content in it. This page walks through creating your first workspace, connecting the GitHub integration to it, and asking your first question.

Time budget: **10-15 minutes**, most of which is waiting for the initial sync to complete.

## What is a workspace?

A workspace is the primary scope for content in OutcomeOps AI Assist. Think of it as **a team's context bubble**: the code they own, the tickets they work on, the documents they write, and the databases they query. When a user chats with the platform, they select one or more workspaces, and the retrieval layer only pulls from content in those workspaces.

Workspaces are how you keep the security team's incident runbooks separate from the platform team's terraform modules. Both content sets exist in the same deploy, but each team's chat only sees the workspaces they're a member of.

Two special cases are worth knowing about up front:

- **Org admins** can see and manage every workspace in the deploy, regardless of membership.
- **Global workspaces** are readable by every authenticated user without membership. Use these for content one team owns but everyone should be able to query --- security standards, architecture decisions, an OutcomeOps Help workspace. See [Global Workspaces](../administration/global-workspaces.md).

## Sign in as the initial org admin

The tfvar `initial_org_admin_email` you set during deploy grants the org-admin role to that email address the first time they sign in via Azure AD. Open the UI in a browser, sign in with that email, and you'll land on the Workspaces page.

## Create a workspace

1. Click **New Workspace** in the top-right.
2. Enter a name (e.g., `platform-team`) and an optional description.
3. Select **Private** for a team-scoped workspace. (You can flip it to Global later; see [Global Workspaces](../administration/global-workspaces.md).)
4. Click **Create**.

You now have an empty workspace. The next step is to give it content by connecting an integration.

## Connect the GitHub integration

The [GitHub App integration](../integrations/github.md) is the easiest first integration because most organizations already have a GitHub org.

1. Open your new workspace and click the **Integrations** tab.
2. Click **+ Connect** next to GitHub.
3. You'll be redirected to GitHub to install the OutcomeOps GitHub App on your org. Grant access to the specific repositories you want the workspace to ingest.
4. After the install completes, you're redirected back to the workspace. Click **Add repos** and pick the repos to include in this workspace.

Ingest starts in the background. Each repo goes through:

- **Cloning + chunking** --- files are split into semantic chunks.
- **Code-map generation** --- a Claude Haiku pass extracts the callers-callees-implementers-definitions symbol graph.
- **Embedding + indexing** --- chunks are embedded (Titan v2 by default) and written to S3 Vectors.

A small repo (a few thousand files) is typically ready to query in 2-5 minutes. Larger monorepos take longer; you can watch progress in the Integrations panel.

## Ask your first question

Open the **Chat** tab. Verify your new workspace is selected in the workspace picker (top of the composer). Try one of these queries:

- "What's the structure of this repo?"
- "Which handler processes the OAuth callback for &lt;integration&gt;?"
- "Show me how &lt;function name&gt; is called across the codebase."

The response streams token-by-token, with source citations linked in the sidebar. Click a source to see the exact file + line range the platform pulled the answer from --- every claim is grounded in code you can inspect.

## What's next

- **Invite teammates.** Workspace admins can add members from the **Members** tab. Members can chat and add repos; workspace admins can also configure integrations.
- **Enable more integrations.** Jira for issue context, Confluence for design docs, database for schema-aware queries. See the [Integrations](../integrations/index.md) index.
- **Consider a global workspace.** For content everyone in your org should be able to query without joining a team --- security standards, architecture decisions, onboarding docs. See [Global Workspaces](../administration/global-workspaces.md).
- **Wire up code generation.** The [code generation](../features/code-generation.md) feature turns Jira issues (or GitHub Issues) into PRs. It's the most impactful workflow and worth spending an hour setting up once you've validated chat.
