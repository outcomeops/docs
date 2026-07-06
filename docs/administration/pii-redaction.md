---
title: PII Redaction
description: Comprehend-based PII detection and redaction at ingest time, with org-level and workspace-level controls.
---

# PII Redaction

OutcomeOps AI Assist can detect and redact personally identifiable information (PII) at **ingest time** --- before content lands in the vector index, the code-map graph, or the DynamoDB rows the LLM synthesizes from. Detection uses AWS Comprehend; redaction replaces matched spans with type markers (`[EMAIL]`, `[SSN]`, `[NAME]`).

This is the compliance story: content the platform ingests never contains raw PII if redaction is enabled, so retrieved context sent to the LLM is already clean.

## Two levels of control

**Org-level toggle** (deploy-wide, tfvar):

```hcl
pii_redaction_enabled = true
```

When `true`, redaction runs on every ingest, across every workspace. When `false`, no redaction happens anywhere in the deploy --- content lands raw.

**Workspace-level privacy mode** (per-workspace, UI):

- **STRICT** --- redaction runs on this workspace's content regardless of the org-level toggle. Use for workspaces that ingest higher-risk content (HR, customer data, healthcare).
- **NONE** --- redaction is skipped for this workspace even if the org toggle is on. Use for workspaces where redaction would break the use case (a workspace of technical fixture data, for example).
- **INHERIT** (default) --- follow the org-level toggle. If `pii_redaction_enabled = true`, redact; if `false`, don't.

Setting the org toggle to `true` and leaving every workspace as INHERIT is the standard configuration. Escalate individual workspaces to STRICT or drop them to NONE only when the workspace's content demands it.

## What Comprehend detects

Out of the box, Comprehend detects these entity types (see the [AWS Comprehend documentation](https://docs.aws.amazon.com/comprehend/latest/dg/how-pii.html) for the complete list):

- Person names, addresses, ages
- Email addresses, phone numbers, URLs
- SSN, US bank account, credit card, PIN
- Passport, driver's license, government identifiers
- IP addresses, MAC addresses
- Dates and times
- Financial identifiers (routing numbers, SWIFT codes)
- International tax identifiers
- Vehicle identification numbers

Each detected span gets replaced with a type marker (`[EMAIL]`, `[NAME]`, `[SSN]`) at ingest.

## Skip types

Some entity types are false-positive-prone for developer content --- `URL` matches every link in a README, `IP_ADDRESS` matches the example IPs in a networking doc, `DATE_TIME` matches the copyright year in every LICENSE file. You skip these via a tfvar:

```hcl
pii_redaction_skip_types = ["URL", "IP_ADDRESS", "MAC_ADDRESS", "DATE_TIME"]
```

Skipped types pass through unredacted even when redaction is enabled. The default skip list above is tuned for developer knowledge bases; adjust it for your content.

## Cost consideration

Comprehend charges per 100 characters analyzed (per its pricing page). For a large ingest --- an entire monorepo of docs, thousands of Confluence pages --- the Comprehend bill can be meaningful. Two mitigations:

- **Skip types aggressively.** Every skip type reduces the effective character count Comprehend has to consider carefully.
- **Turn redaction off for lower-risk workspaces.** A workspace that only ingests open-source repos probably doesn't need PII detection; drop it to NONE mode to skip Comprehend entirely for that workspace.

Solo developer deploys often set `pii_redaction_enabled = false` at the org level and re-enable on specific workspaces via STRICT mode --- the inverse of the enterprise default.

## What PII redaction does NOT do

- **Not a substitute for access control.** Redaction protects content in the platform's KB from ending up in an LLM prompt. It does not restrict who can query the workspace --- workspace membership + public/global flags do that.
- **Not retroactive.** Redaction runs at ingest. Content already in the KB when you flip the flag stays as it was until re-ingested. To re-run redaction on already-ingested content, trigger a full re-sync of the affected integrations.
- **Not perfect.** Comprehend has a documented false-positive and false-negative rate. For content where a missed detection is unacceptable, you need additional controls (access limits on the source system, redaction at the source before ingest, or a manual review workflow).
- **Not visible in the source system.** The platform redacts a copy in its KB; the source (GitHub file, Confluence page, Jira issue) is untouched.

## Workspace-level privacy mode

Set from Workspace Settings → Privacy. Requires workspace admin.

Effective mode is computed as:

| Org toggle | Workspace setting | Effective |
| --- | --- | --- |
| `false` | INHERIT | No redaction |
| `false` | STRICT | Redaction runs |
| `false` | NONE | No redaction |
| `true` | INHERIT | Redaction runs |
| `true` | STRICT | Redaction runs |
| `true` | NONE | No redaction |

The UI shows the current effective mode + explains why (org / workspace setting / inherited).

## Verifying redaction is working

Two checks after enabling:

1. **Ingest a doc with known PII** (an email address, a phone number) into a workspace with redaction enabled.
2. **Query the platform** with a search that would return that doc. Check the citation --- the platform should return the doc with the PII replaced by `[EMAIL]` / `[PHONE]` markers.

If the source system still shows the raw PII, that's expected --- redaction runs on the platform's copy in its KB, not on the source.

## Common problems

| Symptom | Cause | Fix |
| --- | --- | --- |
| Every doc has `[URL]` markers instead of links | `URL` isn't in `pii_redaction_skip_types` | Add `URL` to the skip list, re-apply tfvars, re-sync content |
| Sensitive PII still appearing in retrievals | Content ingested before `pii_redaction_enabled = true` was applied | Trigger a full re-sync on the affected workspaces from the Integrations panel |
| Comprehend cost is high | Redaction is running on high-volume workspaces where it's not needed | Set those workspaces to NONE mode; the org toggle stays on for the rest |
| Redaction status unclear per workspace | Workspace admin doesn't know which mode is effective | UI's Workspace Settings → Privacy page shows the effective mode + explanation |

## Related

- [Global Workspaces](global-workspaces.md) --- global workspaces inherit the org redaction posture the same as any other workspace.
- [Chat](../features/chat.md#guardrails) --- how redaction interacts with the LLM's context window.
- AWS Comprehend documentation for the [complete list of PII entity types](https://docs.aws.amazon.com/comprehend/latest/dg/how-pii.html) and [pricing](https://aws.amazon.com/comprehend/pricing/).
