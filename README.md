# ZKProofport Circuit Improvement Proposals (CIPs)

**CIPs** define circuit standards for zero-knowledge proofs within the ZKProofport ecosystem, just as [EIPs](https://eips.ethereum.org/) standardize Ethereum itself.

Anyone can propose a circuit. Once reviewed and accepted, it becomes part of the ZKProofport circuit registry and is available to all dApps and agents via the SDK.

## What is a CIP?

A CIP is a design document describing a new ZK proof circuit for the ZKProofport infrastructure. Each CIP specifies:

- **What** is being proved (e.g., KYC status, country, asset ownership)
- **How** it is proved (circuit logic, public/private inputs, constraints)
- **Where** it is verified (on-chain verifier addresses, supported chains)
- **Security & privacy** guarantees

## CIP Process

```
Draft → Review → Final → (Deprecated)
```

| Status | Meaning |
|--------|---------|
| **Draft** | Initial proposal, open for feedback |
| **Review** | Reviewed by core team, circuit implemented and under audit |
| **Final** | Audited, deployed, and available in the circuit registry |
| **Deprecated** | Superseded or no longer maintained |

See [CIP-0](CIPS/cip-0.md) for the full process definition.

## Categories

| Category | Description |
|----------|-------------|
| **Identity** | Prove who you are without revealing personal data |
| **Credit** | Prove financial standing without revealing balances |
| **Qualification** | Prove credentials, affiliations, or achievements |
| **Social** | Prove social reputation or relationships |
| **Agent** | Enable AI agent identity and capabilities |
| **Meta** | Improvements to the CIP process itself |

## Current CIPs

| CIP | Title | Category | Status |
|-----|-------|----------|--------|
| [CIP-0](CIPS/cip-0.md) | CIP Process | Meta | Final |
| [CIP-1](CIPS/cip-1.md) | Coinbase KYC Attestation | Identity | Review |
| [CIP-2](CIPS/cip-2.md) | Coinbase Country Attestation | Identity | Review |

## Contributing

1. Read [CIP-0](CIPS/cip-0.md) for the process
2. Copy [cip-template.md](cip-template.md)
3. Open a PR with your proposal in the `CIPS/` directory

## License

All CIPs are released under [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
