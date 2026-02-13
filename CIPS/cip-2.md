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
| 64-95 | `scope` | `[u8; 32]` | `keccak256(scope_string)` for sybil resistance |
| 96-117 | `country_list` | `[[u8; 2]; 11]` | List of 2-byte ISO 3166-1 country codes (max 11) |
| 118 | `inclusion_mode` | `u8` | `1` = inclusion (must be in list), `0` = exclusion (must NOT be in list) |
| 119-150 | `nullifier` | `[u8; 32]` | `keccak256(user_address ‖ signal_hash ‖ scope)` |

### Private Inputs

Same as CIP-1, plus:

| Field | Type | Description |
|-------|------|-------------|
| `raw_transaction` | `[u8; 300]` | RLP-encoded Coinbase **country** attestation TX |

All CIP-1 private inputs apply (user_address, user_signature, user_pubkey, coinbase_attester_pubkey, merkle_proof).

### Nullifier

Same derivation as CIP-1: `keccak256(user_address ‖ signal_hash ‖ scope)`

### Verification Logic

```
PARTS 1-2: Same as CIP-1 (User Ownership + Attestation Authenticity)

PART 3: Country Verification
├─ Verify TX calls attestCountry(uint256) (selector: 0x0a225248)
├─ Extract 2-byte country code from calldata
├─ Verify calldata address matches user_address
├─ If inclusion_mode == 1:
│   └─ Assert country IS in country_list
└─ If inclusion_mode == 0:
    └─ Assert country is NOT in country_list

PART 4: Sybil Resistance (same as CIP-1)
```

### Country Code Encoding

Country codes use ISO 3166-1 alpha-2 encoding (2 ASCII bytes):

| Code | Country | Bytes |
|------|---------|-------|
| US | United States | `[0x55, 0x53]` |
| KR | South Korea | `[0x4B, 0x52]` |
| SG | Singapore | `[0x53, 0x47]` |

The `country_list` supports up to 11 countries per proof. Unused slots are zero-padded.

### Dependencies

Same as CIP-1: `keccak256` v0.1.1 + `coinbase_libs`

### Proof System

Same as CIP-1: Noir + Barretenberg UltraHonk + mopro for mobile

## Verification

### On-Chain Verifiers

| Chain | Address | Status |
|-------|---------|--------|
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
