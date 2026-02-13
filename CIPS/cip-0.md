---
cip: 0
title: CIP Process
author: ZKProofport (@zkproofport)
status: Final
category: Meta
circuit-language: N/A
data-source: N/A
created: 2026-02-13
---

## Abstract

CIP-0 defines the Circuit Improvement Proposal process for ZKProofport. It describes how new ZK circuits are proposed, reviewed, and published to the circuit registry.

## Motivation

ZKProofport's value comes from its extensible circuit architecture. As the ecosystem grows, a clear and open process is needed to:

1. Allow anyone to propose new proof circuits
2. Ensure circuits meet security and privacy standards before deployment
3. Maintain a canonical registry of available circuits
4. Enable composability across circuits

Like EIPs for Ethereum, CIPs create a transparent, community-driven standards process for ZK privacy circuits.

## Specification

### CIP Types

| Category | Description | Examples |
|----------|-------------|---------|
| **Identity** | Prove who you are | KYC, Country, Age |
| **Credit** | Prove financial standing | DeFi holdings, RWA ownership |
| **Qualification** | Prove credentials | GitHub contributions, email domain |
| **Social** | Prove social reputation | Farcaster followers, X followers |
| **Agent** | Enable AI agent capabilities | Agent identity, capability proof |
| **Meta** | CIP process improvements | This document |

### CIP Status

```
Draft → Review → Final → (Deprecated)
```

- **Draft**: Author has submitted a proposal. Open for community feedback via GitHub Issues.
- **Review**: Core team has reviewed and accepted the proposal. Circuit is implemented and undergoing security audit.
- **Final**: Circuit has been audited, deployed on-chain, and added to the ZKProofport circuit registry. Available via SDK.
- **Deprecated**: Circuit has been superseded or is no longer maintained. Existing proofs remain valid.

### CIP Format

Every CIP must include:

1. **YAML Preamble**: CIP number, title, author, status, category, circuit language, data source, creation date
2. **Abstract**: Short description of what the circuit proves
3. **Motivation**: Why this circuit needs to exist
4. **Circuit Specification**: Public inputs, private inputs, nullifier derivation, constraint count, logic overview
5. **Verification**: On-chain verifier addresses, off-chain verification method, SDK integration example
6. **Security & Privacy**: Privacy guarantees, security assumptions, audit status

### Submitting a CIP

1. Fork this repository
2. Copy `cip-template.md` to `CIPS/cip-<number>.md`
3. Fill in the template
4. Submit a Pull Request
5. Engage with feedback on the PR

### CIP Numbering

CIP numbers are assigned sequentially. The author should use the next available number.

## License

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
