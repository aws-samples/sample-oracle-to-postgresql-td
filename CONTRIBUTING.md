# Contributing Guidelines

Thank you for your interest in contributing to this project. Whether it's a bug report, new
transformation rule, correction, or additional documentation, we greatly value feedback and
contributions from the community.

Please read through this document before submitting any issues or pull requests to ensure we
have all the necessary information to effectively respond to your contribution.

## What this repository contains

This repository is a **transformation definition (TD)** for [AWS Transform](https://docs.aws.amazon.com/transform/latest/userguide/custom-get-started.html).
It contains Markdown instruction files only — no application source code and no deployable
infrastructure. The content is consumed by an AI agent that rewrites Java and Kotlin source
code in a target repository, so the accuracy and safety of the guidance matters directly.

| File | Purpose |
|---|---|
| `transformation_definition.md` | Core transformation rules |
| `summaries.md` | Consolidated learnings and migration statistics |
| `document_references/before_after_examples.md` | Before/after code examples |
| `document_references/common_pitfalls.md` | Anti-patterns to avoid |
| `document_references/migration_checklist.md` | Per-file verification checklist |

AWS Transform only accepts `transformation_definition.md`, `summaries.md`, and the
`document_references/` directory. Do not add other files to the TD payload.

## Reporting bugs and feature requests

Before filing an issue, please search existing open and recently closed issues to confirm it
has not already been reported. When filing, include as much detail as you can:

- A reproducible test case: the Oracle input, the PostgreSQL output you received, and the
  output you expected
- The version of the TD you are using and the AWS Transform CLI version (`atx --version`)
- The technology stack involved (ORM, framework, build tool, PostgreSQL version, whether
  orafce is installed)
- Anything unusual about your environment

Do not include customer names, real account identifiers, credentials, or production data in
issues. Reduce examples to a minimal synthetic case first.

## Contributing via pull requests

Before sending a pull request, please:

1. Check that you are working against the latest source on the `main` branch
2. Search open and recently merged pull requests to make sure the change is not already in flight
3. Open an issue to discuss any significant rule change, so effort is not wasted

To send a pull request:

1. Fork the repository
2. Modify the source, focusing on the specific change you are contributing
3. Commit to your fork using a clear commit message
4. Send the pull request, answering any default questions in the template
5. Pay attention to any automated CI failures and stay involved in the review conversation

### Guidelines for new or changed rules

- **Rule numbering**: rules in `transformation_definition.md` are referenced by number.
  Append new rules rather than renumbering existing ones.
- **Keep the four files consistent**: a new rule usually needs a matching before/after example,
  a pitfall entry if there is a known wrong approach, and a checklist line item.
- **Every rule needs a real before and after**: show the Oracle input and the PostgreSQL output.
  State the actual error message the pitfall produces where you know it.
- **Never introduce a rule that changes business logic.** Syntax and type handling only. This is
  the first CRITICAL rule in the TD and it is not negotiable.
- **Preserve parameter binding.** Do not add guidance that inlines a bound parameter into SQL
  text or builds SQL by concatenating request-derived values.
- **Do not weaken security posture during migration.** Guidance that touches connection
  configuration must keep encryption in transit enabled and keep credentials out of source
  files and configuration files.
- **Genericise all examples.** Table, column, index, sequence, and file names in examples must
  be invented or neutral. Do not contribute identifiers copied from a customer codebase.

## Finding contributions to work on

Looking at existing issues is a great way to find something to work on, particularly anything
labelled `help wanted` or `good first issue`.

## Code of Conduct

This project has adopted the [Amazon Open Source Code of Conduct](https://aws.github.io/code-of-conduct).
See [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) for details.

## Security issue notifications

If you discover a potential security issue in this project, please follow the process in
[SECURITY.md](SECURITY.md). Do **not** open a public GitHub issue for security reports.

## Licensing

This project is licensed under the MIT-0 License. See [LICENSE](LICENSE). We may ask you to
confirm the licensing of your contribution.
