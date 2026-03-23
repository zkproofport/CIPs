---
cip: 3
title: OIDC Domain Attestation
author: ZKProofport (@zkproofport)
status: Review
category: Qualification
circuit-language: Noir
data-source: OIDC JWT id_token (Google Workspace, Microsoft 365)
created: 2026-02-13
requires: N/A
original-work: StealthNote by Saleel (https://github.com/saleel/stealthnote), MIT License
---

## Abstract

CIP-3 defines a zero-knowledge circuit that proves a user owns a valid organizational email address — from Google Workspace or Microsoft 365 — without revealing the email address, name, or any personal information. The proof extracts only the **domain** (e.g., `aztecprotocol.com`) from the OIDC JWT's email claim, proving organizational membership while keeping the user's identity private.

> **Attribution**: The core circuit architecture — JWT RSA signature verification via partial SHA, and the `noir-jwt` library — originates from the [StealthNote](https://github.com/saleel/stealthnote) project by [Saleel](https://github.com/saleel), licensed under MIT. ZKProofport's adaptation replaces StealthNote's ephemeral keypair model with a scope-bound nullifier system for sybil resistance, and adds Microsoft 365 OIDC support alongside the original Google Workspace support. See [Attribution](#attribution) for details.

## Motivation

Organizations frequently need to verify that a person belongs to a specific company or institution — for corporate deal rooms, governance, compliance, anonymous whistleblowing, and internal coordination. However, existing verification methods expose personal identity.

Google Workspace and Microsoft 365 power email for millions of organizations. When a user signs in via OIDC, the provider returns a JWT (id_token) signed with RSA-2048 containing the user's email domain. This circuit leverages that JWT to prove domain membership privately:

- **Anonymous corporate forums** — Post messages proving you work at a company, without revealing who you are (StealthNote's original use case)
- **Deal rooms & governance** — Gate access to corporate-only spaces without collecting employee data
- **Whistleblower protection** — Prove you're an insider without exposing your identity to anyone, including the OIDC provider
- **Cross-org collaboration** — Prove organizational affiliation to partners privately
- **AI agent delegation** — An agent can prove it acts on behalf of someone from a specific organization

## Circuit Specification

### Data Source

The circuit verifies an OIDC **JWT (id_token)** — a `$header.$payload.$signature` string where:

- `header`: Signature algorithm metadata (RSA SHA-256)
- `payload`: User claims including `email`, `email_verified` (Google) / `xms_edov` (Microsoft), and other standard OIDC claims
- `signature`: RSA-2048 signature by the provider's public key

**Supported providers:**

| Provider | Email verified claim | Domain source |
|----------|---------------------|---------------|
| Google Workspace (`provider=0`) | `email_verified: true` | `email` claim (domain after `@`) |
| Microsoft 365 (`provider=1`) | `xms_edov: true` | `email` claim (domain after `@`) |

### Public Inputs

| Index | Field | Type | Description |
|-------|-------|------|-------------|
| 0-17 | `pubkey_modulus_limbs` | `[u128; 18]` | OIDC provider's RSA-2048 public key modulus (split into 18 limbs) |
| 18 | `domain` | `BoundedVec<u8, 64>` | Organizational domain extracted from `email` claim (e.g., `aztecprotocol.com`) |
| 19-50 | `scope` | `[u8; 32]` | `keccak256(scope_string)` — dApp-defined scope for nullifier isolation |
| 51-82 | `nullifier` | `[u8; 32]` | `keccak256(keccak256(email) ‖ scope)` — privacy-preserving unique ID for sybil resistance |
| 83 | `provider` | `u8` | OIDC provider identifier: `0` = Google, `1` = Microsoft |

### Private Inputs

| Field | Type | Description |
|-------|------|-------------|
| `partial_data` | `BoundedVec<u8, 768>` | JWT data (header.payload) after partial SHA — only the tail containing email-related claims |
| `partial_hash` | `[u32; 8]` | 256-bit partial SHA hash of the preceding JWT data |
| `full_data_length` | `u32` | Total byte length of the full JWT data |
| `base64_decode_offset` | `u32` | Offset to align Base64 payload decoding |
| `redc_params_limbs` | `[u128; 18]` | RSA reduction parameters |
| `signature_limbs` | `[u128; 18]` | RSA signature limbs (provider's signature on the JWT) |

### Nullifier

- **Derivation**: `keccak256(keccak256(email) ‖ scope)`
- **Scope**: dApp-defined string (e.g., `"myapp.xyz"`) hashed to 32 bytes
- **Purpose**: Prevents the same email address from generating multiple proofs for the same scope (sybil resistance), while keeping the email private
- **Cross-scope unlinkability**: Different scopes produce different nullifiers for the same email, preventing cross-service tracking

> **Design change from StealthNote**: StealthNote's original design uses an ephemeral keypair model — no nullifier, and users can create unlimited anonymous identities. ZKProofport's adaptation adds scope-bound nullifiers for sybil resistance, which is required for use cases like anonymous community login where 1-person-1-account guarantees are needed.

### Verification Logic

```
PART 1: JWT RSA Signature Verification
├─ Reconstruct full SHA-256 hash from partial_hash + partial_data
├─ Verify RSA-2048 signature against provider's public key (pubkey_modulus_limbs)
└─ Confirm JWT is authentically signed by the OIDC provider

PART 2: Email & Domain Verification
├─ Assert email_verified (Google) or xms_edov (Microsoft) claim is true
├─ Extract email claim from JWT payload
├─ Find '@' character in email
├─ Assert domain bytes after '@' match the public domain input
└─ Assert email ends at domain end (no trailing characters)

PART 3: Nullifier Verification (Sybil Resistance)
├─ Compute email_hash = keccak256(email)
├─ Compute expected = keccak256(email_hash ‖ scope)
└─ Assert nullifier == expected
```

### Constraint Optimization

The circuit uses **partial SHA** to minimize constraints:
- JWT header + payload can be 2000+ bytes, but only the tail (containing email-related claims) is processed inside the circuit
- The SHA-256 hash of the prefix is computed outside the circuit and passed as `partial_hash`
- The circuit continues the SHA-256 computation from the partial state
- This dramatically reduces constraint count while maintaining soundness

### Dependencies

| Dependency | Source | Description |
|------------|--------|-------------|
| `jwt` | `../deps/noir-jwt` (path) | JWT verification library for Noir (originally by [Saleel](https://github.com/saleel/noir-jwt)) |
| `keccak256` | `../deps/keccak256` (path) | Keccak256 hash for nullifier derivation |

### Proof System

- **Language**: Noir (nargo 1.0.0-beta.8)
- **Backend**: Barretenberg UltraHonk
- **Mobile proving**: mopro (Rust FFI to Barretenberg)

## User Flow

```
1. User clicks "Sign in with Google" or "Sign in with Microsoft"
   (must be a Workspace/365 organizational account)
2. Provider returns JWT (id_token) containing email and domain claims
3. Client generates ZK proof (JWT is private input, domain is public output)
4. Proof + domain + nullifier sent to verifier
5. Verifier checks: (a) ZK proof is valid, (b) nullifier not already used (sybil check)
```

## Verification

### On-Chain Verifiers

| Chain | Address | Status |
|-------|---------|--------|
| Base Sepolia | — | Deployed |

### Off-Chain Verification

```typescript
import { verifyProof } from '@zkproofport-app/sdk';

const result = await verifyProof({
  proof,
  publicInputs,
  meta: { circuit: 'oidc_domain_attestation' },
  mode: 'offchain'
});
```

### SDK Integration

```typescript
import { ProofportSDK } from '@zkproofport-app/sdk';

const sdk = new ProofportSDK({ defaultCallbackUrl: 'https://myapp.com/verify' });
const request = sdk.createOidcDomainRequest({
  scope: 'myapp.xyz',
  allowedDomains: ['aztecprotocol.com', 'coinbase.com'] // optional filter
});
const { deepLink, qrDataUrl } = await sdk.requestProof(request);
```

## Security & Privacy

### Privacy Guarantees

| Data | Visible to Verifier? |
|------|---------------------|
| User's email address | ❌ Hidden |
| User's name | ❌ Hidden |
| JWT payload (all claims) | ❌ Hidden |
| Provider's RSA signature | ❌ Hidden |
| Organizational domain (e.g., `coinbase.com`) | ✅ Public |
| Nullifier (sybil resistance) | ✅ Public |
| OIDC provider (Google/Microsoft) | ✅ Public |

### Trust Assumptions

1. **OIDC provider attestation is honest** — Google/Microsoft accurately reflects organizational email domains
2. **Provider will not forge JWTs** — The provider could theoretically create fake JWTs for any domain
3. **RSA-2048 is secure** — The JWT signature scheme is sound
4. **Provider key rotation** — Providers rotate JWT signing keys; the `pubkey_modulus_limbs` public input identifies which key was used

### Known Limitations

- Currently supports **Google Workspace** and **Microsoft 365** (other OIDC providers planned via `provider` field extension)
- Provider key rotation means proofs are only valid while the signing key is active
- Email-based nullifier: if a user's email changes (e.g., leaves company), the old nullifier becomes stale

### Audit Status

- [x] Internal review (StealthNote original + ZKProofport adaptation)
- [ ] External audit
- [ ] Formal verification

## Attribution

This circuit builds upon [StealthNote](https://github.com/saleel/stealthnote) by [Saleel](https://github.com/saleel), released under the [MIT License](https://github.com/saleel/stealthnote/blob/main/LICENSE).

**From StealthNote (Saleel's original contributions):**
- JWT RSA signature verification via partial SHA (constraint optimization technique)
- The [`noir-jwt`](https://github.com/saleel/noir-jwt) library for Noir
- Email claim extraction and domain verification logic
- The fundamental insight that OIDC JWTs can serve as a data source for ZK proofs

**ZKProofport's adaptations:**
- **Nullifier system**: Replaced StealthNote's ephemeral keypair model with scope-bound keccak256 nullifiers (`keccak256(keccak256(email) ‖ scope)`) for sybil resistance
- **Microsoft 365 support**: Added `provider` field and `xms_edov` claim verification for Microsoft OIDC
- **Removed ephemeral keypair**: StealthNote's `ephemeral_pubkey`, `ephemeral_pubkey_salt`, and Poseidon2 nonce verification were removed in favor of the nullifier-based identity model
- **Integration with ZKProofport ecosystem**: Mobile proving via mopro, SDK integration, on-chain verification

Technical details of StealthNote's original design are documented in Saleel's blog post: [StealthNote — ZK Proof of JWT](https://saleel.xyz/blog/stealthnote/).

## License

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
