# `trivy` (Trivy Vulnerability Scan)

Composite action that runs [Trivy](https://github.com/aquasecurity/trivy) against your
repository. It runs the scan twice: once to produce a full SARIF report (uploaded as a
workflow artifact) and once as a **HIGH/CRITICAL severity gate** that fails the job when
findings are present. A `.trivyignore` file at the scan root is honored if it exists.

## Inputs

| Input Name                | Description                                                                                   | Required | Default   |
|---------------------------|-----------------------------------------------------------------------------------------------|----------|-----------|
| `scan_type`               | Trivy scan type: `config` (IaC), `fs` (filesystem/deps), or `repo` (git).                     | `true`   |           |
| `scan_ref`                | Path or git URL to scan.                                                                       | `false`  | `.`       |
| `setup_gradle`            | Set up JDK + Gradle + AWS-OIDC CodeArtifact and generate an ephemeral `gradle.lockfile` before scanning. See [Gradle support](#gradle-support). | `false`  | `"false"` |
| `java_version`            | JDK version used when `setup_gradle` is true.                                                  | `false`  | `"17"`    |
| `gradle_repo_url`         | Maven repo URL written to `gradle.properties`. Required when `setup_gradle` is true.           | `false`  | `""`      |
| `gradle_repo_username`    | Username written to `gradle.properties` (CodeArtifact convention).                             | `false`  | `"aws"`   |
| `aws_role_to_assume`      | AWS IAM role ARN to assume via OIDC. Required when `setup_gradle` is true.                     | `false`  | `""`      |
| `aws_account_id`          | AWS account ID that owns the CodeArtifact domain. Required when `setup_gradle` is true.        | `false`  | `""`      |
| `aws_region`              | AWS region for the CodeArtifact domain. Required when `setup_gradle` is true.                  | `false`  | `""`      |
| `aws_codeartifact_domain` | AWS CodeArtifact domain name. Required when `setup_gradle` is true.                            | `false`  | `""`      |

## Behavior

- **SARIF report** is written to `trivy-results.sarif` and uploaded as the
  `trivy-sarif-<scan_type>` artifact (30-day retention).
- **Severity gate** scans for `CRITICAL,HIGH` and fails the job if any are found. A table
  of findings is written to the job summary and the step log.
- **Skipped dirs**: `**/.terraform` for `config`; plus `**/build,**/.gradle,**/node_modules`
  for `fs`/`repo`.

## Checkout

For `config` and `fs` scans, the caller must check out the repository **before** invoking
this action (`actions/checkout`). For a `repo` scan against a remote git URL, no checkout
is required.

## Gradle support

Trivy's `fs` scan reads transitive dependencies from a `gradle.lockfile`. Gradle repos
usually don't commit one, so set `setup_gradle: true` to have the action set up JDK +
Gradle, authenticate to AWS CodeArtifact via OIDC, and generate an **ephemeral**
`runtimeClasspath`-only lockfile before scanning. Your `build.gradle.kts` is not modified.

This requires the calling workflow to grant `id-token: write`. Because composite actions
cannot read the `secrets` context, the AWS values are passed as inputs. For a turn-key
version that maps secrets for you, use the
[reusable workflow](../../README.md#reusable-workflow) instead.

## Example Usage

Ready-to-copy caller workflows live in [`examples/`](../../examples):

- [`trivy-config.yml`](../../examples/trivy-config.yml) — `config` (IaC) scan.
- [`trivy-fs.yml`](../../examples/trivy-fs.yml) — `fs` (filesystem/deps) scan.
- [`trivy-gradle.yml`](../../examples/trivy-gradle.yml) — Gradle `fs` scan via the
  [reusable workflow](../../README.md#reusable-workflow) (the turn-key path for
  `setup_gradle`).

To run a Gradle scan with the composite action directly instead of the reusable
workflow, set `setup_gradle: true` and pass the `aws_*` / `gradle_repo_url` inputs from
the table above, with `id-token: write` granted at the workflow level.
