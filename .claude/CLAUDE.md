# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Architecture Overview

This repository provides a reusable composite GitHub Action — and a thin reusable
workflow wrapper around it — for running Trivy vulnerability and misconfiguration scans.
The action is consumed via `uses: c0x12c/gh-actions-trivy-scan/actions/trivy@v1.0.0`; the
workflow via `uses: c0x12c/gh-actions-trivy-scan/.github/workflows/trivy.yml@v1`.

### Core Components

**Composite Action:**
- `actions/trivy/` — Trivy scan action (`config`/`fs`/`repo` scan types). Runs a full
  SARIF scan (uploaded as an artifact) plus a HIGH/CRITICAL severity gate that fails the job.
  Also holds the optional Gradle pre-scan setup (`setup_gradle: true` → JDK + Gradle +
  ephemeral `runtimeClasspath` lockfile; **CodeArtifact auth is opt-in**, keyed on
  `gradle_repo_url` being set). `gradle_working_directory` supports monorepos where
  `gradlew` is in a subdir. **All scan/Gradle logic lives here — single source of truth.**

**Reusable Workflow:**
- `.github/workflows/trivy.yml` — `workflow_call` wrapper. Does `checkout`, maps generic
  inputs/secrets, then delegates the scan to `actions/trivy@v1.0.0`. It contains no scan logic
  of its own. Consumed via `uses: c0x12c/gh-actions-trivy-scan/.github/workflows/trivy.yml@v1`.

**Examples:**
- `examples/` — ready-to-copy caller workflows. The READMEs reference these files rather
  than inlining YAML, so an example is the single place a usage snippet is maintained.

**Release Management:**
- `tools/semtag` — Semantic versioning tool for automated releases
- `tools/create_release.sh` — Script for creating releases with proper version tagging
- `.github/workflows/` — Automated workflows for version management and releases

### Action Pattern

The Trivy action:
1. Validates `scan_type` (`config`, `fs`, or `repo`)
2. Ensures a `.trivyignore` file exists
3. Runs a full SARIF scan and uploads `trivy-results.sarif` as an artifact
4. Runs a HIGH/CRITICAL severity gate, publishing findings to the job summary
5. Fails the job if HIGH/CRITICAL findings are present

The caller is responsible for `actions/checkout` (for `config`/`fs` scans) and for any
workflow-level `permissions`.

## Common Development Tasks

### Release Management

**Create a new release:**
```bash
./tools/create_release.sh patch   # or: minor | major
```

The release process updates version tags and creates GitHub releases, updating both the
specific version tag (v1.2.3) and the moving major/minor tags (v1, v1.2).

### Action Development

When modifying the action:
1. Update `actions/trivy/action.yml` (the single source of truth for scan/Gradle logic).
2. If the change affects the reusable-workflow surface (a new input/secret to pass
   through), update `.github/workflows/trivy.yml` too — but never duplicate scan logic
   there; it only checks out, maps inputs/secrets, and delegates to `actions/trivy@v1.0.0`.
3. Update or add an `examples/` workflow. Keep usage snippets in `examples/` and
   **reference them** from the READMEs instead of pasting YAML into the docs.
4. Update `actions/trivy/README.md` and the root `README.md`.
5. Update `CHANGELOG.md`.
6. Create a PR following the existing template.
7. After merge, create a new release to make changes available.

**Contributor gotchas (for other agents):**
- Composite actions **cannot read the `secrets` context** — AWS values reach the action
  as plain inputs; the reusable workflow is what maps `secrets.*` into those inputs.
- Composite-action boolean inputs are **strings**: gate with `if: inputs.x == 'true'`.
- Keep the action/workflow **vendor-neutral** — no caller-specific secret/var names
  (e.g. no `*_DEV`) or project names baked in. Pass concrete values in.
- The reusable workflow delegates to `actions/trivy@v1.0.0`, so the `v1` tag must exist when
  it runs. A workflow tested on a feature branch runs the *released* action, not the
  branch copy — verify action changes via a direct `uses: ./actions/trivy` job or after
  release.

### Version Strategy

Semantic versioning with moving tags: `v1.2.3` (specific), `v1.2` (latest patch in minor),
`v1` (latest minor in major). Consumers typically reference `@v1`.
