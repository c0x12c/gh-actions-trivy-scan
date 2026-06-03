# gh-actions-trivy-scan

This repository provides a reusable composite GitHub Action for running
[Trivy](https://github.com/aquasecurity/trivy) vulnerability and misconfiguration scans.
The action runs a full SARIF scan (uploaded as an artifact) and a HIGH/CRITICAL severity
gate that fails the job when findings are present.

## Available Composite Actions

### `trivy` (Trivy Vulnerability Scan)

Runs a Trivy scan of type `config` (IaC), `fs` (filesystem/deps), or `repo` (git repo).
See [`actions/trivy/README.md`](actions/trivy/README.md) for full documentation.

#### Inputs

| Input Name  | Description                                                               | Required | Default |
|-------------|---------------------------------------------------------------------------|----------|---------|
| `scan_type` | Trivy scan type: `config` (IaC), `fs` (filesystem/deps), or `repo` (git). | `true`   |         |
| `scan_ref`  | Path or git URL to scan.                                                  | `false`  | `.`     |

#### Example Usage

See [`examples/trivy-config.yml`](examples/trivy-config.yml) (IaC scan) and
[`examples/trivy-fs.yml`](examples/trivy-fs.yml) (filesystem/deps scan) for ready-to-copy
caller workflows.

## Reusable Workflow

For Gradle repos — or any caller that wants checkout, OIDC, and secret mapping handled
for it — call the reusable workflow instead of the composite action. It checks out the
repo, optionally sets up JDK + Gradle + AWS-OIDC CodeArtifact and generates an ephemeral
`runtimeClasspath` lockfile (`setup_gradle: true`), then delegates the scan to the
composite action.

Consumed via `uses: c0x12c/gh-actions-trivy-scan/.github/workflows/trivy.yml@v1`. See
[`examples/trivy-gradle.yml`](examples/trivy-gradle.yml) for a complete Gradle example
(it needs `id-token: write` and the `aws_role_to_assume` / `aws_account_id` secrets). For
a non-Gradle scan, omit `setup_gradle` and the `aws_*` inputs/secrets; only
`contents: read` is required.

## Version Strategy

This repository uses semantic versioning with moving tags:

- `v1.2.3` — specific version
- `v1.2` — latest patch in a minor version
- `v1` — latest minor in a major version

Consumers typically reference the major version tag (e.g. `@v1`) for automatic updates.
