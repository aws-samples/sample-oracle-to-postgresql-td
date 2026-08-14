# Security Policy

## Reporting a vulnerability

If you discover a potential security issue in this project, notify AWS Security via our
[vulnerability reporting page](https://aws.amazon.com/security/vulnerability-reporting/) or
email [aws-security@amazon.com](mailto:aws-security@amazon.com).

Please do **not** create a public GitHub issue for a security report, and do not include
customer names, credentials, account identifiers, or production data in your report.

## What "vulnerability" means for this repository

This repository contains a transformation definition (TD) for AWS Transform. It is Markdown
instruction content — there is no application source code, no dependency manifest, and no
deployable infrastructure here, so there is nothing in this repository to exploit directly.

The relevant risk is different: the content is executed by an AI agent that rewrites Java and
Kotlin source in a target repository. Insecure guidance in this repository propagates into
every codebase the TD is run against. Reportable issues therefore include any rule or example
that would cause the agent to:

- Disable or downgrade encryption in transit (for example, producing a PostgreSQL JDBC URL
  without an explicit `sslmode`)
- Write credentials into source files, configuration files, or logs
- Convert a bound query parameter into an inlined literal or a concatenated SQL fragment,
  introducing SQL injection
- Broaden database or IAM privileges beyond what the migration requires
- Weaken schema resolution safety, such as `search_path` ordering that allows an object in an
  earlier schema to shadow a function the application calls
- Silently disable tests that cover authorization, input validation, or constraint enforcement
- Introduce real customer data into test fixtures

## Security considerations when using this TD

Read these before running the transformation against your own repository.

### Run the agent with review, not blind trust

The AWS Transform CLI examples in the README use `-t` (trust all tools) with `-x`
(non-interactive). That grants an agent unattended write access to your source tree. Run the
first transformation without `-t` so you can see which tools are requested, work on a clean
branch of a repository you own, and review the full diff on the generated staging branch
before merging.

### Credentials

Use short-lived credentials — IAM Identity Center (`aws sso login`) or an assumed role — rather
than long-lived IAM user access keys. Scope permissions to what the transformation needs; the
`AWSTransformCustomFullAccess` managed policy is a quick-start convenience, not a
least-privilege grant for shared or production accounts.

Database credentials belong in AWS Secrets Manager or SSM Parameter Store, or use IAM database
authentication for Amazon RDS and Aurora. Migration is a common moment for passwords to end up
committed in `application.yml` — check your diff for this specifically.

### Encryption in transit

The PostgreSQL JDBC driver does not negotiate TLS by default, whereas the Oracle deployment you
are migrating from may have had encryption enabled. Set `sslmode` explicitly on the new
connection URL (`verify-full` with the Amazon RDS CA bundle for RDS and Aurora) and confirm the
connection is encrypted after migration.

### orafce and `search_path`

The TD recommends placing the `oracle` schema first in `search_path` so orafce compatibility
functions resolve. Because PostgreSQL resolves unqualified names in `search_path` order, any
role that can create objects in an earlier schema can shadow functions your application calls.
Install orafce as a superuser or `rds_superuser`, keep the `oracle` and `public` schemas owned
by an administrative role, and revoke `CREATE` from application roles
(`REVOKE CREATE ON SCHEMA public FROM PUBLIC`).

### Review the migration diff for security regressions

Beyond compilation and tests, check the diff for: connection strings that lost TLS settings,
new plaintext secrets, tests newly marked `@Ignore`, broadened exception handling that swallows
failures, and any query where a parameter placeholder became a literal.

## Supported versions

Security fixes are applied to the `main` branch. There are no released or supported historical
versions of this content.
