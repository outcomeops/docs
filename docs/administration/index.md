---
title: Administration
description: How to operate a deployed OutcomeOps AI Assist instance -- flags, workspaces, redaction, HA, audit stream, model selection.
---

# Administration

This section covers **how to operate** a deployed OutcomeOps AI Assist instance --- distinct from **how to use it** (that's in [Features](../features/index.md)).

Read these when you're setting up governance, satisfying a SecOps review, configuring HA, or wiring in a SIEM:

- **[Integration Gating](integration-gating.md)** --- one flag per integration, all default false. What your SecOps reviewer needs to see when they open your tfvars.
- **[Global Workspaces](global-workspaces.md)** --- the pattern for making one team's content queryable across the whole org without maintaining membership lists.
- **[PII Redaction](pii-redaction.md)** --- Comprehend-based detection + redaction at ingest time, with org-level + workspace-level controls.
- **[Model Selection](model-selection.md)** --- swap Claude for GPT-5.5, update pricing tfvars, rotate Mantle API keys.
- **[High Availability](high-availability.md)** --- active/passive multi-region topology, failover runbook, failback.
- **[SIEM Audit Stream Export](audit-stream-export.md)** --- Kinesis Data Stream + OCSF envelope for streaming audit events to Splunk / Datadog / Security Lake / your SIEM of choice.
- **[Security Scanner Endpoint](security-scanner.md)** --- password-authenticated login for AWS Security Agent and other headless scanners that can't do Azure AD OIDC.
