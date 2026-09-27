# ADR 0002: Publish to GitHub Packages instead of Maven Central

## Status

Accepted. Implementation: Gradle `maven-publish` publication `e2eeClient`
(`com.samvaad:e2ee-client:0.1.0`) targeting the GitHub Packages Maven
repository `https://maven.pkg.github.com/animesh0404/samvaad-e2ee-lib`,
published by the release-triggered workflow
`.github/workflows/publish.yml`. No version `0.1.0` artifact has been
published yet; the first publication happens on the first explicit GitHub
Release.

## Context

Until now the library had no Maven repository at all: consumers
(`samvaad-server`, `samvaad-tui`) resolve `com.samvaad:e2ee-client:0.1.0`
through a Gradle composite build against a local checkout of this
repository (ADR 0001; server ADR 0024, which defers publishing). A shared
binary artifact is needed so consumers no longer require a sibling
checkout. Maven Central was checked and `com.samvaad:e2ee-client:0.1.0`
is not there; no publishing infrastructure existed (no `maven-publish`
configuration, no workflow, no credentials).

## Decision

1. Use GitHub Packages as the Maven repository for now, at
   `https://maven.pkg.github.com/animesh0404/samvaad-e2ee-lib`, instead
   of Maven Central. Maven Central (namespace claim, artifact signing,
   staging workflow) is deferred, not rejected; it remains the option if
   public unauthenticated resolution is ever required.
2. Publish the frozen coordinate `com.samvaad:e2ee-client:0.1.0` via a
   `maven-publish` publication named `e2eeClient` built
   `from components.java`, so the generated POM carries the library's
   runtime dependencies. The `artifactId` is pinned explicitly to
   `e2ee-client` because the Gradle project name (`samvaad-e2ee-lib`)
   intentionally differs from the Maven artifact.
3. No Java code, API, dependency-version, or license changes are part of
   this slice: configuration, workflow, and documentation only.
4. Credentials come only from the environment (`GITHUB_ACTOR` /
   `GITHUB_TOKEN`); no token is hardcoded. CI publishes with the
   repository-provided `GITHUB_TOKEN` (`contents: read`,
   `packages: write`).
5. Publication is release-driven: the workflow runs on an explicit
   GitHub Release (`release: types: [published]`) and runs the full
   `./gradlew clean test` suite before publishing. Nothing is published
   on push, locally, or automatically.
6. The Gradle composite build remains the resolution mechanism for local
   `samvaad-server` / `samvaad-tui` checkouts until their migration to
   the published package is completed as a separate step. Both paths
   share the same frozen coordinate, so migration is a
   repository-resolution change, not a code change.

## Consequences

- Consumers must declare the GitHub Packages repository and authenticate
  (GitHub Packages Maven requires authentication even for reads) with
  `GITHUB_ACTOR` / `GITHUB_TOKEN`, as documented in `README.md`.
- The first `0.1.0` artifact becomes available only after the first
  GitHub Release triggers the workflow; until then the composite build
  is still the only working resolution path.
- A future move to Maven Central requires a new decision record but no
  coordinate change: group, artifactId, and version stay frozen.
