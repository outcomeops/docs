---
title: SIEM Audit Stream Export
description: Stream every audit-log event to your SIEM via a Kinesis Data Stream in OCSF format.
---

# SIEM Audit Stream Export

Every mutation in OutcomeOps AI Assist writes a row to a DynamoDB audit table. That's the built-in audit trail --- readable from the Admin Audit dashboard and from CloudWatch Logs Insights.

If your security team wants those events in **their** SIEM --- Splunk, Datadog, Sumo Logic, Elastic, AWS Security Lake, or a Firehose-to-S3 pipeline --- the platform can export the audit stream over a **Kinesis Data Stream** in **OCSF v1.3.0** format. You attach the consumer of your choice; the platform doesn't prescribe one.

## Architecture

```text
+----------------+     DDB Streams     +-----------------------+
|  audit-logs    |  ---- INSERT ---->  |  audit-stream-        |
|  DynamoDB      |                     |  publisher Lambda     |
+----------------+                     +-----------+-----------+
                                                   |
                                       OCSF v1.3.0 envelope
                                                   |
                                                   v
                                       +-----------+-----------+
                                       |  Kinesis Data Stream  |
                                       |  <env>-<app>-         |
                                       |  audit-export         |
                                       |  (on-demand, 24h)     |
                                       +-----------+-----------+
                                                   |
                                                   v
                                       +-----------------------+
                                       |  Your SIEM consumer   |
                                       |  - Splunk add-on      |
                                       |  - Firehose -> S3     |
                                       |  - Lambda -> Sumo /   |
                                       |    Datadog / Elastic  |
                                       |  - Security Lake      |
                                       +-----------------------+
```

Every event that lands in the audit table flows through the publisher Lambda, wrapped in an OCSF envelope, and lands on the Kinesis stream. Your consumer reads the stream and forwards to your SIEM.

## Why Kinesis Data Streams, not Firehose

Kinesis Data Streams (KDS) keeps the platform agnostic about your consumer choice. Firehose is "fire and forget to S3 / Splunk" --- fine if that's what you want, but limiting if you need to fan out to multiple destinations or process events before landing. KDS lets you:

- Attach a Splunk add-on that pulls from the stream
- Attach a Datadog forwarder
- Attach a Lambda that fans out to multiple SIEMs
- Attach a Firehose (if you actually want the Firehose pattern) --- your Firehose reads from KDS
- Attach AWS Security Lake

The stream is your integration point; the platform doesn't lock you into one destination.

## Enabling

One tfvar:

```hcl
audit_stream_export_enabled = true
```

That's it. Run `terraform apply` and the stream + publisher Lambda land.

CMK encryption on the stream flows automatically from your existing `enable_cmk_encryption` toggle:

| `enable_cmk_encryption` | Stream encryption |
| --- | --- |
| `false` (default) | AWS-managed `alias/aws/kinesis` |
| `true` | Platform CMK (same key as DDB / S3 / SQS) |

After apply, discover the stream ARN via Terraform output or SSM:

```bash
aws ssm get-parameter \
  --name "/prd/outcome-ops-ai-assist/audit-export/stream-arn" \
  --query 'Parameter.Value' --output text
```

The stream is **on-demand mode** (no shard math, scales automatically) with **24-hour retention**. Consumers who want longer in-stream retention should spin up their own Firehose consumer and land events in S3 with their own lifecycle policy.

## OCSF envelope

Every event is wrapped in OCSF v1.3.0. Consumers use OCSF's `category_uid` + `class_uid` + `activity_id` to filter for the event types they care about. The raw OutcomeOps payload is preserved under `unmapped` for consumers who want the full detail.

An abbreviated example event (login):

```json
{
  "metadata": {
    "product": {"name": "OutcomeOps AI Assist", "vendor_name": "OutcomeOps"},
    "version": "1.3.0",
    "log_name": "audit-logs"
  },
  "category_uid": 3,
  "category_name": "Identity & Access Management",
  "class_uid": 3002,
  "class_name": "Authentication",
  "activity_id": 1,
  "activity_name": "Logon",
  "time": 1720000000000,
  "actor": {
    "user": {"email_addr": "alice@example.com"}
  },
  "src_endpoint": {"ip": "203.0.113.10"},
  "status_id": 1,
  "unmapped": {
    "outcome_ops_action": "ACTION_LOGIN_SUCCEEDED",
    "workspace_id": "",
    "user_agent": "Mozilla/5.0 ..."
  }
}
```

## FilterCriteria patterns for common SIEM tasks

Your consumer's FilterCriteria (Splunk regex, Datadog log processor, Lambda event filter, etc.) can target specific event classes:

- **Every login:** `category_uid = 3 AND class_uid = 3002 AND activity_id = 1`
- **Every workspace deletion:** `unmapped.outcome_ops_action = "ACTION_WORKSPACE_DELETED"`
- **Every OAuth connection:** `unmapped.outcome_ops_action LIKE "ACTION_%_CONNECTED"`
- **Every code generation trigger:** `unmapped.outcome_ops_action = "ACTION_CODE_GENERATION_TRIGGERED"`
- **Every admin membership change:** `unmapped.outcome_ops_action IN ("ACTION_MEMBER_ADDED", "ACTION_MEMBER_REMOVED", "ACTION_MEMBER_ROLE_CHANGED")`

The full list of OutcomeOps action types lives in your deploy's audit table --- query DDB directly to enumerate them, or check the CloudWatch Logs of the workspace-management Lambda for the emit calls.

## Consumer patterns

### Splunk

Use the **Splunk Add-on for AWS**. Configure it to pull from Kinesis Data Streams with the stream ARN. Add-on handles multi-shard fan-out and checkpointing automatically.

### Datadog

Use the **Datadog Forwarder** Lambda subscribed to the Kinesis stream. Datadog receives the events as logs; use log processors to reshape OCSF into your Datadog log pipeline.

### Sumo Logic

Use the **Sumo Logic AWS Lambda Function for Kinesis Streams**. Streams events into a Sumo source with your organization's tags.

### Elastic

Use **Elastic Agent's AWS Kinesis input**. Elastic ingests OCSF; the fields map cleanly into ECS.

### AWS Security Lake

Attach Security Lake's Kinesis subscriber to the stream. Security Lake normalizes OCSF into its Parquet-on-S3 lake for cross-source query via Athena.

### Firehose to S3

If you just want events on S3:

- Create a Firehose delivery stream with Kinesis Data Streams as its source
- Point Firehose at your OutcomeOps audit stream
- Firehose lands events in S3 with your bucket + prefix

## Verify after enabling

```bash
# Fire a test event: sign in to the platform once.
# Then within ~5 seconds, check the stream has an incoming record:

aws kinesis describe-stream-summary \
  --stream-name prd-outcome-ops-ai-assist-audit-export \
  --query 'StreamDescriptionSummary.OpenShardCount'
```

Production consumers (KCL, Firehose, Splunk Add-on for AWS, Lambda event-source mapping) handle multi-shard fan-out transparently. **One-off `aws kinesis get-records` calls do not** --- they target one shard at a time, and on-demand mode silently scales the stream as traffic grows. If you sample the stream by hand, fan across every shard or you'll quietly see only ~1/N of the traffic.

## Cost

Kinesis Data Streams on-demand mode charges per GB written and per GB read. Ballpark for a typical customer with active platform usage: a few dollars per month for the stream itself. Your consumer's cost varies by choice (Firehose $, Splunk Add-on free, Datadog per-log-line, etc.).

## Disabling

Set `audit_stream_export_enabled = false` and re-apply Terraform. The publisher Lambda + Kinesis stream get destroyed. Events continue landing in the DynamoDB audit table (that's the base audit trail; the stream is an export mechanism on top).

## Related

- The DynamoDB audit table is the base audit surface. See the deploy's Admin → Audit dashboard for the built-in queryable view.
- [Integration Gating](integration-gating.md) --- audit stream export is an admin flag, not an integration flag; it stays gated by `audit_stream_export_enabled` regardless of which integrations are enabled.
