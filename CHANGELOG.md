# Change Log
All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](http://keepachangelog.com/)
and this project adheres to [Semantic Versioning](http://semver.org/).

## [Unreleased]

### Added

- Initial release: `trivy` composite action for running Trivy `config`/`fs`/`repo` scans,
  uploading a SARIF report artifact, and failing the job on HIGH/CRITICAL findings.
- Optional Gradle pre-scan setup on the composite action (`setup_gradle: true`): sets up
  JDK + Gradle + AWS-OIDC CodeArtifact auth and generates an ephemeral
  `runtimeClasspath`-only `gradle.lockfile` so Trivy `fs` can read transitive deps. Adds
  the `setup_gradle`, `java_version`, `gradle_repo_url`, `gradle_repo_username`,
  `aws_role_to_assume`, `aws_account_id`, `aws_region`, and `aws_codeartifact_domain` inputs.
- Reusable workflow `.github/workflows/trivy.yml` (`workflow_call`) that handles checkout
  and secret/input mapping, then delegates the scan to `actions/trivy@v1`.
- `examples/` directory with ready-to-copy caller workflows (`trivy-config.yml`,
  `trivy-fs.yml`, `trivy-gradle.yml`).
