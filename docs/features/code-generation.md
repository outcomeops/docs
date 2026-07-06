---
title: Code Generation
description: Turn Jira issues or GitHub Issues into pull requests through label-driven automation.
---

# Code Generation

The code-generation feature turns a ticket into a pull request. A team member writes a Jira issue or GitHub Issue describing what they want, adds a label, and the platform opens a PR in the target repository with either an implementation plan or full code + tests. Reviewers approve or reject the PR the same way they'd review a human contribution.

This is the highest-ROI workflow OutcomeOps AI Assist enables, and it's the one most customers set up first.

## Two modes

| Label | What the platform generates |
| --- | --- |
| `approved-for-plan` | A PR containing an implementation plan --- diagrams, file-by-file changes proposed, test strategy. No actual code changes yet. |
| `approved-for-generation` | A PR containing the full implementation: source files, tests, updates to related docs. |

The recommended flow is **plan first**:

1. Add `approved-for-plan` → review the plan the platform proposes → is the approach right?
2. If yes, add `approved-for-generation` → the platform re-runs and opens a second PR with the actual code implementing the plan.
3. If the plan is wrong, comment on the plan PR, close it, iterate the ticket description, and try again.

Skipping directly to `approved-for-generation` works too, but the plan-first loop catches "wrong approach" cheap before you're reviewing generated code line by line.

## Two triggers

Code generation can fire from either Jira Automation or GitHub Issues. Both flows converge on the same code-generation Lambda:

- **[Jira → SSM → Lambda](../integrations/jira.md#code-generation-flow).** Requires the Jira integration + AWS connector setup.
- **[GitHub Issues → webhook → Lambda](../integrations/github.md).** Requires the `enable_github_issue_integration` flag (this is a separate flag from `enable_github_integration`; it gates the code-gen API Gateway that receives GitHub webhooks).

Pick whichever ticketing system your teams already use. Some customers wire both up so different teams can use their preferred tool.

## Ticket-to-repo mapping

Neither Jira nor GitHub Issues knows which repository a ticket should target. You tell the platform via **component** (Jira) or **repo label** (GitHub Issues):

- **Jira:** create a component in your Jira project whose name matches the GitHub repository path exactly (case-sensitive), e.g., `outcomeops/example-app`. Set that component on the issue.
- **GitHub Issues:** the issue is already in a repo; the platform opens the PR in that same repo.

Multi-repo workflows: for a Jira issue that spans two repos, create the issue with the primary repo as its component. If cross-repo changes are needed, the platform opens the primary PR and comments on the ticket noting the required cross-repo changes for a human to sequence.

## What the platform sees when generating

The code-generation Lambda has scoped access to:

- **The target repo's default branch code + tests + docs** (from ingested content in workspaces where the repo is added).
- **The symbol graph** for the target repo (function definitions, callers, callees, importers). This is what makes generated code aware of what already exists.
- **The workspace's other repos** (if the same workspace ingests multiple repos, the platform can pattern-match on how you build one microservice to guide the shape of another).
- **The issue's description + comments** as the intent input.
- **Any ADRs or docs** ingested in the same workspace, so generated code cites relevant standards.

The Lambda does NOT see:

- Repos in other workspaces the ticket-triggering user doesn't have access to.
- Code from repos outside the ingested set.
- Anything from integrations the workspace doesn't have connected.

Workspace scope is what keeps generated code from accidentally leaking between teams.

## What the platform generates

For plan-mode:

- **`docs/adr/ADR-NNN-<slug>.md`** proposing the architectural decision (if the plan implies a nontrivial decision).
- **`plan/` files** with the file-by-file change proposal, test strategy, and rollout considerations. These are gitignored in the platform's own repo but shipped in generated PRs so reviewers can see them alongside the ADR.

For generation-mode:

- **Source files** implementing the plan.
- **Test files** covering the new code paths.
- **Doc updates** if the change affects `README.md` or existing docs in the repo.
- **A PR description** summarizing what changed and why, with links back to the source ticket.

Every PR is opened by the OutcomeOps GitHub App and attributed to the AI-generated author. Reviewers see standard GitHub review UI; you can request changes, approve, or close.

## Quality guardrails

Before a generated PR opens, the platform runs analysis passes:

- **ADR compliance** --- does the generated code follow the workspace's ADRs?
- **Test coverage** --- do the generated tests cover the new lines?
- **Breaking changes** --- does the diff introduce API breakage?
- **Architectural duplication** --- did the platform generate something that already exists elsewhere in the codebase?
- **License compliance** --- do imported dependencies satisfy the workspace's license policy?

Each analysis result posts as a PR check. Failed checks don't block the PR from opening --- reviewers see the flags and decide whether to fix or accept.

## Enabling the feature

Prerequisites (in tfvars):

- `enable_github_integration = true` --- so the platform can open PRs.
- `enable_jira_integration = true` --- if you want Jira as the trigger.
- `enable_github_issue_integration = true` --- if you want GitHub Issues as the trigger. **Note:** this flag provisions a public API Gateway that receives GitHub webhooks. Only enable if you're comfortable with that public surface.

Then follow the setup guides:

- [Jira setup](../integrations/jira.md) for the Jira Automation → SSM path.
- [GitHub setup](../integrations/github.md) for the GitHub Issues webhook path.

## Rolling out to a team

Start narrow: **one team, one repo, plan-only mode**. Have the team write a real ticket, add `approved-for-plan`, and review the resulting PR together. Discuss what the platform got right and wrong. Iterate on the workspace's system prompt if the plans consistently miss the same context.

Once plan quality is reliable, flip to generation mode for tickets the team is confident about. Keep the plan-first loop as an option for anything nontrivial.

Common anti-pattern: enabling generation mode across every team on day one. Generated code that's 70% right but 30% subtly wrong wastes more reviewer time than it saves. Plan-first mode catches this at a much cheaper stage.
