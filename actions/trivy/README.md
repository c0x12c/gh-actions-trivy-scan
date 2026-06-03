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
| `skip_files`              | Comma-separated paths/globs to skip (Trivy `--skip-files`). For files Trivy can't parse, e.g. encrypted Terraform secrets (`terraform/**/secrets.tf`). | `false`  | `""`      |
| `setup_gradle`            | Set up JDK + Gradle and generate an ephemeral `gradle.lockfile` before scanning. See [Gradle support](#gradle-support). | `false`  | `"false"` |
| `java_version`            | JDK version used when `setup_gradle` is true.                                                  | `false`  | `"17"`    |
| `gradle_working_directory`| Directory containing `gradlew` (for monorepos where Gradle lives in a subdir, e.g. `service`). | `false`  | `"."`     |
| `gradle_repo_url`         | Private CodeArtifact Maven repo URL. **Set this to enable CodeArtifact auth**; leave empty for public Maven. | `false`  | `""`      |
| `gradle_repo_username`    | Username written to `gradle.properties` (CodeArtifact convention). Only used when `gradle_repo_url` is set. | `false`  | `"aws"`   |
| `aws_role_to_assume`      | AWS IAM role ARN to assume via OIDC. Required only when `gradle_repo_url` is set.              | `false`  | `""`      |
| `aws_account_id`          | AWS account ID that owns the CodeArtifact domain. Required only when `gradle_repo_url` is set.  | `false`  | `""`      |
| `aws_region`              | AWS region for the CodeArtifact domain. Required only when `gradle_repo_url` is set.            | `false`  | `""`      |
| `aws_codeartifact_domain` | AWS CodeArtifact domain name. Required only when `gradle_repo_url` is set.                      | `false`  | `""`      |

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
Gradle and generate an **ephemeral** `runtimeClasspath`-only lockfile before scanning.
Your `build.gradle.kts` is not modified.

- **Public Maven** (Maven Central, Google Maven, etc.): just `setup_gradle: true`. No AWS
  inputs, no `id-token: write` needed. Set `gradle_working_directory` if `gradlew` is in a
  subdirectory (e.g. `service`).
- **Private CodeArtifact**: also set `gradle_repo_url` (+ `aws_role_to_assume`,
  `aws_account_id`, `aws_region`, `aws_codeartifact_domain`). This enables OIDC CodeArtifact
  auth and requires the workflow to grant `id-token: write`. Because composite actions
  cannot read the `secrets` context, the AWS values are passed as inputs — for a turn-key
  version that maps secrets for you, use the
  [reusable workflow](../../README.md#reusable-workflow).

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
