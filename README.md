# Bond Settlement DAML — DVP on Canton

> A production-quality reference implementation of DTCC-style bond clearing and
> settlement workflows built on DAML and the Canton distributed ledger.

**Author:** James Tsetsekas  
**Stack:** DAML 2.x · Canton · Haskell-like functional smart contracts  
**Domain:** Fixed-income clearing, DVP settlement, CCP novation, trade reporting

---

## Overview

The Depository Trust & Clearing Corporation (DTCC) is the backbone of U.S.
capital markets — processing over $2 quadrillion in securities annually through
its two principal subsidiaries:

| Entity | Role |
|--------|------|
| **NSCC** (National Securities Clearing Corporation) | Central counterparty (CCP): nets trades, guarantees settlement |
| **DTC** (Depository Trust Company) | Central securities depository (CSD): holds securities, executes delivery |

Since 2022, DTCC has been migrating core clearing and settlement infrastructure
onto **Canton**, Digital Asset's privacy-preserving distributed ledger, using
**DAML** (Digital Asset Modeling Language) as the smart-contract layer. This
repository demonstrates deep familiarity with that architecture by implementing
the full bond trade lifecycle end-to-end.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                   CANTON DISTRIBUTED LEDGER                     │
│                                                                 │
│  ┌─────────────┐   Propose   ┌─────────────┐                    │
│  │   BUYER     │────Trade───▶│   SELLER    │                   │
│  │  (PIMCO)    │◀───Accept───│   (Citi)    │                   │
│  └──────┬──────┘             └──────┬──────┘                   │
│         │                           │                          │
│         └──────────┬────────────────┘                          │
│                    │ Accepted Trade                            │
│                    ▼                                           │
│            ┌───────────────┐                                   │
│            │  NSCC (CCP)   │◀── Novation: CCP interposes      │
│            │  Clears &     │    itself between both sides      │
│            │  Guarantees   │                                   │
│            └───────┬───────┘                                   │
│                    │ Settlement Instruction                    │
│                    ▼                                           │
│   ┌────────────────────────────────────────────┐               │
│   │            DVP ENGINE (Atomic)             │               │
│   │                                            │               │
│   │  ┌────────────┐        ┌────────────────┐  │               │
│   │  │  DELIVERY  │        │    PAYMENT     │  │               │
│   │  │    LEG     │        │      LEG       │  │               │
│   │  │            │        │                │  │               │
│   │  │ DTC moves  │  ════  │ BNY Mellon     │  │               │
│   │  │ bond from  │ Atomic │ debits buyer,  │  │               │
│   │  │ Citi →     │        │ credits Citi   │  │               │
│   │  │ PIMCO      │        │                │  │               │
│   │  └────────────┘        └────────────────┘  │               │
│   └────────────────────────────────────────────┘               │
│                                                                │
│  ┌─────────────┐   Observes all contracts                      │
│  │  REGULATOR  │◀──────────────────────────────────────────    │
│  │ (SEC/FINRA) │   Trade reports → DTCC GTR                     │
│  └─────────────┘                                                │
└─────────────────────────────────────────────────────────────────┘
```

---

## Project Structure

```
bond-settlement-daml/
├── daml.yaml                    # DAML project config (SDK 2.9.3)
├── README.md
├── .gitignore
└── daml/
    ├── Bond.daml                # Bond instrument + issuance workflow
    ├── CashAccount.daml         # Cash/payment ledger
    ├── Trade.daml               # Trade matching & CCP novation
    ├── Settlement.daml          # DVP settlement engine
    ├── Compliance.daml          # KYC, trade reporting, position limits
    └── Test/
        ├── TestBondIssuance.daml   # Bond issuance lifecycle tests
        ├── TestSettlement.daml     # Full DVP settlement flow + failure path
        └── TestCompliance.daml    # KYC, GTR reporting, position limit tests
```

---

## Real-World Party Mapping

| DAML Party | Real-World Entity | Role |
|------------|-------------------|------|
| `issuer` | Goldman Sachs / JPMorgan | Bond originator |
| `seller` | Citi / dealer bank | Selling counterparty |
| `buyer` | PIMCO / BlackRock | Buying counterparty |
| `ccp` (NSCC) | DTCC/NSCC | Clears, novates, guarantees settlement |
| `depository` (DTC) | DTCC/DTC | Holds securities, executes delivery |
| `buyerBank` / `sellerBank` | BNY Mellon / State Street | Custodian & settlement banks |
| `regulator` | SEC / FINRA | Supervisory observer |
| `tradeRepository` | DTCC GTR | Trade reporting repository |

---

## DAML/Canton Concepts Demonstrated

### 1. Multi-Party Authorization
DAML's `signatory` / `observer` model enforces that every state change
requires consent from all relevant parties. The `ExecuteSettlement` choice
requires the CCP alone — but the Settlement contract itself was co-created by
all parties, so their consent was captured at creation time.

### 2. Atomic DVP via Single-Transaction Composition
`ExecuteSettlement` exercises four choices in a single DAML transaction:
- `Bond.UpdateHolder` — transfer security
- `CashAccount.Debit` (buyer) — debit payment
- `CashAccount.Credit` (seller) — credit payment
- `Trade.MarkSettled` — close trade

All succeed or all fail. No partial settlement is possible — this is the
fundamental guarantee that DVP provides.

### 3. Privacy via Sub-Transaction Projection
Canton's unique privacy model means Citi's `CashAccount` balance is visible
only to Citi and its bank — not to PIMCO or the regulator — even though the
settlement is a single atomic transaction. DAML's `observer` declarations
control exactly which parties see which contracts.

### 4. CCP Novation
After `AcceptTrade`, the `CCPNovation` template models NSCC interposing itself
as buyer-to-Citi and seller-to-PIMCO. This is the legal and operational
mechanism that makes DTCC's settlement guarantee possible.

### 5. Continuous Net Settlement (CNS) Model
The `NetSettlement` template aggregates multiple bilateral obligations into a
single net position per participant per security — the core of DTCC's CNS
algorithm that reduces settlement fails and systemic risk.

### 6. Regulatory Observer Pattern
All material contracts list the `regulator` party as an observer. In Canton,
this means the SEC/FINRA node receives a projected copy of every relevant
contract without being a signatory — read access without write authority.

### 7. Position Limit Enforcement
`Position.CheckPositionLimit` is a `nonconsuming` choice — it queries state
without modifying it, enabling pre-trade compliance checks that don't generate
unnecessary ledger events.

---

## Trade Lifecycle State Machine

```
                  ┌──────────┐
     (buyer)      │ Proposed │
  createCmd Trade └────┬─────┘
                       │
               seller: AcceptTrade / RejectTrade
                   ┌───┴────────┐
                   ▼            ▼
             ┌──────────┐  ┌──────────┐
             │ Accepted │  │ Rejected │
             └────┬─────┘  └──────────┘
                  │
           ccp: ExecuteSettlement
                  │
                  ▼
            ┌──────────┐
            │ Settled  │
            └──────────┘
```

---

## Settlement Instruction State Machine

```
  ┌────────────────────────────────────────┐
  │        Settlement Created              │
  │   deliveryStatus = Pending             │
  │   paymentStatus  = Pending             │
  └──────────────┬─────────────────────────┘
                 │
    ccp+depository: ValidateDelivery
    ccp+buyerBank:  ValidatePayment
                 │
  ┌──────────────▼─────────────────────────┐
  │   Both Legs Validated                  │
  │   deliveryStatus = Validated           │
  │   paymentStatus  = Validated           │
  └──────────────┬─────────────────────────┘
                 │
           ccp: ExecuteSettlement
                 │
     ┌───────────┴───────────┐
     ▼                       ▼
  Bond transferred      Cash transferred
  seller → buyer        buyer → seller
     └───────────┬───────────┘
                 ▼
           Trade.MarkSettled
```

---

## Build & Test

### Prerequisites

Install the DAML SDK (requires macOS or Linux):

```bash
curl -sSL https://get.daml.com/ | sh
# Add ~/.daml/bin to your PATH
export PATH="$HOME/.daml/bin:$PATH"
```

Verify installation:

```bash
daml version
# Expected: SDK version: 2.9.3 (or later 2.x)
```

### Build

```bash
cd bond-settlement-daml
daml build
# Produces: .daml/dist/bond-settlement-daml-1.0.0.dar
```

### Run Tests

```bash
daml test
# Runs all Test.* scripts and prints pass/fail per scenario
```

To run a specific test module:

```bash
daml test --files daml/Test/TestSettlement.daml
```

### Start the Canton Sandbox (local development)

```bash
daml start
# Starts Canton sandbox + JSON API at http://localhost:7575
# Navigator UI at http://localhost:7500
```

---

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| CCP as sole `controller` for `ExecuteSettlement` | Mirrors NSCC's role as the single settlement agent; no bilateral coordination required post-validation |
| `nonconsuming` for `CheckPositionLimit` | Pre-trade checks must not consume the position contract; DAML's nonconsuming pattern is ideal |
| Separate `ValidateDelivery` + `ValidatePayment` choices | Reflects that DTC (securities) and the Fed/bank (cash) operate on independent rails; each leg can fail independently before DVP |
| `NetSettlement` template | Demonstrates awareness of CNS netting — a core DTCC value proposition that reduces gross settlement volume by ~98% |
| `BondIssuanceProposal` two-step | Clean separation of issuer intent vs. depository registration — matching T+0 issuance workflow on digital asset platforms |

---

## Related DTCC Initiatives

This codebase directly maps to DTCC's live production work:

- **Project Lithium** — DTCC's internal Canton-based digital asset settlement platform
- **DTCC Digital Launchpad** — Sandbox environment for member firms to test digital asset settlement
- **Tokenized Collateral Network** — Using DLT for real-time collateral mobility (settlement finality T+0)
- **DTCC GTR** — Global trade repository, modelled in `Compliance.daml`

---

## License

MIT — educational reference implementation.  
Not affiliated with or endorsed by DTCC or Digital Asset Holdings.

## Future Enhancements

- **Multilateral netting**: Extend NetSettlement to support CNS-style batch netting across multiple counterparties
- **Corporate actions**: Model dividend payments, stock splits, and redemptions on bond instruments
- **Cross-chain interop**: Bridge Canton settlements with on-chain Ethereum token transfers
- **Regulatory reporting**: Automated DTCC GTR trade report generation from settlement events
