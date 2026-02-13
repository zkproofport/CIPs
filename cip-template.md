---
cip: <CIP number>
title: <CIP title>
author: <name (@github)>
status: Draft
category: <Identity | Credit | Qualification | Social | Agent | Meta>
circuit-language: <Noir | Circom | other>
data-source: <e.g., EAS, Plaid, GitHub API>
created: <YYYY-MM-DD>
requires: <CIP number(s), optional>
---

## Abstract

A short (~200 word) description of the circuit and what it proves.

## Motivation

Why does this circuit need to exist? What problem does it solve? What use cases does it enable?

## Circuit Specification

### Public Inputs

| Index | Field | Type | Description |
|-------|-------|------|-------------|
| 0 | `signal_hash` | `[u8; 32]` | Anti-replay challenge from the dApp |
| ... | ... | ... | ... |

### Private Inputs

| Field | Type | Description |
|-------|------|-------------|
| ... | ... | ... |

### Nullifier

- **Derivation**: Describe how the nullifier is computed
- **Scope**: What scope is used for sybil resistance

### Constraint Count

- Estimated constraint count: `<number>`

### Logic Overview

Describe the circuit's verification steps at a high level.

## Verification

### On-Chain Verifiers

| Chain | Address | Status |
|-------|---------|--------|
| Base Sepolia | `0x...` | Deployed |

### Off-Chain Verification

Describe how off-chain verification works (e.g., `@aztec/bb.js`).

### SDK Integration

```typescript
// Example SDK usage
const result = await sdk.verifyOnChain('circuit_name', proof, publicInputs, provider);
```

## Security & Privacy

### Privacy Guarantees

What is hidden from the verifier?

### Security Assumptions

What trust assumptions does this circuit make?

### Audit Status

- [ ] Internal review
- [ ] External audit
- [ ] Formal verification

## License

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
