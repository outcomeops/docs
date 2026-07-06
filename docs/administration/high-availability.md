---
title: High Availability
description: Active/passive multi-region topology, failover procedure, and failback.
---

# High Availability

OutcomeOps AI Assist supports **active/passive multi-region HA**. One region serves traffic; the other stands by, kept in sync via cross-region replication. Failover is an **operator action** --- a human flips two AppConfig role flags and updates DNS to point at the secondary region. There is no automatic failover; the platform is designed so a human can decide "this outage is real, flip now" and execute a bounded procedure.

This page covers the runbook.

## Topology

| Region | Role | Traffic |
| --- | --- | --- |
| Primary (e.g., `us-west-2`) | `active` | All chat, ingest, code generation |
| Secondary (e.g., `us-east-2`) | `passive` | Replication applier only; scheduled work does not run |

Both regions are provisioned by Terraform. When `secondary_region` is set in your tfvars, the deploy provisions ~30 secondary-region resources (DDB, SQS, S3, S3 Vectors, AppConfig, KMS, IAM, replicator applier Lambda) under the `aws.secondary` provider. Single-region deploys leave `secondary_region = ""` and skip this entirely.

## What replicates

Every mutation in the active region emits an EventBridge event on the platform's replication bus. A per-region **replication applier Lambda** in the passive region consumes those events and applies the mutation to its local DynamoDB tables + S3 buckets. Result: the passive region's state trails the active region by seconds under normal load.

**No global tables.** The topology avoids DynamoDB global tables intentionally --- global tables had a public outage in October 2025 that caused cross-region failures for many customers, and the failure mode surfaced only during failover. The current design uses per-region tables with EventBridge as the transport, so a region-level DDB outage stays contained.

## Enabling HA

Set the secondary region in your tfvars and re-apply Terraform. Everything else the platform needs (KMS replica keys, per-region SSM SecureStrings, AppConfig role flags) is provisioned as part of the same apply.

```hcl
secondary_region                        = "us-east-2"
outcomeops_license_layer_arn_secondary  = "arn:aws:lambda:us-east-2:<layer-account>:layer:prd-outcomeops-license-validator-v2-0-0:4"
ui_vpc_id_secondary                     = "vpc-xxxxxxxxxxxxxxxxx"
ui_private_subnet_ids_secondary         = ["subnet-xxx", "subnet-xxx", "subnet-xxx"]
ui_container_image_secondary            = "<account>.dkr.ecr.us-east-2.amazonaws.com/prd-outcome-ops-ai-assist-chat-ui:latest"
```

The `_secondary` variants of the OAuth SSM parameters (Atlassian client secret, Microsoft client secret, GitHub App private key, etc.) get their initial values via a one-time bootstrap script the operator runs after the first HA apply --- see the operator handoff kit for the exact commands.

## Day-to-day operation

- The **active** region runs everything: chat, integration OAuth callbacks, sync workers, schedulers, code-generation, MCP catalog. Users hit the active region's UI via DNS.
- The **passive** region runs the replication applier only. Its Lambdas exist and are warm, but chat writes and scheduled work do not fire. Reads against the passive region's data are allowed but the UI is not routed there.

Check status any time:

```bash
AWS_PROFILE=ops ENV=prd make ha-status
```

The output tells you:

- Which region is active and which is passive
- Replication lag in seconds
- DLQ depth (any events the applier couldn't apply)
- Verdict: **SAFE TO FLIP** / **UNSAFE (DLQ has messages)** / **SPLIT-BRAIN**

Green verdict means both regions agree on roles and no replication events are stuck. Any other verdict, don't attempt failover --- investigate first.

## Failover procedure

Use this when the primary region is unhealthy and you need to route traffic to the secondary. Ballpark: **5-10 minutes** end-to-end for a human comfortable with the console.

### Step 1: Confirm safety (or accept the tradeoff)

```bash
AWS_PROFILE=ops ENV=prd make ha-status
```

Ideal: verdict says `SAFE TO FLIP`. In a real outage where the primary region is unreachable, you won't get this signal --- you're doing an emergency failover. Skip to step 2 and accept that some in-flight events may be lost.

### Step 2: Flip the secondary region's AppConfig role to `active`

Open the AWS Console for the secondary region:

**AppConfig → Applications → `${env}-${app}-ha` → Environment `${env}` → Profile `active-region` → Create new hosted configuration version**

Content:

```json
{
  "schema_version": 1,
  "role": "active",
  "activated_at": "<current ISO 8601 timestamp>"
}
```

Start a deployment with the `${env}-${app}-ha-instant` deployment strategy. AppConfig pushes the change to every Lambda in the secondary region within seconds.

### Step 3: Flip the primary region's AppConfig role to `passive`

Same procedure, but in the primary region's AppConfig:

```json
{
  "schema_version": 1,
  "role": "passive",
  "activated_at": "<current ISO 8601 timestamp>"
}
```

If the primary region is unreachable (a real outage), skip this step. `make ha-status` will flag the split-brain when the primary recovers, and you complete this step retroactively.

### Step 4: Update DNS

DNS is currently manual --- update your Route 53 record (or your external DNS provider) so `<your-app-fqdn>` resolves to the secondary region's ALB.

If you're using Route 53 with alias records:

- Change the alias target from the primary ALB to the secondary ALB
- TTL of 60s on the record recommended so cutover is fast

If you're using external DNS (Cloudflare, Azure DNS, on-prem BIND):

- Update the record to the secondary ALB's DNS name
- Wait for TTL to expire

Once DNS propagates, users hitting `<your-app-fqdn>` land on the secondary region.

### Step 5: Confirm clean state

```bash
AWS_PROFILE=ops ENV=prd make ha-status
```

You should see:

- Exactly one region active (the secondary)
- Exactly one region passive (the primary, if reachable; otherwise flagged as unreachable)
- No split-brain

## Failback procedure

Run this after the primary region recovers and you want to route traffic back. Same steps in reverse, executed only after replication has caught up.

1. **Confirm in-sync + empty DLQs:**

    ```bash
    AWS_PROFILE=ops ENV=prd make ha-status
    ```

    Verdict must be `SAFE TO FLIP`. If DLQ has messages, drain them first (see below).

2. **Flip the becoming-passive (currently active) region to `passive` FIRST.**

3. **Flip the becoming-active (currently passive) region to `active`.**

4. **Update DNS back to the primary region's ALB.**

Order matters: flipping the new-passive first minimizes the split-brain window. If both regions briefly read `passive`, no scheduled work runs for the in-between minutes --- that's fine, the next tick after re-activation catches up.

## Troubleshooting

### DLQ has messages

`make ha-status` exits with code 2. Some events couldn't apply in the passive region. Investigate the root cause (usually a KMS grant misalignment or an IAM policy drift between primary and secondary), fix it, then replay:

```bash
AWS_PROFILE=ops ENV=prd \
  REGION=us-east-2 \
  QUEUE=prd-outcome-ops-ai-assist-replication-applier-dlq \
  ARGS="--dry-run" \
  make ha-replay-dlq
```

Inspect the dry-run output; if it looks right, drop `ARGS="--dry-run"` and rerun to actually replay.

Three DLQs exist, one per direction:

- **`${env}-${app}-replication-to-secondary-dlq`** (in primary region) --- events primary couldn't deliver to secondary's bus.
- **`${env}-${app}-replication-to-primary-dlq`** (in secondary region) --- events secondary couldn't deliver to primary's bus.
- **`${env}-${app}-replication-applier-dlq`** (in either region) --- events the applier Lambda failed to apply locally.

### Split-brain

`make ha-status` exit code 3: both regions read `active`. This is an unsafe state --- both regions could be writing conflicting data. Resolve by choosing a winning region (usually the one that's healthy) and flipping the other to `passive` immediately. Then investigate what caused the split-brain (usually an incomplete failover procedure).

### DNS TTL is too long

If your DNS TTL is > 60 seconds, users get pinned to the old region for the TTL window during failover. This isn't a bug --- it's how DNS works --- but you can shorten TTL as a preparedness action *before* an actual failover. Alias records at Route 53 already have low TTLs; if you're using external DNS, verify the record's TTL is short.

### Replication is lagging by more than a minute

Under normal load, replication should be sub-second. Sustained lag usually means the applier Lambda is throttled by concurrent-execution limits or the EventBridge bus is backed up. Check CloudWatch metrics for `replication-applier`, and consider increasing the Lambda's reserved concurrency if throttling is the cause.

## Design tradeoffs worth knowing

- **Chat writes are gated to the active region.** Users hitting the passive region's URL directly (bypassing DNS) can read past conversations but cannot post new messages. This is the intentional passive-mode contract.
- **The 12-integration surface duplicates in the secondary region.** Every integration's Lambdas + SQS + scheduler have a `_secondary` counterpart. Cost implication: HA roughly doubles Lambda + SQS spend, since both regions run the same infra. It's a design choice --- true active/passive means no compromises on capacity if failover is needed.
- **DNS is not automated.** Automating DNS failover requires a health-check + Route 53 failover routing setup that many regulated customers explicitly don't want (they want a human in the loop for cross-region traffic changes). If you want automated failover, set up Route 53 failover routing pointing at both ALBs with health checks --- the AppConfig flip becomes automatic-in-response but you still control the record.

## Related

- [Deploy](../getting-started/deploy.md) --- HA tfvars are documented here.
- [Integration Gating](integration-gating.md) --- HA does not gate integrations; whatever's enabled in the primary is duplicated in the secondary.
