---
cip: 1
title: Coinbase KYC Attestation Circuit
author: ZKProofport (@zkproofport)
status: Review
category: Identity
circuit-language: Noir
data-source: EAS (Ethereum Attestation Service) — Coinbase Verified Account
created: 2026-02-13
---

## Abstract

CIP-1 defines a zero-knowledge circuit that proves a user holds a valid Coinbase KYC attestation — without revealing their address, transaction data, or the attester's signature. The proof confirms "this person has been KYC'd by Coinbase" while exposing exactly zero bytes of personal data.

## Motivation

Decentralized applications increasingly need to verify user identity for compliance, access control, and sybil resistance. However, traditional KYC exposes sensitive personal information.

Coinbase has verified 100M+ users and publishes on-chain attestations via EAS. This circuit leverages those existing attestations to create a privacy-preserving identity layer:

- **DeFi protocols** can gate access to KYC'd users without collecting personal data
- **DAOs** can implement sybil-resistant voting
- **Airdrops** can target verified accounts without address linking
- **AI agents** can prove they act on behalf of a KYC'd human

## Circuit Specification

### Public Inputs

| Index | Field | Type | Description |
|-------|-------|------|-------------|
| 0-31 | `signal_hash` | `[u8; 32]` | Anti-replay challenge from the dApp |
| 32-63 | `signer_list_merkle_root` | `[u8; 32]` | Merkle root of valid Coinbase signers |
| 64-95 | `scope` | `[u8; 32]` | `keccak256(scope_string)` for sybil resistance |
| 96-127 | `nullifier` | `[u8; 32]` | `keccak256(user_address ‖ signal_hash ‖ scope)` |

### Private Inputs

| Field | Type | Description |
|-------|------|-------------|
| `user_address` | `[u8; 20]` | User's Ethereum address (hidden) |
| `user_signature` | `[u8; 64]` | User's signature on `signal_hash` (r + s) |
| `user_pubkey_x`, `user_pubkey_y` | `[u8; 32]` each | User's public key |
| `raw_transaction` | `[u8; 300]` | RLP-encoded Coinbase attestation TX (EIP-1559) |
| `tx_length` | `u32` | Actual transaction byte length |
| `coinbase_attester_pubkey_x`, `_y` | `[u8; 32]` each | Coinbase signer's public key (pre-recovered via JS ecrecover) |
| `coinbase_signer_merkle_proof` | `[[u8; 32]; 8]` | Merkle proof for signer validity |
| `coinbase_signer_leaf_index` | `u32` | Leaf position in Merkle tree |
| `merkle_proof_depth` | `u32` | Depth of Merkle proof |

### Nullifier

- **Derivation**: `keccak256(user_address ‖ signal_hash ‖ scope)`
- **Scope**: dApp-defined string (e.g., `"myapp.xyz"`) hashed to 32 bytes
- **Purpose**: Prevents the same user from generating multiple proofs for the same scope (sybil resistance)
- **On-chain registration**: Via `ZKProofportNullifierRegistry.verifyAndRegister()`

### Verification Logic

```
PART 1: User Ownership
├─ Derive address from user's public key → must match user_address
└─ Verify user's signature on signal_hash

PART 2: Attestation Authenticity (Hybrid Model)
├─ Parse RLP-encoded EIP-1559 transaction
├─ Extract and verify Coinbase signer's signature on TX
├─ Verify signer is in valid list (Merkle proof against public root)
└─ Verify TX destination = Coinbase attester contract (0x357458739F90461b99789350868CD7CF330Dd7EE)

PART 3: Logical Link
├─ Verify TX calls attestAccount(address) function (selector: 0x56feed5e)
└─ Verify calldata address matches user_address

PART 4: Sybil Resistance
└─ Verify nullifier is correctly derived from user_address, signal_hash, scope
```

### Dependencies

- `keccak256` — `noir-lang/keccak256` v0.1.1
- `coinbase_libs` — Shared library for EIP-1559 TX parsing, Merkle verification, Ethereum utilities

### Proof System

- **Language**: Noir (nargo 1.0.0-beta.8)
- **Backend**: Barretenberg UltraHonk
- **Mobile proving**: mopro (Rust FFI to Barretenberg)

## Verification

### On-Chain Verifiers

| Chain | Address | Status |
|-------|---------|--------|
| Base Sepolia | `0xEb9eb5452790Cfe549fF83CEB3Dbe1C432231492` | Deployed |

### Off-Chain Verification

```typescript
import { verifyProof } from '@proofport/sdk';

const result = await verifyProof({
  proof,
  publicInputs,
  meta: { circuit: 'coinbase_attestation' },
  mode: 'offchain' // Uses @aztec/bb.js
});
```

### SDK Integration

```typescript
import { ProofportSDK } from '@proofport/sdk';

const sdk = new ProofportSDK({ defaultCallbackUrl: 'https://myapp.com/verify' });
const request = sdk.createCoinbaseKycRequest({ scope: 'myapp.xyz' });
const { deepLink, qrDataUrl } = await sdk.requestProof(request);
```

## Security & Privacy

### Privacy Guarantees

| Data | Visible to Verifier? |
|------|---------------------|
| User's Ethereum address | ❌ Hidden |
| User's public key | ❌ Hidden |
| Coinbase attestation TX | ❌ Hidden |
| Coinbase signer's identity | ❌ Hidden |
| KYC status (yes/no) | ✅ Proved via valid proof |
| Nullifier | ✅ Public (sybil resistance) |

### Security Assumptions

1. The Coinbase EAS attestation is authentic and has not been revoked
2. The Merkle root of valid signers is maintained honestly by the dApp
3. The keccak256 hash function is collision-resistant
4. ECDSA signature verification is sound

### Audit Status

- [x] Internal review
- [ ] External audit
- [ ] Formal verification

## License

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
