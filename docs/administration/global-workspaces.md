---
title: Global Workspaces
description: Make one team's content queryable across the whole org, without maintaining membership lists.
---

# Global Workspaces

A **global workspace** is a public workspace that appears in every authenticated user's workspace picker by default. One team owns the content; everyone in the deploy can query it.

Use global workspaces for content that isn't team-specific but that some team has to own and maintain --- security standards, architecture decisions, onboarding guides, compliance policies. Instead of manually adding every engineer to a "Security" workspace and re-doing that every time someone joins or leaves, you flip the workspace to global and the org membership problem goes away.

## When to use a global workspace

**Good fits:**

- **Security standards** owned by SecOps but queryable by every engineer.
- **Architecture decision records** owned by the platform team but queryable across product engineering.
- **Compliance policies** owned by GRC but queryable when any engineer wonders "can I do X?"
- **Onboarding content** --- how to file expenses, request access, deploy code, get to the incident channel.
- **OutcomeOps Help** --- an internal instance of the OutcomeOps documentation itself, so users can chat with the docs.

**Bad fits:**

- Team-specific runbooks that only that team's on-call rotation needs. Keep those in the team's own private workspace.
- Draft content, WIP standards, anything half-cooked. Global means everyone sees it; use a private workspace to iterate.
- Content that has to have different views for different audiences. If eng needs one framing and legal needs another, ship two workspaces (one global, one private).

## How they behave

- **Discoverability.** Global workspaces show up at the top of every user's workspace picker, tagged **(Global)**, above the user's own team workspaces.
- **Membership isn't tracked.** Users don't join or leave --- they just select the workspace in their picker when they want its content in their chat.
- **Content ownership stays with one team.** The workspace has a normal admin + members list; only they can add integrations, repos, spaces, or edit the system prompt. Everyone else is a read-only consumer.
- **Chat is per-user.** Two users querying the same global workspace have independent conversations. No shared threads.
- **Global workspaces respect PII redaction settings.** If org-level redaction is enabled, redaction runs at ingest time --- so what's in the workspace is already redacted before any user queries it.

## Creating a global workspace

Only **org admins** can flip a workspace to global.

1. Create the workspace as normal (New Workspace → name → private/public → create).
2. Connect the integrations the workspace should ingest --- typically one Confluence space, one repo of policy markdown, or a set of Jira issues.
3. Add the team that owns the content as workspace admins.
4. Sign in as an **org admin**.
5. Navigate to **Admin → Workspaces** → find the workspace → **Make global**.
6. Confirm. Every user's picker now shows this workspace with a **(Global)** tag.

Reversing the flag (Admin → Workspaces → **Remove global**) hides it again --- the workspace stays intact with its members and content, just no longer visible to non-members.

## Naming conventions

Because global workspaces appear in every user's picker, naming matters:

- **Lead with the category** so pickers sort alphabetically into groups: `Global: Security Standards`, `Global: Architecture Decisions`, `Global: Compliance Policies`.
- **Keep names short** --- 30 characters or less. The picker sidebar truncates long names.
- **Avoid team-of-origin in the name** unless the team ownership is the point --- `Global: SecOps Runbook` is fine if the team boundary is meaningful; `Global: Security Standards` is better if the content is the point.

## Example patterns

### Security Standards workspace

- **Content:** a `github.com/yourcompany/security-standards` repo of Markdown standards, connected via the GitHub integration.
- **Owned by:** SecOps team as workspace admins.
- **Global.** Every engineer can chat "what's our secret-management standard?" and get an answer grounded in the actual doc.
- **System prompt:** "Always cite the specific standard you're pulling from. If the question is about an exception, note that exceptions must be approved by SecOps."

### Architecture Decision workspace

- **Content:** the ADR folder from your platform monorepo (or a dedicated `adrs` repo).
- **Owned by:** platform architecture team.
- **Global.** New engineers can chat "why do we use Terraform workspaces instead of separate accounts?" and get the ADR that made the call.
- **System prompt:** "Answer based on the ADR content. If an ADR supersedes another, cite both and note which is current."

### OutcomeOps Help workspace

Once your deploy is running, seed a global workspace from the public [docs.outcomeops.ai](https://docs.outcomeops.ai) content. Users can chat with the docs directly instead of navigating them.

## Interaction with other features

- **[MCP servers](../features/mcp-servers.md)** attached to a global workspace are usable by every user in chat. Great for a company-wide read-only tool (like an internal API browser); risky for a tool that mutates external state (approve carefully).
- **[Code generation](../features/code-generation.md)** doesn't fire from global workspaces --- only from workspaces where a team has explicit ticket → repo mapping. Global workspaces are read-only in practice.
- **[PII redaction](pii-redaction.md)** runs at ingest, so global workspaces inherit whatever redaction posture the org has set. Global doesn't override org-level rules.

## Common gotchas

- **Picker fatigue.** If you have 15 global workspaces, users stop reading them and just click their team's workspace. Aim for 2-5 global workspaces max, at least in year one.
- **Stale content.** Global workspaces attract "set it and forget it" thinking. Assign an owner + a review cadence (quarterly is reasonable) so the content stays fresh.
- **Cross-team ownership drift.** If two teams try to co-own a global workspace, the system prompt ends up muddled. Pick one owner; other teams contribute via PRs to the underlying repo/space.

## Related

- [Workspaces](../features/workspaces.md) --- workspace basics, membership, roles.
- [Integration Gating](integration-gating.md) --- what integrations are provisioned deploy-wide.
- [PII Redaction](pii-redaction.md) --- how content is redacted at ingest, including for global workspaces.
