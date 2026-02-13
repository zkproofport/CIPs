---
cip: 3
title: Corporate Domain Attestation (JWT)
author: ZKProofport (@zkproofport)
status: Draft
category: Identity
circuit-language: Noir
data-source: Google OAuth 2.0 OpenID Connect (JWT id_token)
created: 2026-02-13
requires: N/A
original-work: StealthNote by Saleel (https://github.com/saleel/stealthnote), MIT License
---

## Abstract

CIP-3 defines a zero-knowledge circuit that proves a user owns a valid corporate email address (Google Workspace) without revealing the email address, name, or any personal information. The proof extracts only the **domain** (e.g., `aztecprotocol.com`) from the JWT's email claim, proving organizational membership while keeping the user's identity private.

> **Attribution**: This CIP is based on the [StealthNote](https://github.com/saleel/stealthnote) project by [Saleel](https://github.com/saleel), licensed under MIT. The circuit design, `noir-jwt` library, and ephemeral keypair architecture are Saleel's original work. This CIP proposes adopting and standardizing the circuit within the ZKProofport ecosystem with proper attribution.

## Motivation

Organizations frequently need to verify that a person belongs to a specific company or institution — for corporate deal rooms, governance, compliance, anonymous whistleblowing, and internal coordination. However, existing verification methods expose personal identity.

Google Workspace powers email for millions of organizations. When a user signs in with Google, the OAuth flow returns a JWT (id_token) signed by Google containing the user's email domain (`hd` claim). This circuit leverages that JWT to prove domain membership privately:

- **Anonymous corporate forums** — Post messages proving you work at a company, without revealing who you are (StealthNote's original use case)
- **Deal rooms & governance** — Gate access to corporate-only spaces without collecting employee data
- **Whistleblower protection** — Prove you're an insider without exposing your identity to anyone, including Google
- **Cross-org collaboration** — Prove organizational affiliation to partners privately
- **AI agent delegation** — An agent can prove it acts on behalf of someone from a specific organization

## Circuit Specification

### Data Source

The circuit verifies a Google OAuth 2.0 **JWT (id_token)** — a `$header.$payload.$signature` string where:

- `header`: Signature algorithm metadata (RSA SHA-256)
- `payload`: User claims including `email`, `hd` (hosted domain), `email_verified`, `nonce`
- `signature`: RSA-2048 signature by Google's public key

### Public Inputs

| Index | Field | Type | Description |
|-------|-------|------|-------------|
| 0-17 | `jwt_pubkey_modulus_limbs` | `[u128; 18]` | Google's RSA-2048 public key modulus (split into 18 limbs) |
| 18 | `domain` | `BoundedVec<u8, 64>` | Corporate domain extracted from `email` claim (e.g., `aztecprotocol.com`) |
| 19 | `ephemeral_pubkey` | `Field` | Public key of the user's ephemeral keypair (for message signing) |
| 20 | `ephemeral_pubkey_expiry` | `u32` | Expiry timestamp for the ephemeral keypair |

### Private Inputs

| Field | Type | Description |
|-------|------|-------------|
| `partial_data` | `BoundedVec<u8, 640>` | JWT data (header.payload) after partial SHA — only the tail containing `hd` and `nonce` |
| `partial_hash` | `[u32; 8]` | 256-bit partial SHA hash of the preceding JWT data |
| `full_data_length` | `u32` | Total byte length of the full JWT data |
| `base64_decode_offset` | `u32` | Offset to align Base64 payload decoding |
| `jwt_pubkey_redc_params_limbs` | `[u128; 18]` | RSA reduction parameters |
| `jwt_signature_limbs` | `[u128; 18]` | RSA signature limbs (Google's signature on the JWT) |
| `ephemeral_pubkey_salt` | `Field` | Salt for ephemeral pubkey hash (privacy from Google) |

### Verification Logic

```
PART 1: JWT Signature Verification
├─ Reconstruct full SHA-256 hash from partial_hash + partial_data
├─ Verify RSA-2048 signature against Google's public key
└─ Confirm JWT is authentically signed by Google

PART 2: Nonce Verification (Ephemeral Keypair Binding)
├─ Extract `nonce` claim from JWT payload
├─ Compute Poseidon2(ephemeral_pubkey, ephemeral_pubkey_salt, ephemeral_pubkey_expiry)
└─ Assert nonce == computed hash (binds keypair to JWT)

PART 3: Email & Domain Verification
├─ Assert `email_verified` claim is true
├─ Extract `email` claim from JWT payload
├─ Find '@' character in email
└─ Assert domain bytes after '@' match the public `domain` input

PART 4: Privacy from Google
└─ Salt in nonce hash prevents Google from linking ephemeral_pubkey to OAuth logs
```

### Constraint Optimization

The circuit uses **partial SHA** to minimize constraints:
- JWT header + payload can be 2000+ bytes, but only the tail (containing `hd`, `nonce`) is processed inside the circuit
- The SHA-256 hash of the prefix is computed outside the circuit and passed as `partial_hash`
- The circuit continues the SHA-256 computation from the partial state
- This dramatically reduces constraint count while maintaining soundness

### Dependencies

| Dependency | Version | Description |
|------------|---------|-------------|
| `noir-jwt` | v0.4.4 | JWT verification library for Noir (by [saleel](https://github.com/saleel/noir-jwt)) |
| Poseidon2 | `std::hash::poseidon2` | Noir standard library Poseidon2 hash |

### Proof System

- **Language**: Noir (nargo ≥ 1.0.0)
- **Backend**: Barretenberg UltraHonk
- **Proving environment**: Browser (client-side via WASM)

## User Flow

```
1. User clicks "Sign in with Google" (Google Workspace account)
2. Client generates ephemeral EdDSA keypair + random salt
3. Client sets nonce = Poseidon2(ephemeral_pubkey, salt, expiry)
4. Google returns JWT (id_token) with nonce embedded
5. Client generates ZK proof in browser (JWT is private input)
6. Proof + ephemeral_pubkey + domain sent to verifier
7. User signs messages with ephemeral private key
8. Verifier checks: (a) ZK proof is valid, (b) message signature matches ephemeral_pubkey
```

## Verification

### On-Chain Verifiers

| Chain | Address | Status |
|-------|---------|--------|
| — | — | Not yet deployed |

### Off-Chain Verification

```typescript
import { verifyProof } from '@proofport/sdk';

const result = await verifyProof({
  proof,
  publicInputs,
  meta: { circuit: 'corporate_domain_attestation' },
  mode: 'offchain'
});
```

### SDK Integration

```typescript
import { ProofportSDK } from '@zkproofport-app/sdk';

const sdk = new ProofportSDK({ defaultCallbackUrl: 'https://myapp.com/verify' });
const request = sdk.createCorporateDomainRequest({
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
| Google's signature | ❌ Hidden |
| Corporate domain (e.g., `coinbase.com`) | ✅ Public |
| Ephemeral public key | ✅ Public |

### Privacy from Google

Google sees the `nonce` during OAuth, which is `Poseidon2(ephemeral_pubkey, salt, expiry)`. Since `salt` is secret and Poseidon2 is a ZK-friendly hash, Google cannot derive `ephemeral_pubkey` from the nonce, preventing them from linking messages to OAuth sessions.

### Trust Assumptions

1. **Google Workspace attestation is honest** — Google accurately reflects organizational email domains
2. **Google will not forge JWTs** — Google could theoretically create fake JWTs for any domain
3. **RSA-2048 is secure** — The JWT signature scheme is sound
4. **Google key rotation** — Google rotates JWT signing keys frequently; proofs are only valid while the key is active
5. **Timing attacks** — If a user generates a proof immediately after OAuth, Google could correlate timestamps (mitigated by batching)

### Known Limitations

- Currently supports **Google Workspace only** (Microsoft, other SSO providers planned)
- No nullifier — the same user can generate multiple ephemeral keypairs (by design for anonymity, but limits sybil resistance)
- Google key rotation means proofs have a limited validity window

### Audit Status

- [x] Internal review (StealthNote)
- [ ] External audit
- [ ] Formal verification

## Attribution

This circuit is based on [StealthNote](https://github.com/saleel/stealthnote) by [Saleel](https://github.com/saleel), released under the [MIT License](https://github.com/saleel/stealthnote/blob/main/LICENSE). The `noir-jwt` library used by this circuit is also authored by Saleel and available at [github.com/saleel/noir-jwt](https://github.com/saleel/noir-jwt).

Technical details and design rationale are documented in Saleel's blog post: [StealthNote — ZK Proof of JWT](https://saleel.xyz/blog/stealthnote/).

## License

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
