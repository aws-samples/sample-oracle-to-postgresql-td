# Oracle to PostgreSQL — Transformation Definition for AWS Transform

A comprehensive transformation definition (TD) for [AWS Transform](https://docs.aws.amazon.com/transform/latest/userguide/custom-get-started.html) that migrates Java application code from Oracle Database to PostgreSQL.

## What it does

This TD converts Oracle-specific SQL syntax, JDBC patterns, ORM configuration, and test infrastructure to PostgreSQL-compatible equivalents across your entire codebase. It covers:

- Sequence syntax (`seq.NEXTVAL FROM DUAL` → `nextval('seq')`)
- Oracle functions (NVL, DECODE, ROWNUM, SYSDATE, TRUNC, ADD_MONTHS)
- Outer join `(+)` → ANSI LEFT JOIN
- DELETE/UPDATE syntax differences
- BigDecimal → Number interface for JDBC return types
- Hibernate dialect and configuration changes
- Testcontainers migration (Oracle → PostgreSQL)
- Column alias case sensitivity (UPPERCASE → lowercase)
- setParameterList type matching for INTEGER columns
- Entity field type vs database column type alignment
- PostgreSQL transaction abort cascading handling
- Reserved words as column names (double-quoting)
- 71 transformation rules total

## Supported technology stacks

| Component | Supported Variants |
|---|---|
| Language | Java 8+, Kotlin (JVM) |
| ORM | Hibernate 3.x/4.x, Doma2, JPA |
| Framework | Spring Boot 2.x/3.x, Spring Batch |
| Build | Gradle, Maven |
| Test | JUnit 4/5, DBUnit, DbSetup, AssertJ, Testcontainers |
| Connection Pool | HikariCP |
| Migration Tool | Flyway |
| Compatibility | orafce extension (optional) |

## Prerequisites

### Supported platforms

- Linux (all distributions)
- macOS
- Windows via WSL (native Windows not supported)

### Required software

- **Node.js 22 or later** — [Download](https://nodejs.org/en/download)
- **Git** — your project must be in a Git repository

### Install the AWS Transform CLI

```bash
curl -fsSL https://transform-cli.awsstatic.com/install.sh | bash
```

If your organisation requires you to review installers before running them, download first:

```bash
curl -fsSL https://transform-cli.awsstatic.com/install.sh -o install.sh
less install.sh          # review
bash install.sh
```

Verify:
```bash
atx --version
```

### Configure AWS credentials

Use short-lived credentials. In order of preference:

**IAM Identity Center (recommended):**
```bash
aws configure sso        # one-time setup
aws sso login --profile your_profile_name
export AWS_PROFILE=your_profile_name
```

**An assumed role**, via a profile with `role_arn` and `source_profile` in `~/.aws/config`:
```bash
export AWS_PROFILE=your_profile_name
```

**Environment variables** — use temporary credentials from `sts assume-role` or Identity Center, not long-lived IAM user access keys:
```bash
export AWS_ACCESS_KEY_ID=your_access_key
export AWS_SECRET_ACCESS_KEY=your_secret_key
export AWS_SESSION_TOKEN=your_session_token
```

Long-lived IAM user access keys (no session token) work but are discouraged: they do not expire, and the transformation does not need standing access.

**IAM permissions required:** the `AWSTransformCustomFullAccess` managed policy is the quick-start option for a development account. For shared or production accounts, scope permissions down instead — see [AWS managed policies for AWS Transform](https://docs.aws.amazon.com/transform/latest/userguide/security-iam-awsmanpol.html).

### Configure AWS region

AWS Transform is available in: `us-east-1`, `eu-central-1`, `eu-west-2`, `ca-central-1`, `ap-northeast-1`, `ap-northeast-2`, `ap-southeast-2`, `ap-south-1`.

```bash
export AWS_REGION=us-east-1
```

## Usage

### Step 1: Publish the TD to the registry

Clone and publish:

```bash
git clone <this-repo-url>
cd oracle-to-postgresql-td

atx custom def publish \
  -n "oracle-to-postgresql" \
  --description "Oracle to PostgreSQL migration for Java applications" \
  --sd .
```

### Step 2: Run the transformation

Before you start: work on a repository you own, from a clean working tree, and be ready to review the diff. This hands an agent write access to your source.

First run — no `-t`, so you see and approve each tool the agent uses:

```bash
cd /path/to/your/java-project
git status                      # confirm a clean tree

atx custom def exec \
  -p . \
  -n "oracle-to-postgresql"
```

Once you have seen what it does and you trust the flow, add `-t -x` for unattended runs:

```bash
atx custom def exec \
  -p . \
  -n "oracle-to-postgresql" \
  -t -x -d
```

### With build verification (optional)

```bash
atx custom def exec \
  -p . \
  -n "oracle-to-postgresql" \
  -t -x -d \
  -c "./gradlew compileJava"
```

## CLI flags

| Flag | Description |
|------|-------------|
| `-p <path>` | Path to the code repository to transform |
| `-n <name>` | Name of the transformation definition in the registry |
| `-t` | Trust all tools — skips approval prompts. Omit on your first run |
| `-x` | Non-interactive mode |
| `-d` | Do not learn (opt out of knowledge extraction) |
| `-c <command>` | Build command to verify changes |

`-t -x` together means the agent edits your source with no prompts and no interaction. That is fine for a repeat run on a clean branch, and the staging-branch output (see [Output](#output)) gives you a reviewable diff either way — but review that diff before merging. See [SECURITY.md](SECURITY.md) for what to look for.

## Resuming after interruption

If the transformation is interrupted, resume with:

```bash
atx --conversation-id <CONVERSATION_ID> -t
```

The conversation ID is printed at startup and saved in `~/.aws/atx/custom/`.

## Updating a published TD

```bash
atx custom def publish \
  -n "oracle-to-postgresql" \
  --description "Oracle to PostgreSQL migration (updated)" \
  --sd /path/to/this/repo
```

## File structure

```
.
├── README.md                       # This file
├── transformation_definition.md    # Core transformation rules (71 rules)
├── summaries.md                    # Summaries of reference documentation
└── document_references/            # Supporting documentation
    ├── before_after_examples.md    # Before/after code examples
    ├── common_pitfalls.md          # Common migration pitfalls
    └── migration_checklist.md      # Migration verification checklist
```

## Output

AWS Transform creates a staging branch (e.g., `atx-result-staging-YYYYMMDD_HHMMSS_ID`) with committed changes. After completion:

1. Review the changes on the staging branch
2. Check the diff for the four regressions a database migration tends to introduce:
   - a connection URL that lost its TLS settings (`sslmode` missing — the PostgreSQL driver does not encrypt by default)
   - a credential written into source or configuration
   - a bound parameter (`?`, `:named`) that became a literal or a concatenated string
   - tests newly marked `@Ignore`, and exception handlers widened to `catch (Exception)`
3. Squash into a single commit if desired
4. Push and create a pull request

[SECURITY.md](SECURITY.md) covers each of these in more detail.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `AWS Transform not available in region` | Set `AWS_REGION` to a [supported region](#configure-aws-region), or retry later |
| `Transformation not found in registry` | Publish first with `atx custom def publish` |
| `Unsupported files in TD directory` | Only `transformation_definition.md`, `summaries.md`, and `document_references/` are allowed |
| `Credentials expired` | Refresh AWS credentials or session token |
| Network errors / DNS failures | Verify internet access to `transform-custom.<region>.api.aws` |

## Documentation

- [AWS Transform Getting Started](https://docs.aws.amazon.com/transform/latest/userguide/custom-get-started.html)
- [AWS Transform Troubleshooting](https://docs.aws.amazon.com/transform/latest/userguide/custom-troubleshooting.html)
- [AWS Transform IAM Policies](https://docs.aws.amazon.com/transform/latest/userguide/security-iam-awsmanpol.html)
