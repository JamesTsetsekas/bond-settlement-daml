# Contributing

Thanks for your interest in contributing to the Bond Settlement DAML project.

## Prerequisites

- [DAML SDK](https://docs.daml.com/getting-started/installation.html) (v2.x or v3.x)
- Familiarity with DAML/Haskell-like syntax

## Getting Started

1. Fork the repository
2. Clone your fork
3. Install DAML SDK: `curl -sSL https://get.daml.com/ | sh`
4. Build: `daml build`
5. Test: `daml test`

## Project Structure

```
daml/
  Bond.daml           - Bond instrument templates
  Trade.daml          - Trade matching and lifecycle
  Settlement.daml     - DVP settlement with CCP
  CashAccount.daml    - Cash/payment ledger
  Compliance.daml     - KYC and regulatory compliance
  Test/               - Test scripts
```

## Code Style

- Follow DAML best practices for template design
- Use descriptive choice names that reflect business operations
- Add comments explaining the financial workflow being modeled
- Write test scripts for all new templates

## Pull Requests

1. Create a feature branch from `master`
2. Ensure `daml build` and `daml test` pass
3. Update README for any new templates or workflows
