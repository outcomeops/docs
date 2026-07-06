---
title: Jira
description: Connect Jira for issue-context ingest plus code generation triggered from Jira Automation.
---

# Jira

The Jira integration does two things, both enabled by the same tfvar:

1. **Ingest Jira issues as chat context.** The platform reads issues from the Jira projects you connect, embeds them, and makes them retrievable in chat --- so users can ask "what's the acceptance criteria on PROJ-1234?" and get the answer grounded in the ticket.
2. **Automation-triggered code generation.** Jira Automation rules invoke an AWS SSM document via cross-account IAM, which fires the platform's code-generation Lambda. Add a label like `approved-for-generation` to a Jira issue and the platform opens a PR in the mapped GitHub repo with the implementation.

Setting `enable_jira_integration = true` in your tfvars provisions BOTH surfaces --- workspace-level Jira ingest and the SSM Automation cross-account role. If you only want one and not the other, note that in v1 they ship together.

Time budget: **~45 minutes** for someone with Jira admin + AWS admin access.

## Setup overview

Two paths run in parallel:

- **Workspace-level Jira ingest** --- OAuth handshake in the UI, pick projects to sync. Follows the same pattern as the other OAuth integrations.
- **Code-generation automation** --- Jira Automation → AWS Systems Manager Document → code-generation Lambda → GitHub PR. Requires configuring an AWS connector in Jira Automation and a cross-account IAM role.

The workspace-level ingest is optional; the code-generation flow is what most teams enable first because it's the highest-ROI use case. This page covers the code-generation setup end to end. The workspace-level connect flow is a UI walkthrough similar to [Confluence](confluence.md).

## Code-generation flow

```text
Jira Issue (labeled)
      ↓
Jira Automation Rule (Work Item Transitioned)
      ↓
AWS Systems Manager Document (cross-account, via IAM role)
      ↓
generate-code Lambda
      ↓
GitHub PR (with plan and/or code)
```

Two labels drive the behavior:

| Label | Behavior |
| --- | --- |
| `approved-for-plan` | Creates a PR with the implementation plan only (no code). Use this to review the approach before generating code. |
| `approved-for-generation` | Creates a PR with full code, tests, and documentation. |

You can label an issue `approved-for-plan` first, review the plan PR the platform opens, then switch to `approved-for-generation` if you like the approach.

## Prerequisites

- **OutcomeOps AI Assist deployed** with `enable_jira_integration = true` (see [Integration Gating](../administration/integration-gating.md)).
- **Jira admin access** on the project(s) you want to enable.
- **AWS CLI + permissions** to view Terraform outputs and inspect IAM roles.

## Step 1: Create the labels in Jira

Jira Automation's label condition dropdown only shows labels that already exist on at least one issue. Create a "Label Init" story so the labels become selectable:

1. Create a new Jira issue titled `Label Init - Do Not Delete`.
2. Add both labels:
    - `approved-for-plan`
    - `approved-for-generation`
3. Move the issue to the backlog. Never assign or delete it.

## Step 2: Create a Jira component per repository

Each repository you want to enable for code generation needs a Jira component whose **name matches the GitHub repository path exactly** (case-sensitive).

1. Go to **Project Settings → Components**.
2. Create a component:
    - **Name:** `owner/repo` (e.g., `outcomeops/example-app`)
    - Description: optional

When creating issues, assign them to the appropriate component to tell the platform which repo should receive the generated code.

## Step 3: Deploy Terraform (Phase 1)

Set the flag in your tfvars and apply:

```hcl
enable_jira_integration = true
```

```bash
cd terraform
terraform workspace select prd
terraform apply -var-file=prd.tfvars
```

Copy the IAM role ARN from the output:

```text
jira_automation_role_arn = "arn:aws:iam::123456789012:role/atlassian-automation-prd-outcome-ops-ai-assist"
```

## Step 4: Configure the AWS connector in Jira Automation

1. In Jira, go to **Jira Settings → Automation → Library → Connect to AWS**.
2. Click **Add AWS account**.
3. Enter the Role ARN from Step 3.
4. Jira displays an **External ID**. Copy it. It looks something like:

```text
automation-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

## Step 5: Deploy Terraform (Phase 2)

Update your tfvars with the External ID and re-apply:

```hcl
enable_jira_integration = true
jira_external_id        = "automation-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

```bash
terraform apply -var-file=prd.tfvars
```

This updates the IAM role's trust policy so Jira Automation can actually assume it. Without the correct External ID, the assume-role call fails with an `AccessDenied` error.

## Step 6: Create the Jira Automation rule

1. Go to **Project Settings → Automation → Create rule**.

### Trigger

**When:** Work Item Transitioned

- From status: `To Do`
- To status: `In Progress`

### Condition

**If:** Labels contain any of

- `approved-for-plan`
- `approved-for-generation`

### Action

**Then:** Run AWS Systems Manager automation

- **AWS Account:** your connected account
- **Region:** your deployment region (e.g., `us-west-2`)
- **Document:** `prd-outcome-ops-ai-assist-jira-automation` (substitute your environment name for `prd`)

**Parameters:**

| Parameter | Smart Value |
| --- | --- |
| `issueKey` | `{{issue.key}}` |
| `issueSummary` | `{{issue.summary}}` |
| `issueDescription` | `{{issue.description}}` |
| `componentKey` | `{{issue.components.name}}` |
| `label` | `{{trigger.label.name}}` |

The `label` parameter captures which label triggered the automation, so the Lambda knows whether to generate a plan or full code.

### Save and enable

Name the rule (e.g., "OutcomeOps Code Generation"), save, and enable it.

## Step 7: Test the integration

1. Create a new Jira issue with:
    - A descriptive title (becomes the GitHub PR title)
    - A detailed description (user story, acceptance criteria)
    - Component set to your target repo (e.g., `outcomeops/example-app`)
2. Add the `approved-for-plan` label and transition the issue to `In Progress`.
3. Check Jira Automation's audit log for success or failure.
4. Verify a PR was created in the target GitHub repo with the implementation plan.
5. To generate full code, add `approved-for-generation` and transition back to trigger the rule again.

## Common problems

| Error | Cause | Fix |
| --- | --- | --- |
| `Access Denied` in Jira Automation audit log | External ID in tfvars doesn't match Jira's External ID exactly. | Copy the External ID from Jira again (Step 4), update tfvars, re-apply Terraform. |
| Automation rule doesn't trigger | Wrong label, wrong status transition, or missing component on the issue. | Verify: label is one of the two approved labels, the issue transitioned from `To Do` to `In Progress`, and the issue has a component set matching a GitHub repo path. |
| `SSM Document not found` | Document name in the rule doesn't match Terraform's naming convention. | Document name is `{env}-{app_name}-jira-automation` --- verify the environment prefix matches your deployed tfvars. |
| `Parameter not found: GetParameter` | Your issue description contains `{{variable}}` patterns that Jira interprets as smart values. | Wrap the description in a `{code}` block or escape the braces. |
| No PR appears in GitHub | Component name doesn't match a GitHub repo the platform can access. | Verify the component name is `owner/repo` exactly, and that the GitHub integration is enabled with the target repo added to a workspace. |

## What the flag gates

Setting `enable_jira_integration = false` (or leaving the default) provisions none of this: no SSM Automation document, no cross-account IAM role, no workspace-level Jira Lambdas, no SQS queues. SecOps can look at your tfvars and know exactly which surface exists. See [Integration Gating](../administration/integration-gating.md) for the full flag list.
