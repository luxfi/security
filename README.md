# Lux Security

Security stack for the Lux ecosystem: audits, formal verification, MPC-backed
key management, post-quantum cryptography, threshold signing, **Quasar**
post-quantum consensus, and vulnerability disclosure.

This repo is the public-facing index. Detailed audit reports and incident
post-mortems live in [`luxfi/audits`](https://github.com/luxfi/audits) (LaTeX
source) under embargo until publication.

## Stack at a glance

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Lux Security Stack                            │
├──────────────────────────────────────────────────────────────────────┤
│  Cryptography (luxfi/crypto)                                         │
│  ──────────────────────────                                          │
│   Classical:  BLS12-381, Ed25519, secp256k1, X25519                  │
│   PQ KEM:     ML-KEM-768 (FIPS 203), X-Wing hybrid (ML-KEM + X25519) │
│   PQ Sig:     ML-DSA-65 (FIPS 204), SLH-DSA (FIPS 205)               │
│   PQ Threshold: Ringtail (Ring-LWE / Module-LWE, 2-round)            │
│   ZK:         Groth16, Plonk (Z-Chain precompiles)                   │
│   FHE:        TFHE / LuxFHE (Z-Chain private compute)                │
├──────────────────────────────────────────────────────────────────────┤
│  Threshold + MPC                                                     │
│  ────────────────                                                    │
│   luxfi/mpc:        CGGMP21 (ECDSA), FROST (EdDSA)                   │
│   luxfi/threshold:  Ringtail Ring-LWE 2-round threshold              │
│                     (NOT threshold ML-DSA — separate scheme)         │
├──────────────────────────────────────────────────────────────────────┤
│  Consensus (Quasar)                                                  │
│  ──────────────────                                                  │
│   Three independent signing paths, each toggleable:                  │
│     1. BLS                           — classical, fastest            │
│     2. Ringtail                      — PQ threshold (lattice)        │
│     3. ML-DSA-65                     — PQ identity (FIPS 204)        │
│   Modes: BLS-only / BLS+ML-DSA / BLS+Ringtail / Triple (full Quasar) │
├──────────────────────────────────────────────────────────────────────┤
│  Cross-chain (Warp V2)                                               │
│  ─────────────────────                                               │
│   PQ messaging via random Ringtail validation                        │
│   Private messaging via Z-Chain FHE                                  │
├──────────────────────────────────────────────────────────────────────┤
│  KMS (luxfi/kms)              │  Audits        │  Disclosure         │
│  ───────────────              │  ────────      │  ──────────         │
│   MPC-backed (CGGMP21/FROST)  │  luxfi/audits  │  security@lux.network │
│   ZapDB storage               │  LaTeX, peer   │  PGP via .well-known/ │
│   age + X-Wing replication    │  reviewed      │  Immunefi bounty    │
└──────────────────────────────────────────────────────────────────────┘
```

## Cryptography — `luxfi/crypto`

Lux runs **NIST-PQC algorithms in production today**. Every signing and
key-exchange surface is **crypto-agile**: the verifier dispatches on the
algorithm tag, accepting both classical and post-quantum keys during the
migration window.

| Primitive | Spec | Class | Use |
|-----------|------|-------|-----|
| **ML-KEM-768** | FIPS 203 | PQ KEM (Module-LWE) | Hybrid KEM via X-Wing, KMS v2 wrapping |
| **X-Wing** | IETF draft | Hybrid KEM | ML-KEM-768 + X25519 |
| **ML-DSA-65** | FIPS 204 | PQ sig (Module-LWE+SIS) | Identity proofs, validator RT keys |
| **SLH-DSA** | FIPS 205 | PQ sig (hash-based) | Stateless backup signature |
| **BLS12-381** | — | Classical pairing | Aggregate sigs, validator threshold |
| **Ed25519** | — | Classical | Service signing |
| **secp256k1** | — | Classical | EVM compatibility |
| **TFHE / LuxFHE** | — | FHE | Z-Chain private compute |

GPU-accelerated paths exist for ML-DSA, ML-KEM, BLS. KAT (known-answer test)
suites included for every PQ primitive.

## Threshold + MPC — `luxfi/mpc`, `luxfi/threshold`

Three distinct threshold paths, each used for a different purpose:

| Scheme | Type | Round complexity | Repo | Use |
|--------|------|------------------|------|-----|
| **CGGMP21** | Threshold ECDSA | Multi-round | `luxfi/mpc` | secp256k1 keygen + signing |
| **FROST** | Threshold EdDSA | 2-round | `luxfi/mpc` | Ed25519 keygen + signing |
| **Ringtail** | Threshold lattice | **2-round** | `luxfi/threshold` (uses [`luxfi/ringtail`](https://github.com/luxfi/ringtail)) | **PQ threshold** for Quasar consensus and Warp V2 |

> **Ringtail is its own Ring-LWE / Module-LWE 2-round threshold scheme — not
> "threshold ML-DSA". They are separate cryptographic constructions that both
> happen to be lattice-based.**

ML-DSA-65 (FIPS 204) is used in Quasar **as a single-signer PQ identity
signature, run alongside the threshold path** — not as a threshold scheme.

## Consensus — `luxfi/consensus` (Quasar family)

Quasar is the Lux consensus engine. P-Chain uses Quasar as the post-quantum
overlay for both linear (Nova) and DAG (Nebula) consensus modes.

```
Photon → Wave → Focus → Nova/Nebula → Quasar
(committee) (vote) (confidence) (chain mode) (PQ overlay)
```

**Three independent signing paths** in Quasar, each toggleable:

| Path | Layer | Scheme |
|------|-------|--------|
| 1 | BLS | BLS12-381 threshold (classical) |
| 2 | Ringtail | Ring-LWE 2-round threshold (PQ) |
| 3 | ML-DSA-65 | FIPS 204 identity signatures (PQ) |

| Mode | Composition | Use case |
|------|-------------|----------|
| **BLS-only** | path 1 | Fastest classical, PoA-style deployments |
| **BLS + ML-DSA** | paths 1+3 | Dual with PQ identity proof |
| **BLS + Ringtail** | paths 1+2 | Dual with PQ threshold |
| **Triple (full Quasar)** | paths 1+2+3 | Full PQ — `IsTripleMode()` returns true |

In triple mode, `TripleSignRound1` runs all three signing paths in parallel
goroutines. Each layer can be enabled per-deployment based on threat model
and performance requirements.

## Z-Chain — PQ identity via ML-DSA + Groth16

Lux Z-Chain (`luxfi/chains/zkvm`) provides **PQ identity** via ML-DSA-65
signatures verified inside Groth16 zero-knowledge proofs. This pattern lets
PQ identity attestations be folded into existing zk circuits without
revealing the underlying identity material.

Z-Chain also provides:

- **Groth16 + Plonk** verification precompiles (`VerifierTypeGroth16 = 0x01`)
- **FHE compute** via TFHE/LuxFHE (`fhe/` subdir) for private state
- **Cross-chain proofs** via the `crosschain` precompile

These primitives back **Quasar's PQ identity path** and **Warp V2 private
messaging**.

## Q-Chain — quantum-resistant VM

`luxfi/chains/quantumvm` (Q-Chain) is the production quantum-resistant VM:

- Ringtail key support (configurable `RingtailKeySize`, on/off via `RingtailEnabled`)
- Quantum signature verification with cache (`QuantumSigCacheSize`)
- Quantum stamp validation with time-window enforcement (`QuantumStampWindow`)
- Parallel transaction processing for high throughput
- Versioned PQ algorithms (`QuantumAlgorithmVersion`)

Q-Chain validators require a **HybridProofOfPossession** binding both BLS and
RT (ML-DSA-65) keys atomically, generated in a single MPC DKG session
(CGGMP21 for BLS, FROST for RT).

## Warp V2 — `luxfi/warp`

Cross-chain messaging with PQ safety:

- **Random Ringtail validation** — PQ-safe message authentication across chains
- **Z-Chain FHE** — private cross-chain messages
- Protocol-first definitions in `protocol/*.proto`
- Chain-agnostic backends — EVM, non-EVM, custom VMs

## Key management — `luxfi/kms`

MPC-backed Lux KMS. **No Infisical, no PostgreSQL, no Node.js.** Pure Go
binary backed by `luxfi/mpc` for threshold cryptography and `luxfi/zapdb` for
storage.

```
Client (ATS/BD/TA) → KMS (Go, :8080) → luxfi/mpc (CGGMP21/FROST via ZAP)
                              │
                         luxfi/zapdb (embedded)
                              │
                         S3 replication: age-encrypted
                              │
                         X25519 + X-Wing hybrid (PQ upgrade path)
```

KMS API:

| Verb | Path | Op |
|------|------|----|
| POST | `/v1/vaults/{id}/wallets` | Keygen (CGGMP21 or FROST) |
| POST | `/v1/transactions` | Sign |
| POST | `/v1/wallets/{id}/reshare` | Reshare without downtime |

Operator authentication via Universal Auth (client-id + client-secret →
short-lived bearer). Deploy CI fetches per-environment secrets via KMS
endpoints — **never plaintext, never env files, never plaintext in
databases**.

## Audits — `luxfi/audits`

External audit reports, peer-reviewed scope (LaTeX, never `.md`):

| Date | Component | Source |
|------|-----------|--------|
| 2025-12-11 | DEX VM | `2025-12-11-dexvm-audit.tex` |
| 2025-12-11 | Oracle | `2025-12-11-oracle-audit.tex` |
| 2025-12-11 | Perpetuals | `2025-12-11-perpetuals-audit.tex` |
| 2025-12-30 | Architecture | `2025-12-30-architecture-review.tex` |
| 2025-12-30 | Consensus | `2025-12-30-consensus-audit.tex` |
| 2025-12-30 | Smart contracts | `2025-12-30-contracts-audit.tex` |
| 2025-12-30 | Cryptography | `2025-12-30-crypto-audit.tex` |
| 2025-12-30 | Database | `2025-12-30-database-audit.tex` |
| 2025-12-30 | DEX VM | `2025-12-30-dexvm-audit.tex` |
| 2025-12-30 | Network | `2025-12-30-network-audit.tex` |

## Formal verification

Standard harness across protocol repos:

- **Halmos** — symbolic execution over `check_*` properties (`make halmos`)
- **Foundry invariants** — handler-based stateful fuzzing
- **Foundry fuzzing** — 1000-run property tests in CI
- **Slither + Aderyn** — Solidity static analysis
- **Semgrep SAST** — `p/solidity` + `p/smart-contracts`
- **`go test -fuzz`** — Go services and node, corpus checked in
- **`-race` mandatory** — race detection on every Go CI run

```bash
make audit       # lint + tests + slither + semgrep + aderyn
make halmos      # symbolic property checking
make test-fuzz   # 1000-run fuzz suite
```

## Signing & supply chain

- **Container images** — Sigstore Cosign, pushed to `ghcr.io/luxfi/*`
- **Releases** — GPG-signed `RELEASE.md` with sha256 of every artefact
- **Build provenance** — in-toto attestations attached to each image
- **Reproducible builds** — `CGO_ENABLED=0`, stripped buildinfo, deterministic Docker

```bash
cosign verify --certificate-identity-regexp '^https://github.com/luxfi/' \
  ghcr.io/luxfi/<service>:<tag>
```

## Vulnerability disclosure

| | |
|---|---|
| Email | `security@lux.network` |
| PGP | [`/.well-known/security.txt`](https://lux.network/.well-known/security.txt) |
| Embargo | 90 days standard |
| Bug bounty | [Immunefi](https://immunefi.com) |

In-scope: contracts, node, exchange, liquid, dex, bridge, KMS, consensus,
warp, MPC, threshold, crypto, compliance, broker, bank, DAO, Z-Chain, Q-Chain.
Out-of-scope: third-party deps (file with upstream), social engineering,
physical attacks. Bounty tiered by severity (CVSS), paid in LUX or USDC.

## Standards

- [SLSA Level 3](https://slsa.dev) supply-chain target
- [OWASP ASVS](https://owasp.org/www-project-application-security-verification-standard/) v4.x for service code
- [SWC Registry](https://swcregistry.io/) for smart-contract weaknesses
- [NIST PQC](https://csrc.nist.gov/projects/post-quantum-cryptography) FIPS 203 / 204 / 205 only

## Related repos

| Repo | Role |
|------|------|
| [luxfi/audits](https://github.com/luxfi/audits) | LaTeX audit reports (canonical) |
| [luxfi/crypto](https://github.com/luxfi/crypto) | Cryptographic primitives (PQ + classical) |
| [luxfi/mpc](https://github.com/luxfi/mpc) | Multi-party computation (CGGMP21, FROST) |
| [luxfi/threshold](https://github.com/luxfi/threshold) | Threshold signature framework (multi-chain) |
| [luxfi/ringtail](https://github.com/luxfi/ringtail) | Ringtail Ring-LWE 2-round threshold |
| [luxfi/lattice](https://github.com/luxfi/lattice) | Lattice operations + GPU acceleration |
| [luxfi/consensus](https://github.com/luxfi/consensus) | Quasar consensus family (BLS + Ringtail + ML-DSA) |
| [luxfi/warp](https://github.com/luxfi/warp) | Cross-chain messaging (PQ via Ringtail, private via Z-Chain FHE) |
| [luxfi/chains](https://github.com/luxfi/chains) | VM plugins (Z-Chain zkvm, Q-Chain quantumvm, etc.) |
| [luxfi/kms](https://github.com/luxfi/kms) | MPC-backed key management |
| [luxfi/dao](https://github.com/luxfi/dao) | On-chain governance (Safe + DLUX) |
| [luxfi/compliance](https://github.com/luxfi/compliance) | KYC, AML, regulatory frameworks |

## License

Copyright 2024-2026 Lux Partners Limited. Apache 2.0 unless otherwise noted.
