---
title: FAQ
description: Technical answers to the questions people actually ask before and during adoption.
---

# FAQ

## Deployment

### Where does my data go?

Nowhere. OutcomeOps AI Assist runs entirely in your AWS account. Content ingested from your integrations lands in your S3 buckets, your DynamoDB tables, your S3 Vectors index. LLM calls go to your Bedrock endpoints. Chat conversations save to your DynamoDB. Nothing leaves your account.

### Does OutcomeOps have any access to my data?

No. The deploy runs under your IAM. There is no cross-account role that grants OutcomeOps access to anything in your account. The [source](https://github.com/bcarpio/outcome-ops-ai-assist) is open under Apache 2.0; you build the images and run the Terraform yourself.

### Do I need to open outbound firewall rules?

Not for OutcomeOps --- the platform never calls home. Outbound traffic goes to AWS service endpoints (Bedrock, S3, DynamoDB, and so on) and to the integrations you enable (GitHub, Atlassian, Microsoft Graph, your databases). Allow those and nothing else.

### How do I upgrade?

Pull the version you want from the [repository](https://github.com/bcarpio/outcome-ops-ai-assist) (`git pull`), run `make pre-deploy` to rebuild layers + images, then `terraform apply`. What changed is in the commit history.

### Which AWS regions are supported?

Any region where Bedrock supports the models you configure. Claude Sonnet + Haiku (defaults) are broadly available. OpenAI on Bedrock (GPT-5.5) is us-east-2 only at time of writing. Verify the model IDs in your tfvars against your target region before applying.

### What's the deploy footprint?

Ballpark for a full-enable deploy (every integration flag `true`):

- ~40 Lambda functions (shared core + one integration + one sync + one scheduler per enabled integration)
- ~15 DynamoDB tables (workspaces, audit, analytics, symbol graph, code maps, etc.)
- ~10 SQS queues + 10 DLQs (one pair per integration)
- ~5 KMS keys (encryption at rest across the stack)
- 2 Fargate services (chat UI + MCP catalog)
- 1 Application Load Balancer (UI)
- ~30 SSM SecureString parameters (integration credentials)

Disable integrations you don't use and the footprint drops proportionally. See [Integration Gating](administration/integration-gating.md).

## Models

### Which Bedrock models are supported?

The current defaults:

- **Advanced:** Claude Sonnet 5.5 (synthesis, PR-check analysis, code generation)
- **Basic:** Claude Haiku 5.5 (query rewriting, summarization, code-map extraction)
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

Compliance is a property of the **deploy**, not the software. OutcomeOps AI Assist deploys into your AWS account, using your KMS keys, under your IAM. The platform's design supports compliance postures --- audit trails, KMS-encrypted everything, no cross-account access, in-account processing --- but certification is on your side, scoped to your deploy.

### How do I audit what users did?

Every mutation writes an audit row to DynamoDB with the user's email, source IP, user agent, action, and workspace. Read-only chat queries are logged separately in the chat audit table (with input, output, retrieval sources, tool calls, and token counts). Both are queryable from the Admin → Audit dashboard, or via CloudWatch Logs Insights if you want raw access.

### Can I stream audit events to my SIEM?

Yes, via the audit stream export feature. Set `audit_stream_export_enabled = true` in your tfvars --- Terraform provisions a Kinesis Data Stream and a publisher Lambda that re-emits every audit row as an OCSF v1.3.0 envelope. Attach your SIEM's consumer (Splunk, Datadog, Sumo, Security Lake, Firehose-to-S3) to the stream.

## Content + Integrations

### Can I add an integration OutcomeOps doesn't natively support?

Yes, via [MCP servers](features/mcp-servers.md). If the external system speaks the Model Context Protocol, you point the platform at its endpoint and users can invoke its tools from chat. Common pattern: wrap a REST API in a thin MCP shim you self-host, and add it as a workspace-scoped MCP.

### Which programming languages are supported?

Code maps and the symbol graph --- the structural "who calls X", "what implements Y", and cross-repo breaking-change detection --- cover **Python, TypeScript/JavaScript, Java, C#/.NET, and Go**. A plain library repo is discovered and indexed automatically, with no `.outcomeops.yaml` required. ABAP/SAP is supported for code-map extraction (read-only), without cross-repo binding. Breaking-change and architectural-duplication analysis reach across repository boundaries for all five of the first group.

Code *generation* (the optional Jira-ticket-to-PR pipeline) ships backends for Python, Java, TypeScript, and React; the other languages are for understanding and review, not generation.

### What does the platform do with generated code?

It opens a pull request. The PR is authored by the OutcomeOps GitHub App, attributed to AI generation, and shows up in your normal PR review flow. You approve, request changes, or close --- same as any other PR. The platform doesn't merge on its own.

### How fast does content become queryable after ingest?

Small repos: 2-5 minutes from add to chat-queryable. Large monorepos: 20-60 minutes. Confluence spaces, Jira projects, database schemas: similar range depending on volume. The Integrations panel shows sync status per item.

### Does the platform re-sync forever, or does it stop?

Continuous. Every integration has an hourly scheduler that fires the sync worker; each worker looks at what's changed since last time and updates only the delta. If you add a new file to an ingested repo, it becomes queryable within the next hour. If you don't want a specific repo synced, remove it from the workspace.

## Support + Contribution

### Where do I file bugs or feature requests?

For product issues (deployment problems, bugs, feature requests): open an issue on [GitHub](https://github.com/bcarpio/outcome-ops-ai-assist/issues).

For documentation issues (typos, wrong details, missing content): [open an issue in the docs repo](https://github.com/outcomeops/docs/issues/new), or fix it yourself with a pull request. Include the page URL and what's confusing.

### Can I contribute to the docs?

Yes. Issues and pull requests are both welcome on the [docs repo](https://github.com/outcomeops/docs); every page also has an edit link that opens the source file on GitHub. See the [docs repo README](https://github.com/outcomeops/docs) for how to build the site locally.

### Is there a Slack / Discord community?

Not currently. Questions, bugs, and feature requests go through [GitHub Issues](https://github.com/bcarpio/outcome-ops-ai-assist/issues).

### What SLA does OutcomeOps offer?

None. OutcomeOps AI Assist is open-source software with community support and no paid support plan; the availability you get is what your AWS deploy provides. The platform is multi-AZ by default and supports multi-region HA (active/passive with per-region KMS keys). Configure your AWS account for the availability posture you need; the platform inherits it.
