---
title: Workspaces
description: How workspaces scope content and membership in OutcomeOps AI Assist.
---

# Workspaces

A workspace is the primary scope for content and membership. It's a **team's context bubble** --- the code they own, the tickets they work on, the docs they write, the databases they query. When a user chats, they pick one or more workspaces, and retrieval only pulls from content in those workspaces.

## Membership + roles

Three role levels within a workspace:

- **Workspace admin** --- can add / remove members, configure integrations, add repos + projects, set the workspace's system prompt, and delete the workspace. There must be at least one workspace admin per workspace.
- **Member** --- can chat with the workspace selected and read its content. Cannot change membership or integrations.
- **Org admin** (org-level, not per-workspace) --- can see and manage every workspace regardless of membership. Set via the `initial_org_admin_email` tfvar at deploy time; subsequent org admins are promoted from the org admin console.

## Public workspaces

By default a workspace is **private** --- only its members can select it in chat. Flip a workspace to **public** and any authenticated user in the deploy can select it and query its content, without being a member.

The typical use case: one team owns a knowledge set that other teams want to reference. Public workspaces let you share without maintaining membership lists.

Public workspaces are still bounded --- users must be signed in to the deploy. They don't become internet-accessible.

## Global workspaces

A **global workspace** is a public workspace that appears in every user's workspace picker by default. Use this for content everybody should be able to query without joining a team --- security standards, architecture decisions, an OutcomeOps Help workspace.

Global workspaces are configured by org admins. See [Global Workspaces](../administration/global-workspaces.md) for the pattern and use cases.

## Adding integrations to a workspace vs. adding a workspace to an integration

This is the direction that surprises new users.

Integrations are **deploy-wide** --- when the operator sets `enable_github_integration = true` and deploys, the GitHub App integration exists across the whole deploy. But no workspace consumes any GitHub content until someone **adds repos to a workspace**.

Every integration has a picker that says "which content should this workspace ingest?" That's how you scope integrations to workspaces without provisioning a separate integration per workspace.

So the flow is:

1. Deploy: operator enables the integration flag → provisions the platform-wide integration Lambdas.
2. Workspace admin: opens the workspace's Integrations tab → clicks Connect → completes OAuth → picks specific repos / projects / spaces to include in **this** workspace.
3. Chat: user selects the workspace → retrieval pulls from the added content.

The same repo can be added to multiple workspaces if two teams both care about it. Ingestion happens once at the deploy level; each workspace maintains its own membership list of which repos it wants to include in its retrieval scope.

## System prompt

Each workspace has an optional **system prompt** that gets appended to the LLM's system message for every chat turn in that workspace. Use it to encode team-specific context:

- "This is the platform team's workspace. Assume the reader is familiar with our Terraform module conventions."
- "This is the compliance team's workspace. Always cite the security standard when answering."
- "This is the onboarding workspace. Explain jargon inline and link to detailed pages."

Workspace admins edit the system prompt from Workspace Settings → System Prompt. Changes take effect immediately on the next chat turn.

## Conversation ownership

Conversations belong to the user who created them, scoped to a workspace. Two users chatting with the same workspace selected have independent conversation histories.

Sharing: users can generate a read-only share link to a specific conversation. The recipient can read the conversation but can't send new messages. Share links respect workspace membership --- if the recipient isn't a member (and the workspace isn't public), the share link 404s.

## Deleting a workspace

Only workspace admins can delete a workspace. Deletion is irreversible:

- All conversations in the workspace are deleted.
- All ingested content is removed from the KB (vectors, code maps, ticket rows).
- All integration connections are torn down.
- Members lose access.

The underlying integrations remain enabled for the deploy; only this workspace's use of them ends.

## Best practices

- **One workspace per team** works well for most orgs. Cross-team collaboration happens via public / global workspaces or by inviting members to another team's workspace.
- **Don't create a workspace per repo.** The platform is designed for coarser scoping --- if one team owns 20 microservices, one workspace with all 20 repos added is the right shape.
- **Use global workspaces sparingly.** They appear in every user's picker; too many creates picker fatigue. Reserve them for content genuinely everyone should query.
