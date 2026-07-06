---
title: Database
description: Connect PostgreSQL, MySQL, or SQL Server so the platform can query schemas + tables as chat context.
---

# Database

The Database integration connects OutcomeOps AI Assist to your relational databases and makes **schemas + tables** queryable as chat context. Ask "what columns does the customers table have?" or "which tables reference user_id?" and the platform answers from live schema introspection, not from a stale doc.

Time budget: **~20 minutes per database connection**.

## What this connects

- **Table + column metadata.** Column names, types, nullability, comments.
- **Schema-level relationships.** Foreign keys, primary keys, indexes.
- **Row counts** (approximate) at introspection time.
- **NOT ingested:** table row data. The platform reads structure, never content.
- **Schema-drift notifications.** An SNS topic fires when a DBA adds or drops a schema between syncs; table-level changes inside already-selected schemas are picked up automatically.

## Supported engines

| Engine | Versions | Auth |
| --- | --- | --- |
| **PostgreSQL** | 12+ | Password (via SSM SecureString) |
| **MySQL** | 8.0+ | Password (via SSM SecureString) |
| **SQL Server** | 2019+ | Password (via SSM SecureString) |

## Design principles

- **Read-only.** The platform never writes to your databases. Give it a read-only user.
- **Schemas-only scope.** You pick which schemas the platform can see; it auto-discovers tables within selected schemas but never sees anything else.
- **BYO CA bundle.** For databases behind private CAs (typical in regulated deploys), you upload your CA bundle to SSM and the connection layer trusts it. No mucking with Lambda base images.

## Prerequisites

- **Database admin access** to create a read-only user.
- **Network path from the platform's VPC to the database.** Typically a peering, Transit Gateway, or VPN connection. The platform's Fargate + Lambda subnets need to reach the database on its port.
- **`enable_database_integration = true`** in the tfvars.

## Step 1: Create a read-only database user

The platform reads schema metadata via a dedicated read-only user, never the database owner or a shared app user.

### PostgreSQL

```sql
CREATE USER outcomeops_reader WITH PASSWORD '<strong-password>';
GRANT CONNECT ON DATABASE your_db TO outcomeops_reader;

-- For each schema you want to expose:
GRANT USAGE ON SCHEMA public TO outcomeops_reader;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO outcomeops_reader;

-- Grant on future tables too:
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT ON TABLES TO outcomeops_reader;
```

### MySQL

```sql
CREATE USER 'outcomeops_reader'@'%' IDENTIFIED BY '<strong-password>';
GRANT SELECT ON your_db.* TO 'outcomeops_reader'@'%';
FLUSH PRIVILEGES;
```

### SQL Server

```sql
USE master;
CREATE LOGIN outcomeops_reader WITH PASSWORD = '<strong-password>';

USE your_db;
CREATE USER outcomeops_reader FOR LOGIN outcomeops_reader;
ALTER ROLE db_datareader ADD MEMBER outcomeops_reader;
GRANT VIEW DEFINITION ON SCHEMA :: dbo TO outcomeops_reader;
```

## Step 2: Store the password in SSM

The platform reads credentials from SSM at connect time. Put the password in a SecureString:

```bash
aws ssm put-parameter \
  --name "/prd/outcome-ops-ai-assist/database/production-postgres-password" \
  --value "<the password from Step 1>" \
  --type SecureString \
  --key-id alias/aws/ssm \
  --region us-west-2
```

Use a name that identifies the specific database (`production-postgres-password`, `analytics-mysql-password`). One parameter per database connection.

## Step 3: (Optional) Upload your CA bundle

If your database sits behind a private CA (typical for enterprise-managed RDS or Aurora deployments with internal certificate authorities), upload the CA bundle:

```bash
aws ssm put-parameter \
  --name "/prd/outcome-ops-ai-assist/database/ca-bundle" \
  --value "$(cat internal-ca-bundle.pem)" \
  --type String \
  --region us-west-2
```

The platform reads the CA bundle at connection time and uses it for TLS verification. If your database uses a public CA (AWS RDS's default), skip this step.

## Step 4: Enable the flag and deploy

```hcl
enable_database_integration = true

# Optional -- schema-drift SNS topic subscription
database_drift_notification_emails = ["dba-team@example.com"]
```

```bash
terraform apply -var-file=prd.tfvars
```

See the [Deploy guide](../getting-started/deploy.md).

## Step 5: Create a connection from the UI

1. Open the OutcomeOps UI as a **workspace admin**.
2. Navigate to **Workspace Settings → Integrations**.
3. Under Database, click **New connection**.
4. Fill in:
    - **Name:** human-readable label (e.g., "Production Postgres").
    - **Engine:** PostgreSQL / MySQL / SQL Server.
    - **Host + port + database.**
    - **Username:** `outcomeops_reader` (from Step 1).
    - **Password SSM parameter:** the parameter name from Step 2 (e.g., `/prd/outcome-ops-ai-assist/database/production-postgres-password`).
    - **CA bundle SSM parameter (optional):** from Step 3.
5. Click **Test connection**. The platform validates credentials + reachability. If it fails, see Common problems below.
6. Click **Save**.

## Step 6: Select schemas to expose

1. On the saved connection, click **Configure schemas**.
2. The picker shows every schema the read-only user can see.
3. Check the schemas you want the platform to make queryable. Uncheck the rest.
4. Save. Initial introspection fires immediately; subsequent syncs run hourly.

Tables inside selected schemas are auto-discovered on each sync. Add a new table to a selected schema and it appears in chat within an hour --- no reconfiguration needed.

## Common problems

| Error | Cause | Fix |
| --- | --- | --- |
| `Connection refused` on Test | Network path from the platform's VPC to the database is blocked. | Verify security groups, NACLs, and route tables. The platform's Fargate + Lambda subnets need outbound access to the database's port. |
| `password authentication failed` | Password in SSM doesn't match the user's password. | Re-run the SSM `put-parameter --overwrite` with the correct password. Trim any accidental whitespace. |
| `certificate verify failed` | Database uses a private CA, and you didn't upload the CA bundle. | Upload the CA bundle per Step 3, edit the connection, add the SSM parameter path. |
| `permission denied for schema X` | The read-only user doesn't have `USAGE` on schema X. | Re-run the Step 1 grants for schema X. |
| Schemas missing from the picker | User doesn't have visibility on those schemas. | Grant the user access to the schemas, then click Refresh. |
| Drift SNS never fires | You didn't subscribe an endpoint to the drift topic, OR the first apply's SubscribeNow confirmation email was ignored. | Check the AWS SNS console for the topic named `*-database-drift`, subscribe an email endpoint, confirm the subscription in the email you receive. |

## Rotating the password

1. Change the password on the database user.
2. Run `aws ssm put-parameter --overwrite` on the parameter with the new value.
3. Wait ~15 minutes for the next Lambda cold start, or manually trigger a sync from the UI to force the new value to load.

No Terraform apply needed.

## Disconnecting

Workspace Settings → Integrations → find the connection → **Disconnect**. The connection record is deleted; ingested schema metadata is removed from the workspace's KB. The read-only user still exists on your database until you drop it manually.

## What the flag gates

Setting `enable_database_integration = false` provisions no Database integration Lambdas, no SQS queues, no scheduler, no drift SNS topic. See [Integration Gating](../administration/integration-gating.md).
