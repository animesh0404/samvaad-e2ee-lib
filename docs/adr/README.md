# Architecture Decision Records

This directory records durable decisions for `samvaad-e2ee-lib` as an
independent repository.

Decisions originating in `animesh0404/samvaad-server` are attributed to
their server ADR numbers. The server ADRs themselves remain the authority
for server-side transport, enrollment, and device-identity behavior; only
the client-library boundary portions are restated here so this repository
is understandable on its own.

## Index

| ADR | Decision |
| --- | --- |
| [0001](0001-standalone-e2ee-client-library.md) | Standalone reusable JVM E2EE client library |
| [0002](0002-github-packages-publication.md) | Publish to GitHub Packages instead of Maven Central |
