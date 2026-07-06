---
title: Model Selection
description: Swap Claude for GPT-5.5, update pricing tfvars, and rotate Mantle API keys for OpenAI-on-Bedrock.
---

# Model Selection

OutcomeOps AI Assist uses **two Bedrock models** per chat turn:

- **Advanced model** --- runs the synthesis step that produces the streamed answer, plus code generation and PR analysis. Higher-cost, higher-quality.
- **Basic model** --- runs auxiliary steps: query rewriting for retrieval, conversation summarization, code-map extraction during ingest. Cheaper, faster.

Defaults:

| Slot | Default | Provider |
| --- | --- | --- |
| Advanced | Claude Sonnet 4.6 | Anthropic on Bedrock |
| Basic | Claude Haiku 4.5 | Anthropic on Bedrock |
| Embedding | Titan Embed Text v2 | Amazon on Bedrock |
| Rerank | Cohere Rerank v3.5 (optional) | Cohere on Bedrock |

All swappable via tfvars. This page covers the common swaps: **OpenAI GPT-5.5 as advanced**, **GPT-5.4 as basic**, and the pricing tfvars that keep cost analytics accurate.

## Why swap

- **Compliance-approved model.** Your organization has vetted one model family and needs the platform to use it exclusively.
- **Regional availability.** Bedrock doesn't offer Claude in the region you deploy to, but does offer GPT-5.5.
- **Cost.** Different model families have different token pricing; a workspace with heavy usage patterns may cost less on one family than another.
- **Quality on your workload.** Some codebases + languages perform better on one model family than another. Test with a small workspace before rolling out.

## Swapping to GPT-5.5 (advanced) + GPT-5.4 (basic)

### Step 1: Confirm region availability

OpenAI on Bedrock is currently **us-east-2 only** at time of writing. Verify:

```bash
aws bedrock list-foundation-models --region us-east-2 --by-provider openai \
  --query 'modelSummaries[].modelId'
```

You should see `openai.gpt-5.5` and `openai.gpt-5.4` in the output. If your deploy is in a different region and OpenAI isn't available there, either move the deploy or stay on Claude.

### Step 2: Update the model tfvars

```hcl
bedrock_advanced_model_id  = "openai.gpt-5.5"
bedrock_basic_model_id     = "openai.gpt-5.4"
```

### Step 3: Update the pricing tfvars

Cost analytics multiply token counts by these prices. Wrong prices = wrong cost dashboards. Update per the current Bedrock us-east-2 rates:

```hcl
bedrock_advanced_input_price  = 5.50   # openai.gpt-5.5 input per million tokens
bedrock_advanced_output_price = 33.00  # openai.gpt-5.5 output per million tokens
bedrock_basic_input_price     = 2.75   # openai.gpt-5.4 input per million tokens
bedrock_basic_output_price    = 16.50  # openai.gpt-5.4 output per million tokens
```

Verify current rates at [aws.amazon.com/bedrock/pricing](https://aws.amazon.com/bedrock/pricing/) --- Bedrock rates do change, and stale tfvars mean stale cost dashboards.

### Step 4: Set the default backend

OpenAI on Bedrock uses the `/v1/responses` API surface, not the Converse API that Claude uses. The shared model client abstraction picks a backend based on the model prefix, but you can pin it explicitly:

```hcl
bedrock_default_backend = "responses"
```

For Claude deployments, this stays `converse` (the default). For OpenAI, `responses` is required.

### Step 5: Cohere Rerank availability

Cohere Rerank v3.5 is **not available in us-east-2**. If you're moving a deploy to us-east-2 for OpenAI, disable rerank:

```hcl
bedrock_rerank_enabled = false
```

The [dual-channel retrieval](../features/chat.md#retrieval-in-one-paragraph) still works without rerank --- you just lose the cross-lane balancing pass. Answer quality is slightly worse but not dramatically so.

### Step 6: Apply

```bash
cd terraform
terraform workspace select prd
terraform apply -var-file=prd.tfvars
```

The apply updates Lambda env vars in place; no artifact rebuild needed.

### Step 7: Verify

Ask a chat question and watch the audit table. The model_id column should show `openai.gpt-5.5` for synthesis and `openai.gpt-5.4` for query rewriting. The cost dashboard should compute costs at the new prices.

## Mantle API key rotation

GPT-5.5, GPT-5.4, and Codex on Bedrock are accessed via **Mantle**, a Bedrock-adjacent inference service. Mantle requires a long-lived API key for authentication --- unlike Claude on Bedrock, which uses standard AWS IAM.

**This is compliance friction for regulated customers.** The Mantle API key is a long-lived credential that lives in SSM SecureString. It's not tied to short-lived IAM tokens. If your compliance regime requires all credentials to have < 24-hour lifetimes, Mantle-backed OpenAI models don't fit.

### Getting a Mantle API key

Follow the AWS Bedrock console instructions to enable OpenAI on Bedrock --- the enablement flow provisions a Mantle API key visible in the console. Copy it and hand it to your platform operator via a secure channel.

### Storing it in SSM

The operator creates the SSM SecureString:

```bash
aws ssm put-parameter \
  --name "/prd/outcome-ops-ai-assist/mantle/api-key" \
  --value "<the Mantle API key>" \
  --type SecureString \
  --key-id alias/aws/ssm \
  --region us-east-2
```

Tfvars reference the parameter name (not the value):

```hcl
mantle_api_key_ssm_param = "/prd/outcome-ops-ai-assist/mantle/api-key"
```

The Lambdas that call OpenAI-on-Bedrock read the key at cold start and cache it in memory.

### Rotation cadence

Rotate the Mantle API key at least quarterly, and any time you suspect it's been exposed. Rotation procedure:

1. **In the Bedrock console**, generate a new Mantle API key. Don't revoke the old one yet --- you'll need overlap.
2. **Update SSM** with the new value:

    ```bash
    aws ssm put-parameter \
      --name "/prd/outcome-ops-ai-assist/mantle/api-key" \
      --value "<new key>" \
      --type SecureString \
      --overwrite \
      --region us-east-2
    ```

3. **Wait ~15 minutes** for existing warm Lambdas to age out via cold start (or force a redeploy by touching an env var in Terraform).
4. **Verify the new key works** --- ask a chat question, confirm the response streams normally.
5. **Revoke the old key** in the Bedrock console.

Don't revoke the old key before confirming the new one works. Worst case during rotation: SSM updates between two Lambda invocations and you see a few seconds of "old cached key" --- harmless because both keys are valid until the old one is revoked.

### Bedrock API key alternative

For some deployments, Bedrock accepts a long-lived API key directly (not going through Mantle). If your account is eligible, the same rotation procedure applies --- the SSM parameter path just holds a Bedrock API key instead of a Mantle key.

## Switching back to Claude

Reverse the tfvars:

```hcl
bedrock_advanced_model_id  = "us.anthropic.claude-sonnet-4-6"
bedrock_basic_model_id     = "us.anthropic.claude-haiku-4-5-20251001-v1:0"
bedrock_default_backend    = "converse"

bedrock_advanced_input_price  = 3.00
bedrock_advanced_output_price = 15.00
bedrock_basic_input_price     = 1.25
bedrock_basic_output_price    = 2.50

bedrock_rerank_enabled = true   # if your region supports Cohere Rerank
```

Apply and verify. If you had a Mantle key in SSM, you can leave the parameter --- it's ignored when the Claude backend is active. Or delete it as part of cleanup.

## What NOT to swap without care

- **`bedrock_embedding_model_id`** --- changing the embedding model requires **re-embedding every ingested document**. The existing S3 Vectors index becomes incompatible. This is an architectural swap, not a tfvars change; contact your account team.
- **Swapping mid-deploy without a plan.** Users mid-conversation get inconsistent model behavior across turns. Announce swaps ahead of time to your users if the change is user-visible.

## Cost analytics after a swap

The Admin → Cost dashboard sources token counts from the audit table and multiplies by the prices in your tfvars. Prices are captured at query time, not model-invocation time, so **historical cost data doesn't update retroactively** when you change prices. New rows use the new prices; old rows retain whatever prices were in effect when they were written.

This is intentional --- if you change models in the middle of a month, the dashboard shows the actual cost mix accurately (not "here's what it would have cost if the whole month used the new model").

## Related

- [Deploy](../getting-started/deploy.md) --- the model tfvars are set at deploy time and can be updated by re-applying.
- [Chat](../features/chat.md#model-selection) --- brief user-facing mention of the two-model architecture.
- Bedrock pricing: [aws.amazon.com/bedrock/pricing](https://aws.amazon.com/bedrock/pricing/)
- Bedrock model access: enable in the AWS Bedrock console per region + per model.
