# ci-workflows

Reusable [GitHub Actions](https://docs.github.com/actions/using-workflows/reusing-workflows)
workflows shared across `eu.virtualparadox` projects.

## `maven-central-release.yml` — tag-driven Maven Central release

The **git tag is the single source of truth** for the released version. Whatever the source
`pom.xml` says is irrelevant — the workflow sets the version from the tag for the build only and
never writes it back, so release history stays clean. Consuming projects keep their own
`<version>` at `0-SNAPSHOT` as a self-documenting marker ("the version comes from the tag, not by
hand").

| Tag | Published to Maven Central |
|-----|----------------------------|
| `v1.0.1-RC1` | `1.0.1-RC1` (a real, immutable pre-release) |
| `v1.0.1` | `1.0.1` (final release) |

Both run the **full quality pipeline before anything ships**: every static-analysis gate +
tests + (optional) OWASP CVE scan + (optional) CycloneDX SBOM, then GPG signing and the Central
upload. If anything fails — a test, a gate, a newly-published CVE — nothing is published.

### Usage

Add this to a consuming repo as `.github/workflows/release.yml`:

```yaml
name: Release
on:
  push:
    tags: ['v*']
jobs:
  release:
    uses: virtualparadoxorg/ci-workflows/.github/workflows/maven-central-release.yml@v1
    with:
      # optional, per-repo extras:
      maven-args: "-DexcludeArtifacts=my-build-tool,my-it-module"
    secrets: inherit
```

Then release by tagging:

```bash
git tag v1.0.1-RC1 && git push origin v1.0.1-RC1   # -> 1.0.1-RC1 on Central
git tag v1.0.1     && git push origin v1.0.1        # -> 1.0.1 on Central
```

### Inputs

| Input | Default | Purpose |
|-------|---------|---------|
| `java-version` | `21` | JDK to build/release with. |
| `maven-args` | `""` | Extra Maven args appended to the deploy (e.g. `-DexcludeArtifacts=...`). |
| `version-properties` | `""` | Comma-separated pom property names also set to the released version (e.g. a literal build-config version a parent pins), so the source can stay `0-SNAPSHOT` with no manual bumps. |
| `run-security` | `false` | Run the OWASP dependency-check (`security` profile). |
| `run-sbom` | `true` | Generate the CycloneDX SBOM (`sbom` profile). |

### Required secrets (pass with `secrets: inherit`)

`CENTRAL_TOKEN_ID`, `CENTRAL_TOKEN_PASSWORD`, `GPG_PRIVATE_KEY`, `GPG_PASSPHRASE`
(and optionally `NVD_API_KEY` for a fast OWASP scan). These are set at the `virtualparadoxorg`
org level (visibility: all repositories), so consumers only need `secrets: inherit`.

### What it expects of the project

- A Maven build runnable via the `./mvnw` wrapper at the repo root.
- A `release` profile that signs (maven-gpg) and publishes (Sonatype central-publishing) — provided
  by the `eu.virtualparadox:parent` POM.
- `security` / `sbom` profiles (also from the parent) if `run-security` / `run-sbom` are enabled.

### Versioning of this workflow

Consumers pin a major tag: `@v1`. The `v1` tag is moved forward to the latest `v1.x` release.
