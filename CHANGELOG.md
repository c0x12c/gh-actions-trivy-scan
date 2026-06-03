# Change Log
All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](http://keepachangelog.com/)
and this project adheres to [Semantic Versioning](http://semver.org/).

## [Unreleased]

## [1.1.0]

### Added

- `gradle_working_directory` input — generate the lockfile and scan from a subdirectory,
  for monorepos where the Gradle project isn't at the repo root (e.g. `service`).
- `skip_files` input — comma-separated paths/globs passed to Trivy `--skip-files`. Use to
  skip files Trivy can't parse, e.g. git-filter/SOPS-encrypted Terraform secrets
  (`terraform/**/secrets.tf`), which otherwise log a non-fatal `terraform parser` error.

### Changed

- **CodeArtifact auth is now opt-in.** `setup_gradle: true` alone sets up JDK + Gradle and
  generates the lockfile against public Maven (no AWS, no `id-token: write`). CodeArtifact
  auth + OIDC only run when `gradle_repo_url` is set; the `aws_*` inputs are required only
  in that case. Previously all of them were mandatory whenever `setup_gradle` was true.

## [1.0.0]

### Added

- Initial release: `trivy` composite action for running Trivy `config`/`fs`/`repo` scans,
  uploading a SARIF report artifact, and failing the job on HIGH/CRITICAL findings.
- Optional Gradle pre-scan setup on the composite action (`setup_gradle: true`): sets up
  JDK + Gradle + AWS-OIDC CodeArtifact auth and generates an ephemeral
  `runtimeClasspath`-only `gradle.lockfile` so Trivy `fs` can read transitive deps. Adds
  the `setup_gradle`, `java_version`, `gradle_repo_url`, `gradle_repo_username`,
  `aws_role_to_assume`, `aws_account_id`, `aws_region`, and `aws_codeartifact_domain` inputs.
- Reusable workflow `.github/workflows/trivy.yml` (`workflow_call`) that handles checkout
  and secret/input mapping, then delegates the scan to `actions/trivy@v1.0.0`.
- `examples/` directory with ready-to-copy caller workflows (`trivy-config.yml`,
  `trivy-fs.yml`, `trivy-gradle.yml`).
