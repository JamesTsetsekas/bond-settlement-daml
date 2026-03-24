# Architecture

## Settlement Workflow

The system models a complete bond settlement lifecycle with the following parties:

```
                    ┌─────────────┐
                    │   Issuer    │
                    └──────┬──────┘
                           │ Issue Bond
                           ▼
                    ┌─────────────┐
                    │    Bond     │
                    └──────┬──────┘
                           │ Trade Proposal
                           ▼
              ┌────────────────────────┐
              │     Trade Matching     │
              │  Buyer <-> Seller      │
              └────────────┬───────────┘
                           │ Accept
                           ▼
              ┌────────────────────────┐
              │   CCP (DTCC/NSCC)      │
              │  Validates both legs   │
              └────────────┬───────────┘
                           │ Execute
                           ▼
              ┌────────────────────────┐
              │   Atomic Settlement    │
              │  Bond <-> Cash (DVP)   │
              └────────────────────────┘
```

## Party Mapping

| DAML Party | Real-World Entity | Role |
|-----------|-------------------|------|
| issuer | Bond issuer (corporation, government) | Creates bond instruments |
| buyer | Buy-side firm (asset manager, fund) | Purchases bonds |
| seller | Sell-side firm (broker-dealer) | Sells bonds |
| ccp | DTCC/NSCC | Central counterparty, guarantees settlement |
| regulator | SEC/FINRA | Observes transactions for compliance |
| bank | Custodian bank | Holds cash accounts |

## Canton Privacy Model

Each party only sees the contracts where they are a signatory or observer. The CCP sees all settlement contracts but individual trade details between buyer and seller remain private until settlement.

## Key Design Decisions

1. **Two-step bond issuance**: Issuer proposes, DTC accepts. Models real-world DTC eligibility.
2. **CCP novation**: Trade is novated through the CCP, mirroring NSCC's role in US markets.
3. **Atomic DVP**: Both cash and bond legs settle in a single Canton transaction, eliminating principal risk.
4. **Non-consuming compliance checks**: KYC and position limits checked without consuming the contract.
