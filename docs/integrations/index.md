---
title: Integrations
description: Every workspace integration OutcomeOps AI Assist supports, with a setup guide for each.
---

# Integrations

OutcomeOps AI Assist connects to the systems your teams already use. Each integration is provisioned by a single Terraform flag; enable only what you need.

| Integration | What it ingests | Setup guide |
| --- | --- | --- |
| **GitHub** | Repository content, issues, PRs | [GitHub setup](github.md) |
| **Azure DevOps** | Repo content, work items (Boards), PR checks | [Azure DevOps setup](azure-devops.md) |
| **Jira** | Issues (chat context) + automation-triggered code generation | [Jira setup](jira.md) |
| **Confluence** | Space pages | [Confluence setup](confluence.md) |
| **Microsoft 365** | Outlook mail, Teams messages, SharePoint sites, OneDrive folders, OneNote notebooks | [Microsoft 365 setup](microsoft-365.md) |
| **Database** | Schemas + tables (MSSQL, MySQL, PostgreSQL) | [Database setup](database.md) |

## How the gating works

Every integration is **off by default** in a fresh deploy. Each has its own tfvar --- `enable_github_integration`, `enable_azure_devops_integration`, etc. --- and setting the flag to `true` provisions the full backend surface (Lambdas, SQS, EventBridge, SSM parameters). Setting it to `false` (or leaving the default) provisions nothing.

This is the SecOps story: **nothing you don't enable exists in your AWS account.** A reviewer can look at your tfvars and know exactly what surface has been provisioned. See [Integration Gating](../administration/integration-gating.md) for the full flag table.

## Pattern for every integration setup

The setup pages follow a consistent order:

1. **What this connects** --- what content the integration ingests and what the platform does with it.
2. **Prerequisites in the third-party system** --- OAuth app, GitHub App, or Entra ID registration you configure before enabling the flag.
3. **Handoff to the platform operator** --- the tfvars they populate and the SSM SecureString they create.
4. **Enable the flag and deploy** --- Terraform apply.
5. **Connect from the UI** --- workspace admin walks the OAuth handshake and picks repos / projects / spaces / folders to include.
6. **Common problems** --- the errors people actually hit and how to resolve them.

Start with the integration you already have credentials for. Most customers begin with GitHub or Azure DevOps.
