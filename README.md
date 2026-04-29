# Lux Security

Security stack for the Lux ecosystem: audits, formal verification, key
management, threshold/MPC signing, post-quantum cryptography, and
vulnerability disclosure.

This repo is the public-facing index. Detailed audit reports and incident
post-mortems live in [`luxfi/audits`](https://github.com/luxfi/audits) (LaTeX
source) under embargo until publication.

## Stack

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Lux Security Stack                           │
├─────────────────────────────────────────────────────────────────────┤
│  Cryptography             │  Threshold / MPC      │  KMS            │
│  ──────────────           │  ───────────────      │  ─────          │
│  luxfi/crypto             │  luxfi/mpc            │  luxfi/kms      │
│   ML-KEM-768  (PQ KEM)    │   CGGMP21 ECDSA       │   MPC-backed    │
│   ML-DSA-65   (PQ sig)    │   FROST EdDSA         │   ZapDB store   │
│   SLH-DSA     (PQ sig)    │  luxfi/threshold      │   age + X-Wing  │
│   BLS12-381   (aggreg)    │   Ringtail (ML-DSA-65)│   replication   │
│   Ed25519, secp256k1      │  Hybrid PoP (BLS+RT)  │                 │
│   X-Wing (PQ+X25519)      │                       │                 │
├─────────────────────────────────────────────────────────────────────┤
│  Audits                   │  Formal Verification  │  Disclosure     │
│  ──────                   │  ───────────────────  │  ──────────     │
│  External (luxfi/audits)  │  Halmos symbolic      │  security@lux.network │
│  Internal red-team        │  Foundry invariants   │  PGP via .well-known/ │
│  Continuous SAST          │  Foundry fuzzing      │  Immunefi bounty │
│                           │  Slither + Aderyn     │  90-day embargo  │
│                           │  Semgrep + go-fuzz    │                  │
├─────────────────────────────────────────────────────────────────────┤
│  Signing & supply chain                                             │
│  ───────────────────────                                            │
│  Sigstore / Cosign image signatures                                 │
│  In-toto build attestations                                         │
│  Reproducible Go builds (CGO_ENABLED=0, stripped buildinfo)         │
│  GPG-signed RELEASE.md per tag                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## Cryptography — `luxfi/crypto`

Lux ships **NIST-PQC algorithms in production today**. Every signing and
key-exchange surface is **crypto-agile**: the verifier dispatches on the
algorithm tag, accepting both classical and post-quantum keys during the
migration window.

| Primitive | Status | Notes |
|-----------|--------|-------|
| **ML-KEM-768** (Kyber) | Production | NIST PQC FIPS 203 — KEM, used in X-Wing and KMS v2 wrapping |
| **ML-DSA-65** (Dilithium) | Production | NIST PQC FIPS 204 — signature, used as the **RT key** for Q-Chain validators |
| **SLH-DSA** (SPHINCS+) | Production | NIST PQC FIPS 205 — hash-based stateless signature |
| **X-Wing** | Production | Hybrid PQ+X25519 KEM — ML-KEM-768 + X25519, IETF draft |
| **BLS12-381** | Production | Aggregate signatures, validator-set threshold |
| **Ed25519** | Production | Classical signing |
| **secp256k1** | Production | EVM compatibility |
| **TFHE / LuxFHE** | Research | Fully homomorphic encryption bridge for private compute |

GPU-accelerated paths exist for ML-DSA, BLS, and ML-KEM (`gpu.go` in each
package). Batch verification + KAT (known-answer tests) included.

## Threshold + MPC — `luxfi/mpc` and `luxfi/threshold`

| Scheme | Use | Source |
|--------|-----|--------|
| **CGGMP21** | Threshold ECDSA (BLS keygen, signing) | `luxfi/mpc` |
| **FROST** | Threshold EdDSA (RT/ML-DSA keygen, signing) | `luxfi/mpc` |
| **Ringtail** | Threshold ML-DSA-65 framework | `luxfi/threshold` |

**HybridProofOfPossession** binds both BLS and RT (ML-DSA-65) keys atomically
in a single rotation transaction. The MPC daemon generates both key types in
one DKG session (CGGMP21 for BLS, FROST for RT) and produces the combined
proof. Q-Chain validators require this hybrid PoP — neither key alone is
sufficient.

## Key management — `luxfi/kms`

MPC-backed KMS. **No Infisical, no PostgreSQL, no Node.js.** Pure Go binary
backed by `luxfi/mpc` for threshold cryptography and `luxfi/zapdb` for
storage.

```
Client (ATS/BD/TA) → KMS (Go, :8080) → luxfi/mpc (CGGMP21/FROST via ZAP)
                              │
                         luxfi/zapdb (embedded, single-writer)
                              │
                         S3 replication: age-encrypted snapshots
                              │
                         X25519 + X-Wing hybrid (PQ-safe)
```

| Surface | Owner | Notes |
|---------|-------|-------|
| Validator BLS keys | `luxfi/kms` MPC vaults | CGGMP21 keygen + signing |
| Validator RT keys | `luxfi/kms` MPC vaults | FROST keygen, ML-DSA-65 signing (Ringtail) |
| Hot signing keys | Service-specific KMS namespace | Rotated on schedule |
| User key material | Client-side | Never reaches server |
| Replication at rest | age + X-Wing (PQ upgrade path) | S3 backups, hourly snap + 1s inc |

KMS API:

| Verb | Path | Op |
|------|------|----|
| POST | `/v1/vaults/{id}/wallets` | Keygen (CGGMP21 or FROST) |
| POST | `/v1/transactions` | Sign |
| POST | `/v1/wallets/{id}/reshare` | Reshare |

Operator authentication uses Universal Auth (client-id + client-secret →
short-lived bearer). Deploy CI fetches per-environment secrets via the
documented KMS endpoints — **never plaintext, never env files, never
plaintext in the database** (always hashed/encrypted).

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

Reports use LaTeX (per repo convention — `.md` is reserved for guides, not
audit/proof artefacts). The repo
[`luxfi/audits`](https://github.com/luxfi/audits) is canonical.

## Formal verification

Every protocol repo ships a verification harness:

- **Halmos** — symbolic execution over `check_*` properties (`make halmos`)
- **Foundry invariants** — handler-based stateful fuzzing in `src/test/Invariants/`
- **Foundry fuzzing** — 1000-run property tests on every PR
- **Slither** — static analysis on every PR
- **Aderyn** (Cyfrin) — additional Solidity analyzer
- **Semgrep SAST** — `p/solidity` + `p/smart-contracts` rulesets
- **go-fuzz / native `go test -fuzz`** — Go services and node, corpus checked in
- **`-race` mandatory** — race detection in CI for all Go code

Standard Makefile targets across protocol repos (e.g.
[`luxfi/liquid`](https://github.com/luxfi/liquid/blob/main/Makefile)):

```bash
make audit       # lint + tests + slither + semgrep + aderyn
make halmos      # symbolic property checking
make test-fuzz   # 1000-run fuzz suite
```

## Signing & supply chain

- **Container images** — signed with [Sigstore Cosign](https://sigstore.dev),
  pushed to `ghcr.io/luxfi/*`
- **Releases** — GPG-signed `RELEASE.md` manifest with sha256 of every artefact
- **Build provenance** — in-toto attestations attached to each image
- **Reproducible builds** — `CGO_ENABLED=0` Go binaries with stripped
  buildinfo, deterministic Docker

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

- **In-scope**: contracts, node, exchange, liquid, dex, bridge, KMS, IAM,
  compliance, broker, bank, DAO, MPC, threshold, crypto
- **Out-of-scope**: third-party deps (file with upstream), social
  engineering, physical attacks
- **Bounty**: tiered by severity (CVSS), paid in LUX or USDC

## Incident response

`security@lux.network` triages → on-call rotation → public post-mortem within
14 days of resolution. Post-mortems in this repo under `incidents/`.

Standing playbooks: contract pause, validator slashing, key rotation,
service rollback. Each owned by a team lead, reviewed quarterly.

## Standards & references

- [SLSA Level 3](https://slsa.dev) supply-chain target
- [OWASP ASVS](https://owasp.org/www-project-application-security-verification-standard/) v4.x for service code
- [SWC Registry](https://swcregistry.io/) for smart-contract weaknesses
- [NIST PQC](https://csrc.nist.gov/projects/post-quantum-cryptography) FIPS 203/204/205 algorithms only

## Related

| Repo | Role |
|------|------|
| [luxfi/audits](https://github.com/luxfi/audits) | LaTeX audit reports (canonical) |
| [luxfi/crypto](https://github.com/luxfi/crypto) | Cryptographic primitives (ML-KEM, ML-DSA, BLS, etc.) |
| [luxfi/mpc](https://github.com/luxfi/mpc) | Multi-party computation (CGGMP21, FROST) |
| [luxfi/threshold](https://github.com/luxfi/threshold) | Threshold signature schemes (Ringtail ML-DSA-65) |
| [luxfi/kms](https://github.com/luxfi/kms) | MPC-backed key management (validator + service keys) |
| [luxfi/dao](https://github.com/luxfi/dao) | On-chain governance (Safe + DLUX) |
| [luxfi/compliance](https://github.com/luxfi/compliance) | KYC, AML, regulatory frameworks |

## License

Copyright 2024-2026 Lux Partners Limited. Apache 2.0 unless otherwise noted.
