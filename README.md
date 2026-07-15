# HoodPerp Audits

Security review and remediation notes for HoodPerp smart contracts on Robinhood Chain.

## Reports

| Document | Description |
|---|---|
| [report.md](./report.md) | Original findings from Pyro's AI Audit Tool (commit `d9a4c97`) |
| [remediation.md](./remediation.md) | BuybackVault fixes — design, logic, and resolution status |

## How the audit went

1. **Scope** — HoodPerp protocol contracts: `Router`, `WithdrawRouter`, `BuybackVault`, and `UniversalRouterLib` (Robinhood Chain, chain 4663).

2. **Tooling** — [Pyro](https://x.com/0x3b33) ([@PhageSec](https://x.com/PhageSec) / [@sherlockdefi](https://x.com/sherlockdefi)) ran the codebase through an **AI-assisted audit pipeline** with light human oversight. Output is a structured report with severity-tagged findings and recommended diffs.

3. **Outcome** — 2 Medium, 8 Low, and 3 Informational items. The highest-risk cluster was **BuybackVault keeper economics** (uncapped spend + keeper-only price floor).

4. **Remediation** — The HoodPerp team implemented **BuybackVault hardening** (see [remediation.md](./remediation.md)): mandatory spend cap, on-chain HOOD/USDG floor rate, interval validation, and removal of broken Chainlink Automation hooks. Router / WithdrawRouter low-severity items remain tracked for a follow-up pass.

5. **Limitations** — This was not a full manual audit. Findings should be treated as leads; a professional human review and bug bounty are recommended before mainnet vault deployment with significant TVL.

## Auditor

**Pyro** — Co-founder @ PhageSec · Lead Security Researcher @ Sherlock  
100+ audits · 500+ High/Medium findings  
[GitHub](https://github.com/0x3b33) · [Telegram](https://t.me/Pyro3b33)

## Contract repo

Implementation lives in [HoodPerp/hoodperp](https://github.com/HoodPerp/hoodperp) under `contract/`.  
Deployed addresses: [docs.hoodperp.com/contracts](https://docs.hoodperp.com/contracts)
