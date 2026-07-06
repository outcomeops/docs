---
title: Security Scanner Endpoint
description: Password-authenticated login for AWS Security Agent and other headless scanners that can't do Azure AD OIDC.
---

# Security Scanner Endpoint

Most users sign in to OutcomeOps AI Assist through Azure AD OIDC. But **headless security scanners** --- AWS Security Agent, some AI-driven DAST tools, third-party red-team automation --- can't do interactive OIDC. They need a password-authenticated login that hands them a session cookie the platform accepts as an auth alternative.

The **security scanner endpoint** solves this. When enabled, Terraform provisions:

- A **secret-path Lambda** behind the UI's ALB (path is a 128-bit random hex, stored in SSM --- not guessable from the outside)
- A **login form** at that path (GET) + credential validator (POST)
- A **JWT-signed session cookie** that the Express proxy accepts alongside Azure AD sessions
- **Randomly-generated credentials** (path, password, JWT signing key, custom header key), stored in SSM SecureString

Scanners fetch the path + credentials from SSM (via their own IAM role), POST to the login endpoint, get back a JWT cookie, and use it to authenticate every subsequent request.

## When to enable this

Enable if you're running:

- **AWS Security Agent** in scanning mode against your deploy
- **AI-driven DAST tools** that require interactive login (many can't handle OIDC redirects)
- **Third-party pentesting automation** contracted through your SecOps team
- **Any headless scanner** that needs authenticated access but can't complete an OIDC flow

Do NOT enable this for:

- Human user access --- they should use OIDC
- Machine-to-machine API calls from your own infrastructure --- use IAM-signed Function URL calls instead
- Read-only monitoring that doesn't need authentication --- the health / status endpoints are public

## Enabling

One tfvar:

```hcl
security_scanner_enabled = true
```

Requires `deploy_ui = true` (the scanner endpoint lives on the UI ALB). Then `terraform apply`.

Terraform generates four random secrets and stores them in SSM SecureString:

| SSM parameter | Purpose |
| --- | --- |
| `/<env>/<app>/security-scanner/path` | The secret URL path (128-bit hex; the scanner login page lives at `/<this>`) |
| `/<env>/<app>/security-scanner/password` | The scanner's password (32 chars random) |
| `/<env>/<app>/security-scanner/jwt-secret` | The signing key for the JWT cookie (64 chars random) |
| `/<env>/<app>/security-scanner/header-key` | A custom-header key the scanner must include on every request (32 chars random) |

**These are auto-generated and stored securely.** You never see them in plaintext during `terraform apply`. The scanner fetches them from SSM via its own IAM role (see below).

## How the scanner uses the endpoint

### Step 1: The scanner reads credentials from SSM

Give your scanner an IAM role with `ssm:GetParameter` on the four scanner parameters, plus `kms:Decrypt` on the KMS key that encrypts them (usually `alias/aws/ssm`).

```bash
# Scanner's IAM policy needs:
{
  "Effect": "Allow",
  "Action": ["ssm:GetParameter"],
  "Resource": "arn:aws:ssm:*:*:parameter/prd/outcome-ops-ai-assist/security-scanner/*"
},
{
  "Effect": "Allow",
  "Action": ["kms:Decrypt"],
  "Resource": "arn:aws:kms:*:*:alias/aws/ssm"
}
```

The scanner runs:

```bash
# Fetch the path, password, and header key from SSM
SCANNER_PATH=$(aws ssm get-parameter --with-decryption \
  --name "/prd/outcome-ops-ai-assist/security-scanner/path" \
  --query 'Parameter.Value' --output text)

SCANNER_PASSWORD=$(aws ssm get-parameter --with-decryption \
  --name "/prd/outcome-ops-ai-assist/security-scanner/password" \
  --query 'Parameter.Value' --output text)

SCANNER_HEADER_KEY=$(aws ssm get-parameter --with-decryption \
  --name "/prd/outcome-ops-ai-assist/security-scanner/header-key" \
  --query 'Parameter.Value' --output text)
```

### Step 2: The scanner POSTs to the login endpoint

```bash
COOKIE=$(curl -s -c - "https://<your-app-fqdn>/${SCANNER_PATH}" \
  -X POST \
  -H "X-Scanner-Auth: ${SCANNER_HEADER_KEY}" \
  -d "password=${SCANNER_PASSWORD}" \
  | grep session_token | awk '{print $7}')
```

The endpoint validates the password + custom header, mints a JWT signed with the SSM-stored secret, and returns it as a cookie.

### Step 3: The scanner uses the cookie on subsequent requests

Every request to the platform includes the cookie:

```bash
curl -H "Cookie: session_token=${COOKIE}" \
     -H "X-Scanner-Auth: ${SCANNER_HEADER_KEY}" \
     "https://<your-app-fqdn>/api/workspaces"
```

The Express proxy accepts the JWT cookie as an authenticated session. The user identity in the session is `scanner@internal` --- audit rows attributed to the scanner get tagged with this synthetic user so SecOps can filter for scanner activity separately from real user activity.

## Session lifetime

The JWT cookie expires after **1 hour** by default. Scanners running longer scans need to re-authenticate (repeat step 2) periodically. This is intentional --- limits the blast radius if a scanner's session cookie is intercepted.

## Custom header key

The `X-Scanner-Auth` header is a **second factor**. Both the password AND the custom header must be correct on the login POST, and the header must be present on every subsequent request. This defends against:

- **Credential leak.** If the password ends up in a log or the SSM parameter is misconfigured, an attacker still needs the header key.
- **Session replay.** If a session cookie is stolen, the attacker still needs the header key.

Both secrets rotate together via Terraform if you re-generate the tfvars-linked randoms (`terraform taint` on the random resources + apply).

## Common problems

| Symptom | Cause | Fix |
| --- | --- | --- |
| 404 on the secret path | Wrong path fetched from SSM, or the `security_scanner_enabled` flag is false. | Confirm the flag is true; re-fetch the path from SSM. |
| 401 on login POST | Wrong password, or missing `X-Scanner-Auth` header, or the header value is wrong. | Fetch fresh values from SSM. Confirm the header key is present. |
| 401 on API calls after login | JWT cookie expired (1-hour lifetime) or missing on the request. | Re-run the login POST and update the cookie. |
| Scanner works from one region but not another (HA deploys) | The random secrets are region-specific. Each region has its own SSM parameters. | Fetch the credentials from the region you're targeting. |

## Rotating scanner credentials

To rotate all four secrets (path, password, JWT signing key, header key):

```bash
terraform taint random_id.scanner_path
terraform taint random_password.scanner_password
terraform taint random_password.scanner_jwt_secret
terraform taint random_password.scanner_header_key
terraform apply -var-file=prd.tfvars
```

Terraform regenerates all four, updates SSM, and the Lambda picks up the new values within one cold-start cycle (~15 minutes of normal traffic, or force immediately with a redeploy).

**During rotation, existing scanner sessions with cookies signed by the OLD JWT secret become invalid immediately.** Scanners currently mid-scan lose auth and must re-login. Plan rotations for a window when no scan is in flight, or accept the interruption.

## Disabling

Set `security_scanner_enabled = false` and re-apply Terraform. The Lambda, ALB listener rule, and SSM parameters get destroyed. Existing scanner sessions are dropped immediately.

## What this does NOT do

- **Not a replacement for OIDC** for human users. Anyone with the scanner credentials can act as `scanner@internal` --- there's no user attribution beyond that identity. Reserve for machine access only.
- **Not multi-scanner.** All scanners share the same credentials in v1. If you need per-scanner attribution, contact your account team --- the design can be extended to multiple named scanner identities.
- **Not integrated with your identity provider.** The scanner is intentionally out-of-band from OIDC / SSO. That's the point --- SSO can't do headless auth.

## Related

- [Deploy](../getting-started/deploy.md) --- `security_scanner_enabled` tfvar sits alongside the other UI flags.
- [Audit Stream Export](audit-stream-export.md) --- if you want scanner activity in your SIEM, the events already flow through the same audit stream tagged with `scanner@internal`.
