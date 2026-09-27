# samvaad-e2ee-lib

A reusable JVM library providing Samvaad's client-side end-to-end encryption
(E2EE) implementation.

This repository was extracted from
`animesh0404/samvaad-server` (`e2ee-client/` build, extraction commit
`a50ba99`). Git history for the client implementation is preserved in this
repository; server-side transport, persistence, and API code remains in
`samvaad-server`.

## What it is

`samvaad-e2ee-lib` owns the client side of Samvaad V1 one-to-one messaging
crypto: per-device Signal/Sesame sessions, lazy session establishment,
multi-device fan-out, trust evaluation, and durable custody of all
client-side cryptographic state. It is a plain JVM `java-library` with no
framework dependencies.

Gradle coordinates (unchanged from extraction):

```text
group    = com.samvaad
artifact = e2ee-client
version  = 0.1.0
```

```groovy
implementation 'com.samvaad:e2ee-client:0.1.0'
```

## Architecture

Crypto operations flow through a Samvaad-owned seam, so only one class ever
touches the Signal library:

```text
SamvaadCryptoService
        ↓
SignalAdapter          (Samvaad-owned interface; no library types escape it)
        ↓
LibSignalAdapter       (the only class referencing org.signal.libsignal.*)
        ↓
libsignal-client 0.86.5
```

- `SamvaadCryptoService` orchestrates lazy per-device session establishment
  and multi-device fan-out over caller-supplied transport seams
  (`ClaimClient`, `SubmitClient`). The server transport is never a
  dependency; tests supply fakes.
- `SignalAdapter` exposes opaque public bytes and sealed private handles
  (`SealedPrivateHandle`). Raw private key bytes never cross this boundary
  in either direction.
- Wire classification is explicit: the sender sets
  `CryptoTypes.EnvelopeType` (`PREKEY_INIT` / `RATCHET`) and the recipient
  selects the parser from it — never by trial-parsing.

Persistence is owned by the same library through a unified store boundary:

```text
ClientCryptoStore
├── SessionStore
├── DeviceKeyStore
├── TrustStore
└── PrekeyManager (single-concern interface, composed by callers)

        ↓  reference JVM implementations

FileBackedClientCryptoStore   (atomic file snapshot: tmp-write + fsync +
                               atomic move + directory fsync)
FilePrivateKeyVault           (password-based file backend for sealed
                               private handles; implements PrivateKeyVault)
SnapshotJson                  (package-private, JDK-only snapshot codec)
IdentityFingerprints          (JDK-only display fingerprints for
                               out-of-band verification UI)
```

Two transitions are crash-atomic by contract:

1. Outbound encrypt commit — advanced session blob + `COMMITTED` ciphertext
   slot become durable together, or neither does
   (`SessionStore.commitOutboundCiphertext`).
2. Inbound establishment commit — inbound `READY` session + one-time-prekey
   consumption become durable together, or neither does
   (`ClientCryptoStore.commitInboundEstablishment`).

Stores persist sealed private material only as opaque handle references
(`SealedPrivateHandle.handleId()` UUIDs), never as key bytes. The vault owns
the material; the store owns the references and lifecycle flags.
Implementations assume single-process ownership.

## Important properties

- Runs on **Java 25** (`toolchain` + `options.release = 25`).
- Uses **`org.signal:libsignal-client:0.86.5`** as its single external
  dependency (`implementation`, never `api` — no public type exposes
  libsignal). Pinned because later 0.10x lines are not published to Maven
  Central.
- Contains **no Spring** dependency.
- Contains **no Jackson** dependency (snapshot codec and fingerprints are
  JDK-only).
- Contains **no server** dependency — no DTOs, entities, repositories,
  controllers, or configuration from `samvaad-server`.
- Owns client-side crypto persistence formats (`STORE_FORMAT_VERSION = 1`;
  a bump requires a migration path) and all client-side cryptographic state.
- Is consumed by Samvaad JVM applications through the coordinates above
  (local development may resolve them via a Gradle composite build; no
  published artifact is required for that).
- Test doubles (`fake/FakeSignalAdapter`, `fake/InMemoryStores`) live under
  `src/test` and are not shipped in the library jar.

See `docs/adr/README.md` for the architectural decisions behind this
boundary.

## License

Copyright (C) 2026 Animesh Kahara.

This library is free software: you can redistribute it and/or modify it
under the terms of the GNU Affero General Public License as published by
the Free Software Foundation, version 3 only (no "or later" provision).
See the root `LICENSE` file for the full text. Each source file carries
an `SPDX-License-Identifier: AGPL-3.0-only` header.

## Third-party dependencies

- `org.signal:libsignal-client:0.86.5` — Copyright Signal Messenger, LLC,
  licensed under AGPL-3.0-only
  (`https://github.com/signalapp/libsignal`). The library links it
  behind the `SignalAdapter` seam; no Signal source is vendored in this
  repository and Signal's own copyright/license notices ship untouched
  inside the upstream artifact.
- `org.jetbrains.kotlin:kotlin-stdlib`,
  `org.jetbrains.kotlinx:kotlinx-coroutines-core`,
  `org.jetbrains:annotations` (transitive via libsignal) — Apache
  License 2.0.
