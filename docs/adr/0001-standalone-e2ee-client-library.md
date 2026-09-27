# ADR 0001: Standalone reusable JVM E2EE client library

## Status

Accepted. Implementation: this repository — single-module Gradle
`java-library` (`com.samvaad:e2ee-client:0.1.0`) rooted at
`com.samvaad.e2ee.client` with `.signal` and `.persist` subpackages.

Extracted from `animesh0404/samvaad-server` (`e2ee-client/` build, HEAD
`a50ba99`), preserving the seven-commit client history (wire contracts,
crypto-boundary hardening, persistent store boundary, JVM Signal adapter,
key custody, and the extraction itself).

## Context

The JVM E2EE client implementation (`SamvaadCryptoService`,
`SignalAdapter`, `LibSignalAdapter`, `ClientCryptoStore`,
`FileBackedClientCryptoStore`, `PrivateKeyVault`, `FilePrivateKeyVault`,
plus supporting client types) originally lived inside the server
repository — first under
`com.samvaad.samvaad_server.e2ee.client`, then in the in-repo
`e2ee-client/` build (server ADR 0024). Client applications (notably
`samvaad-tui`) must reuse exactly this implementation over the boundary
`SamvaadCryptoService` → `SignalAdapter` → `LibSignalAdapter` →
libsignal, without reimplementing Signal/PQXDH/Kyber and without depending
on the Spring Boot server application.

The protocol direction comes from the server repository and is not
re-decided here:

- Server ADR 0018 (locked): V1 one-to-one messaging uses the Signal
  protocol family with Sesame-style multi-device session management; the
  server transport trust domain must never hold client private handles or
  session state.
- Server ADR 0021 (locked): trust compares canonical identity public-key
  bytes; fingerprints are display-only for out-of-band verification UI.
- Server ADR 0022 (locked): the two crash-atomic durability boundaries
  (outbound encrypt commit; inbound establishment commit) and the rule that
  the server PostgreSQL schema is not reused for client state.
- Server ADR 0023 (locked): `LibSignalAdapter` behind the unchanged
  `SignalAdapter` seam, mandatory last-resort Kyber material in every
  session bundle, and password-based file private-key custody for the JVM
  reference backend.
- Server ADR 0024 (accepted): the mechanical extraction that created the
  `e2ee-client` build consumed via composite builds.

## Decision

1. This repository is the home of the reusable client library: plain JVM
   `java-library` on Java 25, single external dependency
   `org.signal:libsignal-client:0.86.5` (`implementation`, never `api` —
   no public type exposes libsignal). No Spring, no Jackson, no JPA, no
   server DTOs/entities/repositories. `SnapshotJson` stays package-private
   and JDK-only, as do the vault and fingerprint implementations.
2. Package names (`com.samvaad.e2ee.client`, `.signal`, `.persist`) and
   Gradle coordinates (`com.samvaad:e2ee-client:0.1.0`) are frozen at
   their extracted values. The repository name (`samvaad-e2ee-lib`) and
   the artifact name intentionally differ.
3. Test doubles (`fake/FakeSignalAdapter`, `fake/InMemoryStores`) do not
   ship in the library jar; they live under `src/test` for the library's
   own proofs. Downstream consumers needing fakes maintain their own.
4. No protocol, persistence-format, wire-format, fingerprint, trust-state,
   or key-custody behavior changes are part of extraction: file contents
   at extraction are byte-identical to `samvaad-server` HEAD `a50ba99`
   (`e2ee-client/` subtree), except the repository-level files listed
   under Consequences.
5. The library owns its persistence formats (`STORE_FORMAT_VERSION = 1`;
   a bump requires a migration path) and all client-side cryptographic
   state. Server-side enrollment, device directory, mailbox transport, and
   recovery-code storage remain `samvaad-server` responsibilities and are
   not duplicated here.

## Consequences

- Repository-level files new (or changed) relative to the extracted
  subtree: `settings.gradle` root name (`samvaad-e2ee-lib`; artifact
  coordinates unchanged), `.gitignore`, `.gitattributes`, `README.md`,
  and this `docs/adr/` set. No `LICENSE` file exists yet — choosing one
  is required before publishing the repository or any artifact.
- Consumers resolve `com.samvaad:e2ee-client:0.1.0` either from a
  published artifact (deferred) or via a Gradle composite build pointing
  at a local checkout of this repository. Versioning of this coordinate
  is the shared contract; publishing infrastructure is a separate,
  explicitly deferred decision.
- Cryptographic decisions (protocol, Kyber mandate, trust, custody,
  store boundaries) stay locked in the server ADRs cited above; this ADR
  records only the repository/library boundary. Any change to locked
  behavior must go through the owning decision record first.
