---
cip: 2
title: Coinbase Country Attestation Circuit
author: ZKProofport (@zkproofport)
status: Review
category: Identity
circuit-language: Noir
data-source: EAS (Ethereum Attestation Service) — Coinbase Verified Country
created: 2026-02-13
requires: CIP-1
---

## Abstract

CIP-2 defines a zero-knowledge circuit that proves a user's Coinbase-attested country matches (or does not match) a specified list — without revealing the user's address, country code, or any transaction data. The circuit supports both **inclusion mode** ("user is from one of these countries") and **exclusion mode** ("user is NOT from these countries").

## Motivation

Regulatory compliance often requires geographic restrictions. Current solutions demand users disclose their location, which creates privacy risks and data liability for dApps.

CIP-2 extends CIP-1's attestation verification with country-specific logic:

- **DeFi protocols** can enforce OFAC sanctions compliance without collecting user data
- **Token sales** can restrict geographic regions without KYC providers
- **Gaming platforms** can comply with local gambling laws privately
- **DAOs** can create region-specific governance without doxxing members

## Circuit Specification

### Public Inputs

| Index | Field | Type | Description |
|-------|-------|------|-------------|
| 0-31 | `signal_hash` | `[u8; 32]` | Anti-replay challenge from the dApp |
| 32-63 | `signer_list_merkle_root` | `[u8; 32]` | Merkle root of valid Coinbase signers |
| 64-83 | `country_list` | `[[u8; 2]; 10]` | List of 2-byte ISO 3166-1 country codes (max 10, zero-padded) |
| 84 | `country_list_length` | `u32` | Number of valid entries in country_list (1–10) |
| 85 | `is_included` | `bool` | `true` = inclusion (must be in list), `false` = exclusion (must NOT be in list) |
| 86-117 | `scope` | `[u8; 32]` | `keccak256(scope_string)` for sybil resistance |
| 118-149 | `nullifier` | `[u8; 32]` | `keccak256(user_address ‖ signal_hash ‖ scope)` |

### Private Inputs

Same as CIP-1, plus:

| Field | Type | Description |
|-------|------|-------------|
| `raw_transaction` | `[u8; 300]` | RLP-encoded Coinbase **country** attestation TX |

All CIP-1 private inputs apply (user_address, user_signature, user_pubkey, coinbase_attester_pubkey, merkle_proof).

### Nullifier

Same 2-step derivation as CIP-1: `user_secret = keccak256(user_address ‖ signal_hash)` → `nullifier = keccak256(user_secret ‖ scope)`

### Verification Logic

```
PARTS 1-2: Same as CIP-1 (User Ownership + Attestation Authenticity)

PART 3: Country Verification
├─ Verify TX calls attestCountry(uint256) (selector: 0x0a225248)
├─ Extract 2-byte country code from calldata[14..16]
├─ Verify calldata address matches user_address
├─ Validate country_list_length (1 ≤ length ≤ 10)
├─ If is_included == true:
│   └─ Assert country IS in country_list[0..country_list_length]
└─ If is_included == false:
    └─ Assert country is NOT in country_list[0..country_list_length]

PART 4: Sybil Resistance (same as CIP-1)
```

### Country Code Encoding

Country codes use ISO 3166-1 alpha-2 encoding (2 ASCII bytes):

| Code | Country | Bytes |
|------|---------|-------|
| US | United States | `[0x55, 0x53]` |
| KR | South Korea | `[0x4B, 0x52]` |
| SG | Singapore | `[0x53, 0x47]` |

The `country_list` supports up to 10 countries per proof. Unused slots are zero-padded. The `country_list_length` field specifies how many entries are valid.

### Dependencies

Same as CIP-1: `keccak256` v0.1.1 + `coinbase_libs`

### Proof System

Same as CIP-1: Noir + Barretenberg UltraHonk + mopro for mobile

## Verification

### On-Chain Verifiers

| Chain | Address | Status |
|-------|---------|--------|
| Base Mainnet | `0xF3D5A09d2C85B28C52EF2905c1BE3a852b609D0C` | Deployed |
| Base Sepolia | `0xD0F3eE648386B59B484157332E736388Fcc41F47` | Deployed |

### SDK Integration

```typescript
import { ProofportSDK } from '@proofport/sdk';

const sdk = new ProofportSDK({ defaultCallbackUrl: 'https://myapp.com/verify' });

// Inclusion: prove user is from US, KR, or SG
const request = sdk.createCoinbaseCountryRequest({
  scope: 'myapp.xyz',
  countries: ['US', 'KR', 'SG'],
  mode: 'inclusion'
});

// Exclusion: prove user is NOT from sanctioned countries
const request = sdk.createCoinbaseCountryRequest({
  scope: 'myapp.xyz',
  countries: ['KP', 'IR', 'CU'],
  mode: 'exclusion'
});
```

## Security & Privacy

### Privacy Guarantees

| Data | Visible to Verifier? |
|------|---------------------|
| User's Ethereum address | ❌ Hidden |
| User's actual country | ❌ Hidden |
| Match result (in list / not in list) | ✅ Proved via valid proof |
| Country list checked against | ✅ Public (defined by dApp) |
| Nullifier | ✅ Public (sybil resistance) |

**Key privacy property**: The verifier learns that the user's country matches (or doesn't match) the provided list, but never learns which specific country the user is from.

### Security Assumptions

Same as CIP-1, plus:
- Coinbase country attestation accurately reflects the user's verified country
- ISO 3166-1 country codes are correctly encoded in attestation calldata

### Audit Status

- [x] Internal review
- [ ] External audit
- [ ] Formal verification

## License

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
