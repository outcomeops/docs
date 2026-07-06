---
title: FAQ
description: Technical answers to the questions people actually ask before and during adoption.
---

# FAQ

## Deployment

### Where does my data go?

Nowhere. OutcomeOps AI Assist runs entirely in your AWS account. Content ingested from your integrations lands in your S3 buckets, your DynamoDB tables, your S3 Vectors index. LLM calls go to your Bedrock endpoints. Chat conversations save to your DynamoDB. Nothing leaves your account.

### Does OutcomeOps have any access to my data?

No. The deploy runs under your IAM. There is no cross-account role that grants OutcomeOps access to anything in your account. OutcomeOps ships you Terraform modules and container images; you run them.

### What does the license server see?

Two things: (1) the license JWT you provide, which the license server validates; (2) a repo-count snapshot the platform sends periodically (how many repos are connected across all workspaces). No source code, no chat content, no user emails. The snapshot exists so OutcomeOps can enforce license terms; contact us if you need to see the exact payload.

### Do I need to open outbound firewall rules?

For non-air-gapped tiers: yes, one --- to `license.outcomeops.ai` for license validation and (optionally) help-content updates. For air-gapped enterprise tiers: no --- licensing is baked into the deploy artifact and help content ships as an immutable snapshot per release.

### How do I upgrade?

Bump the Terraform module version in your tfvars, run `make pre-deploy` to rebuild layers + images, then `terraform apply`. Release notes ship at `outcomeops.ai/releases` (or via your account team for regulated deploys).

### Which AWS regions are supported?

Any region where Bedrock supports the models you configure. Claude Sonnet + Haiku (defaults) are broadly available. OpenAI on Bedrock (GPT-5.5) is us-east-2 only at time of writing. Verify the model IDs in your tfvars against your target region before applying.

### What's the deploy footprint?

Ballpark for a full-enable deploy (every integration flag `true`):

- ~40 Lambda functions (shared core + one integration + one sync + one scheduler per enabled integration)
- ~15 DynamoDB tables (workspaces, audit, analytics, symbol graph, code maps, license snapshot, etc.)
- ~10 SQS queues + 10 DLQs (one pair per integration)
- ~5 KMS keys (encryption at rest across the stack)
- 2 Fargate services (chat UI + MCP catalog)
- 1 Application Load Balancer (UI)
- ~30 SSM SecureString parameters (integration credentials + license)

Disable integrations you don't use and the footprint drops proportionally. See [Integration Gating](administration/integration-gating.md).

## Models

### Which Bedrock models are supported?

The current defaults:

- **Advanced:** Claude Sonnet 4.6 (synthesis, PR-check analysis, code generation)
- **Basic:** Claude Haiku 4.5 (query rewriting, summarization, code-map extraction)
- **Embedding:** Titan Embed Text v2 (vectors)
- **Rerank:** Cohere Rerank v3.5 (dual-channel retrieval reranking; optional)

All four are swappable via tfvars. Regulated customers who need a specific model (their compliance-approved variant) override the model IDs. OpenAI GPT-5.5 on Bedrock is supported via a shared model client abstraction (currently us-east-2 only).

### Can I use my own LLM (not Bedrock)?

Not directly in v1. The shared model client is Bedrock-first with a Mantle backend for OpenAI-on-Bedrock. Direct-to-Anthropic-API or OpenAI API access isn't the default posture --- regulated deploys prefer Bedrock's within-AWS traffic + compliance boundary.

If you have a use case for a non-Bedrock model (an on-prem Llama, a private inference endpoint), talk to us --- extending the shared model client is a plausible engineering path.

### Are model costs transparent?

Yes. Every LLM call the platform makes logs its input and output token counts to DynamoDB, tagged by workspace and user. The audit dashboard rolls up cost per workspace, per user, per model. Set daily / monthly cost alerts in your tfvars (`cost_alert_workspace_monthly_usd`, etc.).

## Governance

### Can SecOps review what my deploy provisions?

Yes. Your tfvars are the source of truth --- each `enable_*_integration` flag maps to a documented set of resources (see [Integration Gating](administration/integration-gating.md)). The unauthenticated `/api/config/integrations` endpoint returns the live manifest so SecOps can verify the deploy state matches the tfvars.

### Does OutcomeOps AI Assist meet SOC 2 / HIPAA / FedRAMP?

Compliance is a property of the **deploy**, not the software. OutcomeOps AI Assist deploys into your AWS account, using your KMS keys, under your IAM. The platform's design supports compliance postures --- audit trails, KMS-encrypted everything, no cross-account access, in-account processing --- but certification is on your side. Ask your account team about the deploy checklist for your compliance regime.

### How do I audit what users did?

Every mutation writes an audit row to DynamoDB with the user's email, source IP, user agent, action, and workspace. Read-only chat queries are logged separately in the chat audit table (with input, output, retrieval sources, tool calls, and token counts). Both are queryable from the Admin → Audit dashboard, or via CloudWatch Logs Insights if you want raw access.

### Can I stream audit events to my SIEM?

Yes, via the audit stream export feature. Set `audit_stream_export_enabled = true` in your tfvars --- Terraform provisions a Kinesis Data Stream and a publisher Lambda that re-emits every audit row as an OCSF v1.3.0 envelope. Attach your SIEM's consumer (Splunk, Datadog, Sumo, Security Lake, Firehose-to-S3) to the stream.

## Content + Integrations

### Can I add an integration OutcomeOps doesn't natively support?

Yes, via [MCP servers](features/mcp-servers.md). If the external system speaks the Model Context Protocol, you point the platform at its endpoint and users can invoke its tools from chat. Common pattern: wrap a REST API in a thin MCP shim you self-host, and add it as a workspace-scoped MCP.

### What does the platform do with generated code?

It opens a pull request. The PR is authored by the OutcomeOps GitHub App, attributed to AI generation, and shows up in your normal PR review flow. You approve, request changes, or close --- same as any other PR. The platform doesn't merge on its own.

### How fast does content become queryable after ingest?

Small repos: 2-5 minutes from add to chat-queryable. Large monorepos: 20-60 minutes. Confluence spaces, Jira projects, database schemas: similar range depending on volume. The Integrations panel shows sync status per item.

### Does the platform re-sync forever, or does it stop?

Continuous. Every integration has an hourly scheduler that fires the sync worker; each worker looks at what's changed since last time and updates only the delta. If you add a new file to an ingested repo, it becomes queryable within the next hour. If you don't want a specific repo synced, remove it from the workspace.

## Support + Contribution

### Where do I file bugs or feature requests?

For product issues (deployment problems, bugs, feature requests): open a ticket at the [OutcomeOps Support Portal](https://outcomeops.atlassian.net/servicedesk/customer/portal/1). Your account team triages from there.

For documentation issues (typos, wrong details, missing content): [open an issue in the docs repo](https://github.com/outcomeops/docs/issues/new). Include the page URL and what's confusing.

### Can I contribute to the docs?

Issues yes, pull requests no. See the [docs repo README](https://github.com/outcomeops/docs) for the contribution policy. If you have suggested wording for a fix, include it in your issue --- we'll lift it into the source when it fits.

### Is there a Slack / Discord community?

Not currently. OutcomeOps AI Assist is early-stage; support flows through your account team. As adoption grows we may open a broader community channel.

### What SLA does OutcomeOps offer?

OutcomeOps ships software; the SLA is what your AWS account provides. The platform is multi-AZ by default and supports multi-region HA (active/passive with per-region KMS keys). Configure your AWS account for the availability posture you need; the platform inherits it.
