<p align="center">
  <img src="static/logo.jpg" alt="Auditware" width="130"/>
</p>

<h1 align="center">Auditware - Public Audit Portfolio</h1>

<p align="center">
  <a href="https://calendly.com/d/cnqr-49f-xdy/auditware-consultation"><img src="https://img.shields.io/badge/Book%20a%20Call-006BFF?style=flat&logo=googlecalendar&logoColor=white" alt="Book a Call"/></a>
  &nbsp;
  <a href="https://t.me/forefy_t"><img src="https://img.shields.io/badge/Telegram-Auditware-2CA5E0?style=flat&logo=telegram&logoColor=white" alt="Telegram"/></a>
</p>

Auditware covers Web3's full security surface - from smart contracts to the infrastructure, operations, and human factors surrounding them. A decade of expertise across every layer a protocol touches.

A large sum of our work isn't public, as clients decide what gets disclosed. What you'll find here is a representative slice of that work - but a good read that's for sure.

---

## What You Can Find Here

| Domain                   | Coverage                                                            |
| ------------------------ | ------------------------------------------------------------------- |
| **EVM Smart Contracts**  | Solidity, DeFi, bonding curves, AMMs, NFT protocols, privacy/ZK     |
| **Solana Programs**      | Anchor/Rust, token programs, NFT marketplaces                       |
| **Rust Smart Contracts** | CosmWasm, Solana native, L1 SDKs, consensus logic                   |
| **Web Apps**             | Wallet security, authentication, XSS/SSRF/LFI/SQLI                  |
| **Infrastructure**       | AWS/GCP/Azure, Terraform audit, CI/CD pipelines, container security |
| **Operational Security** | Endpoint security, key custody, org-level threat modeling           |

---

## Audit Portfolio

| Client                                                                                                                                                | Type                            | Scope                                                               | Notable Findings                                                                                                                     |
| ----------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| [0xbow - Privacy Pools Core](0xbow/Privacy%20Pools%20Core/Privacy%20Pools%20Core%20Audit%20Report.md)                                                 | Smart Contract                  | EVM privacy protocol (ZK proof-based private transfers)             | DoS via root history overwrite, insecure ERC20 approval                                                                              |
| [0xbow - Privacy Pools Batch Relayer](0xbow/Privacy%20Pools%20Batch%20Relayer/Privacy%20Pools%20Batch%20Relayer%20Audit%20Report.md)                  | Smart Contract                  | EVM relayer contract                                                | High - unrestricted processor address enables loss of funds; unvalidated pool parameter                                              |
| [0xbow - Seed Phrase Generation](0xbow/Privacy%20Pools%20Seed%20Phrase%20Generation/Privacy%20Pools%20Seed%20Phrase%20Generation%20Audit%20Report.md) | Web App · Cryptography          | Cryptographic entropy & key derivation (TypeScript SDK + frontend)  | High - authentication bypass via insecure EIP-712 implementation in seed derivation                                                  |
| [ShieldFlow](ShieldFlow/ShieldFlow%20Audit%20Report.md)                                                                                               | Smart Contract · Infrastructure | Privacy protocol fork · smart contracts · ASP · website · AWS infra | **4 Critical** - SSRF→secret extraction, 53-bit entropy truncation (full fund theft), overly permissive OIDC, account-wide IAM write |
| [Rarible](Rarible/Rarible%20Audit%20Report.md)                                                                                                        | Smart Contract                  | Solana NFT marketplace (3 Anchor programs)                          | **Critical** - loss of funds via premature order closure; 15 total findings                                                          |
| [Conduct Protocol](Conduct/Conduct%20Audit%20Report.md)                                                                                               | L1 Protocol                     | Bitcoin-staked L1 · Rust SDK · consensus · p2p                      | 7 High - secret key exposure, VRF chain selection flaws, integrity violations                                                        |
| [Slingshot](Slingshot/Slingshot%20Audit%20Report.md)                                                                                                  | Mobile App                      | Non-custodial mobile crypto wallet + threat model                   | Unauthenticated encryption, PIN brute-force, transaction auth bypass                                                                 |
| [Bullpen](Bullpen/Bullpen%20Audit%20Report.md)                                                                                                        | Web App                         | Telegram mini-app (TypeScript + Rust)                               | XSS → credential theft, missing CSRF protection                                                                                      |
| [Aura](Aura/Aura%20Audit%20Report.md)                                                                                                                 | Smart Contract                  | EVM bonding curve protocol                                          | Front-running on initial soul acquisition, access control gaps                                                                       |
| [ECF - Apps.fun](Ethereum%20Community%20Foundation/Apps.fun/Apps.fun%20Smart%20Contract%20Audit%20Report.md)                                          | Smart Contract                  | EVM AMM with virtual reserves → Uniswap V4 graduation               | High - poisoned position DoS; owner centralization risk                                                                              |
| [ECF - Beth](Ethereum%20Community%20Foundation/Beth/Beth%20Smart%20Contract%20Audit%20Report.md)                                                      | Smart Contract                  | ETH burn wrapper (ERC20)                                            | No significant issues found                                                                                                          |
| [ECF - Blobhouse](Ethereum%20Community%20Foundation/Blobhouse/Blobhouse%20Smart%20Contract%20Audit%20Report.md)                                       | Smart Contract                  | On-chain RPS wagering game (commit-reveal)                          | **Critical** - fund appropriation via unchecked forfeit winner (resolved)                                                            |
| [ECF - Blobkit](Ethereum%20Community%20Foundation/Blobkit/Blobkit%20Smart%20Contract%20Audit%20Report.md)                                             | Smart Contract · SDK            | EIP-4844 blob storage escrow + TypeScript SDK + proxy server        | Owner unlimited emergency withdrawal, zero blob tx hash acceptance                                                                   |
| [ECF - Swapboard](Ethereum%20Community%20Foundation/Swapboard/Swapboard%20Smart%20Contract%20Audit%20Report.md)                                       | Smart Contract · Web App        | Trustless OTC ERC20 swap orderbook + frontend                       | High - insufficient input validation (XSS); forced browsing                                                                          |
| [OpSec Review](OpSec/OpSec%20Review%20-%20Redacted.md) _(redacted)_                                                                                   | OpSec                           | Org-level operational security                                      | Endpoint, key custody, access management                                                                                             |
| [OpSec Audit](OpSec/OpSec%20Audit%20-%20Redacted.md) _(redacted)_                                                                                     | OpSec                           | Operational security audit                                          | Unmanaged endpoints, credential hygiene gaps, device segregation risks                                                               |

---

## Why Auditware

- **Full-stack coverage** - we audit the entire attack surface: contracts, SDKs, APIs, infrastructure, and human factors. Most firms stop at Solidity.
- **Privacy & ZK expertise** - trusted by 0xbow (Privacy Pools, the Vitalik Buterin-backed compliance primitive) and ShieldFlow for cryptographic correctness and protocol-level threat modeling.
- **Rust expertise** - deep experience auditing Rust smart contracts and protocol SDKs across Solana and custom L1s. Trusted by Rarible for their Solana NFT marketplace. We also built [Radar](https://github.com/Auditware/radar), an open-source static analysis tool for Rust smart contracts.
- **Open-source security tooling** - we built and maintain [W3OSC](https://github.com/W3OSC), the Web3 Open Source Standard org: [web3-opsec-standard](https://github.com/W3OSC/web3-opsec-standard) (the community Web3 OpSec standard), [depenemy](https://github.com/W3OSC/depenemy) (supply chain risk scanner), [multisigmonitor](https://github.com/W3OSC/multisigmonitor) (Safe multisig analysis & monitoring), [skill-warden](https://github.com/W3OSC/skill-warden) (AI skill security scanner), and more.
- **Security automation & monitoring** - we design and build in-house tooling tailored to your protocol: on-chain monitoring, alerting pipelines, automated threat detection, and custom security infrastructure.
- **Tailored development services** - beyond auditing, we develop security-first software for clients: SDKs, tooling, integrations, and hardened internal systems built with the same rigor we apply to audits.

---

## Learn more on how it works

<p align="center">
  <a href="https://calendly.com/d/cnqr-49f-xdy/auditware-consultation"><img src="https://img.shields.io/badge/Book%20a%20Call-Calendly-006BFF?style=for-the-badge&logo=googlecalendar&logoColor=white" alt="Book a Call"/></a>
  &nbsp;&nbsp;
  <a href="https://t.me/forefy_t"><img src="https://img.shields.io/badge/Telegram-Auditware-2CA5E0?style=for-the-badge&logo=telegram&logoColor=white" alt="Telegram"/></a>
</p>
