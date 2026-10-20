# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Spring Boot 3.3 / Java 17 demo backend for offline UPI payments relayed over a simulated Bluetooth mesh. A sender's phone encrypts a payment, phones gossip the opaque packet between them, and any "bridge" phone that gets internet uploads it to `/api/bridge/ingest`. The server then decrypts it, drops duplicates and settles it. The mesh itself is simulated in-process, so the whole flow runs on one machine. `README.md` has the full design rationale, API table and production-vs-demo notes.

## Commands

Use the Maven wrapper (no Maven install needed; JDK 17+ required). On Windows, use `mvnw.cmd`.

```bash
./mvnw spring-boot:run          # run app; dashboard at http://localhost:8080
./mvnw test                     # all tests
./mvnw test -Dtest=IdempotencyConcurrencyTest                                                     # one class
./mvnw test -Dtest=IdempotencyConcurrencyTest#singlePacketDeliveredByThreeBridgesSettlesExactlyOnce  # one method
./mvnw package                  # build jar
```

The project has no linter or formatter configured.

Branches: work on `dev` and merge into `main` to release. `main` is what Render deploys, so don't push to it directly.

Deployment: `Dockerfile` (multi-stage Maven build, then JRE image) and `render.yaml` (Render free plan; auto-deploys every push to `main`). The app reads `PORT` (default 8080) and `H2_CONSOLE_ENABLED` (default `true`; set to `false` on Render). The H2 console is at `/h2-console` (JDBC URL `jdbc:h2:mem:upimesh`, user `sa`, empty password).

## Architecture

Packages are under `com.demo.upimesh`. The core is the ingestion pipeline in `service/BridgeIngestionService.ingest()`. Its steps must happen in this order:

1. `HybridCryptoService.hashCiphertext`: the SHA-256 hash of the **ciphertext** is the dedup key. Do not use `packetId`, because intermediaries can rewrite it. Do not use the plaintext, because dedup must happen before the expensive RSA decrypt.
2. `IdempotencyService.claim`: an atomic `ConcurrentHashMap.putIfAbsent`, standing in for Redis `SET NX`. The first claimer proceeds and later ones return `DUPLICATE_DROPPED`. A hash stays claimed even if a later step fails, so a packet that failed once is never retried. A scheduled task evicts entries after `upi.mesh.idempotency-ttl-seconds`; `AppConfig` enables `@EnableScheduling`.
3. `HybridCryptoService.decrypt`: RSA-OAEP unwraps a per-packet AES-256 key, then AES-GCM decrypts and authenticates. The wire layout is `[256B RSA-wrapped key][12B IV][ciphertext+16B tag]`. Tampering throws, which returns `INVALID`.
4. Freshness check on the encrypted `signedAt` against `upi.mesh.packet-max-age-seconds`, with 300 s of future clock-skew tolerance.
5. `SettlementService.settle`: one `@Transactional` debit, credit and ledger insert. If the balance is too low, it writes a `REJECTED` ledger row instead of throwing. As defense in depth, `Account` has `@Version` optimistic locking and `Transaction.packetHash` has a unique index.

The outcome is an `IngestResult` record with `SETTLED`, `DUPLICATE_DROPPED` or `INVALID`. Exceptions are caught and mapped to `INVALID`; they do not propagate.

Supporting pieces:
- `crypto/ServerKeyHolder`: generates a new RSA-2048 keypair at every startup, so packets don't survive a restart.
- `service/DemoService`: seeds accounts (such as `alice@demo` and `bob@demo`) and plays the sender phone through `createPacket(...)`, which builds a `PaymentInstruction` with a nonce and `signedAt`, encrypts it and wraps it in a `MeshPacket`.
- `PaymentInstruction.pinHash` is the SHA-256 of the PIN. It travels inside the encrypted payload but the server never checks it.
- `service/MeshSimulatorService` and `VirtualDevice`: the in-memory mesh. The default devices are four offline phones plus `phone-bridge`, the only one with `hasInternet=true`. `gossipOnce()` lets every device share with every other device using a start-of-round snapshot and decrements the TTL per hop. `collectBridgeUploads()` lists the packets that bridges will upload.
- `controller/ApiController`: holds all REST endpoints. The demo endpoints are `/api/demo/send`, `/api/mesh/gossip`, `/api/mesh/flush` and `/api/mesh/reset`. `/api/mesh/flush` uploads bridge packets in parallel and computes the hop count as `5 - ttl`, so it is only correct when `/api/demo/send` used its default TTL of 5. `/api/bridge/ingest` is the "real" endpoint and reads the headers `X-Bridge-Node-Id` and `X-Hop-Count`. `DashboardController` serves `templates/dashboard.html` (Thymeleaf), the UI for the demo.

Persistence is H2 in-memory with `ddl-auto=create-drop`, so all state resets on restart.

## Tests

`IdempotencyConcurrencyTest` is a `@SpringBootTest`. It covers the encrypt/decrypt round trip, rejection of tampered ciphertext, and three threads ingesting the same packet at once, where exactly one must settle. Any change to the ingestion, idempotency or settlement path must keep that concurrency test passing.

All test methods share one Spring context and in-memory database. `@BeforeEach` only clears the idempotency cache, so account balances carry over between tests. New tests should assert on balance changes, not absolute balances.
