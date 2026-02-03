# Awesome Symbiotic Relay [![Awesome](https://awesome.re/badge.svg)](https://awesome.re)

> Curated resources for the Symbiotic Relay — a distributed middleware mesh that derives validator sets, aggregates BLS (optionally ZK-compressed) signatures, and commits proofs across multiple settlement chains so any protocol can inherit stake-backed security without bootstrapping its own validator set.

Relay effectively acts as a universal attestation fabric: operators plug in to stake-backed validator set, sign outcomes, and have those results automatically gathered, and aggregatation proof is generated. Later aggregation proof can be verified on any EVM the chain where applications live. Whether you run a single lightweight client or a full stack of signer/aggregator/committer roles, Relay abstracts the validator choreography so products can focus on their business logic while Relay handles security guarantees end to end.

---

## Contents
- [About Relay](#about-relay)
- [Getting Started](#getting-started)
- [Architecture & Concepts](#architecture--concepts)
- [Running & Operating Nodes](#running--operating-nodes)
- [Smart Contracts & On-Chain Stack](#smart-contracts--on-chain-stack)
- [SDKs & Client Libraries](#sdks--client-libraries)
- [Tooling, Stats & Integrations](#tooling-stats--integrations)
- [Examples & Tutorials](#examples--tutorials)
- [ZK Circuits & Cryptography](#zk-circuits--cryptography)
- [Ecosystem & Community](#ecosystem--community)

---

## About Relay
- [**What is Relay**](https://docs.symbiotic.fi/get-started/what/relay) – explains how the Relay SDK plugs into Symbiotic’s shared-security marketplace so bridges, oracles, rollups, and AI/compute networks can codify custom slashing + rewards and verify outcomes anywhere.
- [**DeepWiki article**](https://deepwiki.com/symbioticfi/relay) – documents node roles, signal-based architecture, repository structure, API surface, deployment modes, and troubleshooting guidance.
- [**Use cases**](https://docs.symbiotic.fi/get-started/use-cases) – official docs highlight attestation networks, cross-chain messaging, oracle/data feeds, cloud & compute coordination, and liquidity/security routing as first-class scenarios.

---

## Getting Started
- [symbioticfi/relay](https://github.com/symbioticfi/relay) – Go reference implementation with binaries, Dockerfiles, CLI docs, buf-generated protobufs, and an automated local network (`make local-setup`).
- [README](https://github.com/symbioticfi/relay/blob/main/README.md) – quick-start instructions for releases, Docker images, configuration samples, and operational tips.
- [DEVELOPMENT.md](https://github.com/symbioticfi/relay/blob/main/DEVELOPMENT.md) – dev workflow, proto/codegen commands, e2e suites, buf usage.
- [docs/cli/relay_sidecar.md](https://github.com/symbioticfi/relay/blob/main/docs/cli/relay/relay_sidecar.md) – exhaustive CLI flag reference for `relay_sidecar` and `relay_utils`.

---

## Architecture & Concepts
- **Validator-set derivation** – the ValSet Driver contract exposes operator, vault, and voting-power state. Every epoch the relay reads that on-chain data, normalizes voting weights (via VotingPowerProvider rules), and produces a deterministic validator set header that is versioned locally and later committed on-chain for verification. Deterministic derivation pipeline allows relay to skip using classic consensus agreement to construct validator set.
- **Signing & aggregation** – Signer nodes watch for new signature requests tied to the active validator set chosen by application, produce BLS/ECDSA signatures, and gossip them to the mesh. Aggregators collect these partial signatures, enforce quorum/unsigner policies, and compress them either through native BLS aggregation (cheap for small sets) or by generating ZK proofs that keep verification costs flat even with thousands of validators.
- **Consensus vs Aggregation** - Relay doesn’t run consensus in the classical sense; instead it performs aggregation where a threshold of identical operator responses wheighted by voting power becomes the canonical proof. Aggregation threshold can be fully configured by application.
- **Sidecar model vs embedded lib** –  Running Relay as a sidecar means applications never deal with validator-set bookkeeping or proof generation directly. Because all communication happens over gRPC, client stacks can be written in any language while remaining decoupled from Relay internals.
- **Use cases** – Relay is a generic attestation and commitment fabric. Bridges, oracle/data feeds, rollups and shared sequencers, AI or compute marketplaces, cloud task networks, decentralized authentication, and even cross-chain governance can plug into the validator-set + proof pipeline to become verifiable and slashable without standing up bespoke validator fleets.
- **Further reading** – DeepWiki sections *System Architecture*, *Validator Set Management*, *Signature Aggregation*, *Zero-Knowledge Proof System*, and *Deployment Model* expand on the above concepts with diagrams and flow descriptions.

---

## Running & Operating Nodes

### Binaries & Images
- [GitHub Releases](https://github.com/symbioticfi/relay/releases) – download `relay_sidecar` & `relay_utils` binaries (Linux/macOS, amd64/arm64).
- [Docker Hub `symbioticfi/relay`](https://hub.docker.com/r/symbioticfi/relay) – official multi-arch image; pin tags for reproducibility (`docker pull symbioticfi/relay:<tag>`).
- [Makefile targets](https://github.com/symbioticfi/relay/blob/main/Makefile) – `make build-relay-sidecar OS=linux ARCH=amd64`, `make image TAG=dev`, etc.

### Configuration & Flags
- [example.config.yaml](https://github.com/symbioticfi/relay/blob/main/example.config.yaml) – canonical template covering logging, storage, key sources, P2P, sync, metrics, and policy knobs.
- Config priority: CLI flags > env vars (`SYMB_*`, e.g., `SYMB_LOG_LEVEL`) > YAML file (`--config`).
- Key sections:
  - `secret-keys[]` or `keystore` entries for Signer/Aggregator/Committer credentials (BLS BN254 or ECDSA).
  - `driver.chain-id` + `driver.address` for ValSet driver contract.
  - `p2p.listen`, `bootnodes`, `dht-mode`, `mdns` for Mesh connectivity.
  - `evm.chains[]` & `evm.max-calls` for multi-chain settlement RPC usage.

### Local Networks & E2E
- `make local-setup` – spins up an Anvil-based ecosystem with driver contracts, multiple relay sidecars, and sample operator roles (configurable via `OPERATORS`, `AGGREGATORS`, `COMMITERS` env vars).
- [`e2e/`](https://github.com/symbioticfi/relay/tree/main/e2e) – scripts/templates for generating sidecar configs (`e2e/scripts/sidecar-start.sh`) and orchestrated tests.
- Monitoring: enable metrics (`metrics.listen`, `metrics.pprof`) and tune logging via `log.mode` (`json`, `text`, `pretty`).

---

## Smart Contracts & On-Chain Stack
- [symbioticfi/relay-contracts](https://github.com/symbioticfi/relay-contracts) – Solidity modules + Foundry deployment tooling.
  - **VotingPowerProvider**: onboarding schemes (whitelist/blacklist/jail), shared/operator vaults, weighted/priced voting power calculators, hooks for slashing/rewards.
  - **KeyRegistry**: verifies BLS BN254 (`KeyBlsBn254`) and secp256k1 (`KeyEcdsaSecp256k1`) keys by namespace/tag.
  - **ValSetDriver**: data source for off-chain relay nodes deriving validator sets per epoch.
  - **Settlement**: verifies validator-set headers + aggregated proofs using `SigVerifierBlsBn254Simple` or ZK-friendly `SigVerifierBlsBn254ZK`.
  - Deployment script `script/relay-deploy.sh` orchestrates CreateX + TransparentUpgradeableProxy upgrades with TOML configs and custom Foundry hooks.
  - Audits documented under [`audits/`](https://github.com/symbioticfi/relay-contracts/tree/main/audits).

---

## SDKs & Client Libraries
- [symbioticfi/relay-client-ts](https://github.com/symbioticfi/relay-client-ts) – Connect RPC + @bufbuild-based TypeScript client, includes `examples/` with streaming, signing, and error-handling patterns (`npm run basic-usage`).
- [symbioticfi/relay-client-rs](https://github.com/symbioticfi/relay-client-rs) – Rust crate (`symbiotic-relay-client = "0.3.0"`) generated via buf/prost/tonic. Supports git/branch/tag overrides for nightly builds.
- [symbioticfi/relay](https://github.com/symbioticfi/relay/tree/main/api/client) – Go client package mirrored from RPC proto definitions.

---

## Tooling, Stats & Integrations
- [symbioticfi/relay-stats-ts](https://github.com/symbioticfi/relay-stats-ts) – TypeScript library for deriving validator-set snapshots, MiMC/SSZ helpers, aggregator extra-data builders, `getEpochData` convenience calls, and pluggable caches.
- [symbioticfi/network](https://github.com/symbioticfi/network) & [`symbioticfi/cli`](https://github.com/symbioticfi/cli) – reference repos for the broader Symbiotic stack (vaults, staking operations, CLI utilities) when wiring custody or operator flows.

---

## Examples & Tutorials
- [Symbiotic Super Sum](https://github.com/symbioticfi/symbiotic-super-sum) – end-to-end task network with Docker Compose, multiple relay sidecars, Anvil settlement chains, and sum nodes. Includes scripts to generate configs, run tasks via Foundry `cast`, and monitor logs.
- [Cosmos SDK + Relay](https://github.com/symbioticfi/cosmos-relay-sdk) – Cosmos SDK `SimApp` that replaces native staking/slashing with Symbiotic modules; supports both mock relay clients (JSON key files) and real sidecar RPCs.
- [Flight delays insurance](https://github.com/symbioticfi/symbiotic-flight-delays) - Simple flight delays insurance on-chain. Project implements flights oracle and on-chain insurance claims using Relay as attestation layer.
- DeepWiki subsections *Deployment Patterns* & *Troubleshooting Guide* – cover operational gotchas (port conflicts, syncing, chain RPC saturation).
- [Docs “Try Relay” flow](https://docs.symbiotic.fi/get-started/developers/relay-quickstart) – click-through tutorial for wiring the Relay SDK beta, guiding users through staking operators and finalizing cross-chain messages.

---

## ZK Circuits & Cryptography
- [symbioticfi/relay-bn254-example-circuit-keys](https://github.com/symbioticfi/relay-bn254-example-circuit-keys) – test-only proving/verifying keys for BN254 circuits; point `circuits-dir` or `--circuits-dir` at the downloaded path when experimenting with ZK aggregation.
- Settlement module’s `SigVerifierBlsBn254ZK` uses gnark-generated proofs for near-constant gas even with large validator sets; switch between `simple` and `zk` verification depending on validator count vs cost tolerance.

---

## Ecosystem & Community
- [Symbiotic Docs Portal](https://docs.symbiotic.fi/) – entry point for Learn/Integrate/Try flows, points program, analytics, audit references, and social channels (Discord, Telegram, X).
- [DeepWiki badges](https://deepwiki.com/symbioticfi/) – each Symbiotic repo exposes a DeepWiki badge linking to living architecture notes.
- Open-source contributions follow [`CONTRIBUTING.md`](https://github.com/symbioticfi/relay/blob/main/CONTRIBUTING.md) – branching strategy, lint/test requirements, PR etiquette.



Missing a great Relay resource? Open an issue or PR so the ecosystem’s awesome list stays up to date. 🚀
