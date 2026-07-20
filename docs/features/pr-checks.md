---
title: PR Checks
description: Automated AI review checks on GitHub PRs, Azure DevOps PRs, and GitLab MRs, and how to trigger them from CI.
---

# PR Checks

Every enrolled code host gets the same automated review pipeline. When a check run is triggered for a pull request (GitHub, Azure DevOps) or merge request (GitLab), the platform analyzes the changed files and posts results back as comments on the PR/MR:

| Check | What it looks at |
| --- | --- |
| **ADR Compliance** | Lambda handlers + Terraform diffs against the ADRs ingested into your knowledge base |
| **README Freshness** | Whether code changes shipped without documentation updates |
| **Test Coverage** | New handler files without corresponding tests, honoring the repo's `.outcomeops.yaml` test patterns |
| **Breaking Changes** | Modified/removed handlers with downstream dependents |
| **Architectural Duplication** | Similar functionality that already exists in other enrolled repos |
| **License Compliance** | Missing copyright headers, license policy, AI-generated-code detection via commit trailers |

Which checks run is decided per-PR from the changed-file patterns --- a docs-only PR doesn't run test-coverage.

## Comment behavior

Two properties hold on all three providers:

- **Collapsed by default.** Each comment is wrapped in a `<details>` block: reviewers see a one-line status (`⚠️ ADR COMPLIANCE: Found compliance issues...`) and expand only when they want the full findings.
- **Updated in place.** Re-running checks on the same PR (e.g., after pushing a fix) updates the *existing* comments rather than stacking duplicates. One comment per check, always current.

## The trigger contract

Checks are triggered by invoking the `analyze-pr` Lambda. There is no public webhook endpoint for this --- the invoke runs inside your AWS account, authorized by IAM. The payload varies by provider:

=== "GitHub"

    ```json
    {"pr_number": 123, "repository": "owner/repo"}
    ```

=== "Azure DevOps"

    ```json
    {"pr_number": 42, "repository": "org/project/repo",
     "provider": "azure_devops", "connection_id": "<connection-id>",
     "ado_org": "org", "ado_project": "project", "ado_repo": "repo"}
    ```

=== "GitLab"

    ```json
    {"pr_number": 7, "repository": "group/subgroup/project",
     "provider": "gitlab", "connection_id": "<connection-id>"}
    ```

`pr_number` is the MR `iid` for GitLab. `connection_id` is shown in Workspace Settings → Integrations after connecting; it's opaque and not sensitive. AI-generated PRs (from the platform's own code-generation loop) trigger analysis automatically --- no CI wiring needed for those.

## Automating from CI

The common pattern for all three: give the pipeline an AWS identity with exactly one permission --- `lambda:InvokeFunction` on the `*-analyze-pr` function ARN --- and invoke on every PR/MR event. How the pipeline gets that identity depends on your environment:

### GitHub Actions (OIDC, no stored keys)

Best practice: federate GitHub's OIDC provider to an IAM role so no long-lived AWS keys live in GitHub secrets.

1. Create an IAM role with the invoke-only policy and a trust policy on `token.actions.githubusercontent.com` scoped to `repo:YOUR_ORG/YOUR_REPO:*`.
2. Add a workflow triggered on `pull_request` that assumes the role via `aws-actions/configure-aws-credentials` and runs:

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_OIDC_ROLE_ARN }}
    aws-region: us-west-2
- run: |
    aws lambda invoke \
      --function-name prd-outcome-ops-ai-assist-analyze-pr \
      --cli-binary-format raw-in-base64-out \
      --payload "{\"pr_number\": ${{ github.event.pull_request.number }}, \"repository\": \"${{ github.repository }}\"}" \
      response.json && cat response.json
```

### GitLab CI (OIDC or stored keys)

GitLab CI can also federate keylessly: declare an `id_tokens` block, create an IAM OIDC provider for `https://gitlab.com`, and trust `project_path:group/project:ref_type:branch`. Or use stored `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` CI variables with the invoke-only IAM user. The job itself is in the [GitLab setup guide](../integrations/gitlab.md#merge-request-analysis).

### Azure Pipelines (service connection)

Install the AWS Toolkit extension, create an AWS service connection backed by the invoke-only IAM identity, and add the `AWSShellScript@1` step from the [Azure DevOps setup guide](../integrations/azure-devops.md#optional-post-pr-check-results-from-an-azure-pipeline).

### Other environments

- **Self-hosted runners inside AWS** (EC2/ECS/EKS): skip credentials entirely --- attach the invoke-only policy to the runner's instance profile / task role.
- **CI in a different AWS account** than the OutcomeOps deploy: put the invoke-only role in the deploy account with a trust policy on the CI account, and have the pipeline `sts:AssumeRole` across.
- **Manual / ad-hoc**: anyone with the IAM permission can invoke from a terminal --- useful for retrying a check run without pushing a commit.

## Cost visibility

Each Claude-backed check appends its own usage to the comment (model, tokens, dollar cost), so per-PR review cost is visible right where the review lands. Pricing is env-var-driven for customers with negotiated Bedrock rates.
