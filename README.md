# Security Audit Portfolio

Smart contract and application security work by Nishant Kumar. Focus: Soroban/Stellar and Solidity/DeFi contracts, with a growing web/API security track.

## Audit Reports

| Target | Platform | Scope | Status | Report |
|--------|----------|-------|--------|--------|
| ChitChain | Self-audit (own protocol) | Soroban smart contracts | Findings fixed | [chitchain-audit-report.md](audits/chitchain-audit-report.md) |
| Union Finance | Public contest review | Interest accounting (`debtWriteOff`) | Judged invalid (by design) per a prior contest ruling | [union-audit-report.md](audits/union-audit-report.md) |
| Additional work on a live bug bounty program | Immunefi | Bridge/governance contracts | Details withheld per program rules | Not published |

## Methodology

- Manual code review: access control, authorization context, state and storage handling, interest and accounting logic
- Automated tools: Slither, Mythril
- Proof of concept and testing: Foundry (PoC exploits, fuzz and invariant tests)
- Report format: Title, Impact, Proof of Concept, Mitigation

## Writing

- [3 Real Bugs I Found: From CTF Flags to a Live Smart Contract Vulnerability](https://dev.to/nishant_singh_777/3-real-bugs-i-found-from-ctf-flags-to-a-live-smart-contract-vulnerability-3nj4) (dev.to)

## Contact

- GitHub: [Nishantk16](https://github.com/Nishantk16)
- LinkedIn: [Nishant Kumar](https://www.linkedin.com/in/nishant-singh-300774333/)
- dev.to: [nishant_singh_777](https://dev.to/nishant_singh_777)
