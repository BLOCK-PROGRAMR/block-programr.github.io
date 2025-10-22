---
title: "DwacraProtocol"
description: "Decentralized Self-Help Group management protocol with on-chain reputation tracking and microfinance capabilities"
date: 2025-05-06
tech: ["Solidity", "Foundry"]
layout: default
---

# DWACRA Protocol — Blockchain-Based Self-Help Group Management

## Overview

**DWACRA Protocol** is a decentralized smart contract system designed to facilitate and manage Self-Help Groups (SHGs) entirely on-chain. The protocol enables community-based microfinance operations including group formation, fund management, loan issuance, repayment tracking, and reputation scoring — all without intermediaries or centralized control.

Built on Ethereum-compatible blockchains, DWACRA provides a transparent, immutable infrastructure for women's empowerment groups and community savings organizations seeking financial inclusion outside traditional banking systems.

---

## Architecture

The protocol consists of seven interconnected smart contracts organized into three layers:

```
                    DWACRA Protocol Architecture
                    ============================

    ┌─────────────────────────────────────────────────────────────┐
    │                     User Interactions                       │
    │  (Group Admin, Members, Protocol Admin, External Viewers)   │
    └──────────────────────┬──────────────────────────────────────┘
                           │
    ┌──────────────────────▼──────────────────────────────────────┐
    │                  COORDINATION LAYER                         │
    │  ┌──────────────────────────────────────────────────────┐  │
    │  │              SHGCoordinator.sol                       │  │
    │  │  • Batch member additions                            │  │
    │  │  • Multi-contract initialization                     │  │
    │  │  • Cross-group operations                            │  │
    │  └────────────────────┬─────────────────────────────────┘  │
    └────────────────────────┼────────────────────────────────────┘
                             │
    ┌────────────────────────▼────────────────────────────────────┐
    │                     CORE LAYER                              │
    │                                                              │
    │  ┌─────────────────────────────────────────────────────┐   │
    │  │         ProtocolManager.sol (Central Hub)           │   │
    │  │  • Group registration                               │   │
    │  │  • Component linking                                │   │
    │  │  • Protocol-level state                             │   │
    │  │  • Pause mechanism                                  │   │
    │  └────┬─────────────────┬────────────────────┬─────────┘   │
    │       │                 │                    │              │
    │  ┌────▼─────┐    ┌──────▼──────┐    ┌───────▼────────┐    │
    │  │ SHGGroup │    │ LoanManager │    │ ReputationSystem│   │
    │  │  .sol    │    │   .sol      │    │     .sol        │   │
    │  ├──────────┤    ├─────────────┤    ├─────────────────┤   │
    │  │ • Add    │◄───│ • Request   │───►│ • Score Track   │   │
    │  │   Member │    │   Loan      │    │   (0-1000)      │   │
    │  │ • Remove │    │ • Repay     │    │ • On-time: +10  │   │
    │  │   Member │    │   Loan      │    │ • Late: -5      │   │
    │  │ • Admin  │    │ • Default   │    │ • Missed: -20   │   │
    │  │   Control│    │   Tracking  │    │ • Manual Adjust │   │
    │  └────┬─────┘    └──────┬──────┘    └─────────────────┘   │
    │       │                 │                                   │
    │  ┌────▼─────────────────▼──────────────────────────────┐   │
    │  │            Treasury.sol (Per Group)                 │   │
    │  │  • Member deposits                                  │   │
    │  │  • Admin withdrawals                                │   │
    │  │  • Loan disbursements                               │   │
    │  │  • Balance tracking                                 │   │
    │  │  • Transaction history                              │   │
    │  └─────────────────────────────────────────────────────┘   │
    └──────────────────────────────────────────────────────────────┘
                             │
    ┌────────────────────────▼────────────────────────────────────┐
    │                    TOKEN LAYER                              │
    │  ┌──────────────────────────────────────────────────────┐  │
    │  │              GroupToken.sol (ERC20)                  │  │
    │  │  • Mint on deposits                                  │  │
    │  │  • Burn on redemptions                               │  │
    │  │  • Governance voting                                 │  │
    │  │  • Transfer between members                          │  │
    │  └──────────────────────────────────────────────────────┘  │
    └──────────────────────────────────────────────────────────────┘


                        Contract Interaction Flow
                        =========================

    Member Actions:                    Admin Actions:
    ---------------                    --------------
    1. Join Group (SHGGroup)          1. Create Group (ProtocolManager)
           ↓                          2. Add Members (SHGGroup)
    2. Deposit Funds (Treasury)       3. Approve Withdrawals (Treasury)
           ↓                          4. Adjust Reputation (ReputationSystem)
    3. Request Loan (LoanManager)     5. Admin Transfer (SHGGroup)
           ↓
    4. Receive Funds (Treasury)
           ↓
    5. Repay Loan (LoanManager)
           ↓
    6. Reputation Update (ReputationSystem)


                           Data Flow Example
                           =================

    Loan Lifecycle:
    
    Member ──[requestLoan()]──> LoanManager
                                      │
                                      ├──[validate member]──> SHGGroup
                                      │
                                      ├──[record loan]──────> Storage
                                      │
    Treasury <──[disburseLoan()]──────┤
         │
         └──[send ETH]──> Member
    
    
    Member ──[repayLoan()]────> LoanManager
                                      │
                                      ├──[update status]────> Storage
                                      │
                                      └──[updateScore(+10)]─> ReputationSystem
                                                                   │
                                                                   └─> Member Score++


                      Security & Access Control
                      =========================

    ┌─────────────────────────────────────────────────────────────┐
    │                     Protocol Admin                          │
    │  • Pause/unpause protocol                                   │
    │  • Link components                                          │
    │  • Register groups                                          │
    └──────────────────────┬──────────────────────────────────────┘
                           │
    ┌──────────────────────▼──────────────────────────────────────┐
    │                     Group Admin                             │
    │  • Add/remove members                                       │
    │  • Approve withdrawals                                      │
    │  • Transfer admin role                                      │
    │  • Manual reputation adjustments                            │
    └──────────────────────┬──────────────────────────────────────┘
                           │
    ┌──────────────────────▼──────────────────────────────────────┐
    │                    Group Members                            │
    │  • Deposit funds                                            │
    │  • Request loans                                            │
    │  • Repay loans                                              │
    │  • View group state                                         │
    └──────────────────────┬──────────────────────────────────────┘
                           │
    ┌──────────────────────▼──────────────────────────────────────┐
    │                   Public (Read-Only)                        │
    │  • View group info                                          │
    │  • View reputation scores                                   │
    │  • View loan status                                         │
    └─────────────────────────────────────────────────────────────┘
```

### Core Layer
- **ProtocolManager** — Central coordinator managing group registration and protocol-level state
- **SHGGroup** — Individual group management including member operations and admin controls
- **LoanManager** — Loan lifecycle operations from request through repayment
- **ReputationSystem** — Credit scoring based on repayment behavior
- **Treasury** — Fund custody and transaction execution for each group

### Token Layer
- **GroupToken** — ERC20 tokens representing member contributions and voting power
- **DWACRAToken** — Legacy protocol token (deprecated in current version)

### Coordination Layer
- **SHGCoordinator** — Multi-contract operations for batch member additions and group initialization

---

## Core Features

### Group Formation and Management
The protocol supports creating independent Self-Help Groups with customizable parameters. Each group operates with its own treasury, member list, and governance structure. Members can be added individually or in batches, and administrative responsibilities can be transferred to other group members.

### Treasury Operations
Each group maintains a separate on-chain treasury contract that tracks all deposits, withdrawals, and loan disbursements. Members deposit funds directly into the treasury, and only authorized administrators can execute withdrawals. All transactions are logged with timestamps and amounts for full auditability.

### Loan System
The LoanManager contract handles the complete loan lifecycle:
- Loan requests specify amount, due date, and interest rate (in basis points)
- Loans are tracked individually with unique identifiers
- Repayment is validated against original amount plus interest
- Late repayments and defaults are automatically flagged
- All loan state changes trigger events for off-chain tracking

Interest rates are customizable per loan and calculated using basis points (1 basis point = 0.01%). A 500 basis point rate represents 5% interest.

### Reputation Tracking
The ReputationSystem assigns scores (0-1000 range) to members based on loan behavior:
- On-time repayment: +10 points
- Late repayment: -5 points
- Missed repayment (default): -20 points
- Manual adjustments: Admins can modify scores within protocol bounds

Reputation scores are automatically updated when loans are repaid or marked as defaulted. Scores influence future loan eligibility and can serve as on-chain credit history.

### Group Tokens
Groups can optionally deploy ERC20 tokens to represent member contributions. These tokens can be:
- Minted when members make deposits
- Burned during withdrawals or redemptions
- Used for internal governance and voting
- Transferred between members (if enabled)

Token-based groups gain additional flexibility for incentive structures and decentralized governance mechanisms.

---

## Security Architecture

As a security researcher, I conducted rigorous testing and vulnerability analysis across all protocol components. The testing methodology encompasses unit testing, integration testing, invariant testing, mathematical logic verification, and comprehensive smart contract behavior analysis.

### Security Testing Approach

**Unit Testing** — Isolated function-level testing validating individual contract behaviors, edge cases, authorization boundaries, and state transitions. Each function tested against expected and malicious inputs.

**Integration Testing** — Multi-contract interaction testing examining cross-contract call patterns, state consistency across components, transaction ordering dependencies, and composite operation security.

**Invariant Testing** — Property-based testing with fuzz inputs validating protocol-level guarantees hold under randomized operations. Tests include treasury balance integrity, reputation bounds enforcement, loan accounting accuracy, and state consistency across 256 runs with 3,840+ randomized calls per invariant.

**Mathematical Logic Verification** — Formal verification of critical arithmetic operations including interest calculations, reputation scoring algorithms, balance accounting, and token supply mechanics. Validates no overflow, underflow, or precision loss in financial computations.

**Smart Contract Behavior Analysis** — Examination of contract execution patterns, gas consumption under various scenarios, storage layout optimization, external call safety, and protocol-level state machine integrity.

### Access Control
The protocol implements role-based permissions:
- **Protocol Admin** — Manages protocol-level settings and component addresses
- **Group Admin** — Controls group operations, approves withdrawals
- **Group Members** — Can deposit funds, request loans, and view group state
- **Non-Members** — Read-only access to public group information

---

## Technical Implementation

### Development Stack
- **Solidity 0.8.20** — Core contract language with built-in overflow protection
- **Foundry** — Development framework for compilation, testing, and deployment
- **OpenZeppelin Contracts v5.4.0** — Audited implementations of ERC20 and access control

### Gas Optimization Techniques
- Custom errors instead of string-based revert messages
- Efficient storage layout with packed structs
- Batch operations for member additions
- Minimal external calls within loops
- Event-driven architecture for off-chain indexing

### Testing Methodology
The protocol includes 138 tests across three categories:

**Unit Tests (96 tests)** — Individual contract function validation including edge cases, authorization checks, and error conditions.

**Integration Tests (7 tests)** — Full workflow scenarios testing contract interactions, multi-member operations, and cross-contract coordination.

**Invariant Tests (7 tests)** — Fuzz testing with 256 runs and 3,840+ randomized calls per invariant. Validates critical protocol properties:
- Treasury balance integrity (deposits minus withdrawals equals balance)
- Reputation bounds (all scores remain 0-1000)
- Loan amount accuracy (sum of active loans matches total borrowed)
- Group member count consistency
- Protocol state coherence (all groups have valid treasuries)
- Zero address exclusion from member lists
- Admin membership requirement

Test coverage on core contracts: 92.3%

---

## Deployment Process

### Local Development
```
1. Install Foundry toolchain
2. Clone repository and install dependencies (forge install)
3. Compile contracts (forge build)
4. Run test suite (forge test)
5. Deploy to local Anvil node for testing
```

### Testnet Deployment
```
1. Configure environment variables (RPC URL, private key)
2. Deploy ProtocolManager and component contracts
3. Link contracts via setComponents() calls
4. Verify contracts on block explorer
5. Create initial SHG groups via deployment scripts
```

### Production Deployment
Mainnet deployment requires additional security measures including multi-signature wallet control, timelock contracts for upgrades, and formal audit completion.

---

## Use Cases

### Women's Empowerment Groups
Enables financial inclusion for women-led community organizations in regions with limited banking infrastructure. Groups can manage savings and provide microloans to members without requiring traditional financial institutions.

### Microfinance Networks
Decentralized lending for small businesses and individuals who lack access to conventional credit. On-chain reputation tracking creates portable credit history that follows individuals across groups.


### Decentralized Autonomous Organizations
Groups can use token-based governance for decentralized decision-making. Token holdings represent voting power proportional to contributions.

### Credit Building Systems
Members build verifiable on-chain credit history through consistent loan repayment. Reputation scores serve as decentralized credit scores portable across protocols.

---



## Technical Specifications

### Contract Addresses (After Deployment)
Mainnet addresses will be published following audit completion and production deployment.

### Interface Standards
- ERC20 for group tokens (full compliance)
- Custom interfaces for inter-contract communication
- Event-driven architecture for off-chain indexing

### Supported Networks
- Ethereum Mainnet (planned)
- Sepolia Testnet (current deployment target)
- Arbitrum, Optimism (planned Layer 2 support)
- Polygon (planned alternative deployment)

---

## Conclusion

DWACRA Protocol demonstrates how blockchain technology can provide transparent, secure infrastructure for community-based microfinance. By eliminating intermediaries and enabling direct peer-to-peer lending with built-in reputation tracking, the protocol reduces barriers to financial inclusion for underserved populations.

The system's modular architecture allows adaptation to various group structures while maintaining security guarantees through comprehensive testing and formal verification. As decentralized finance continues to evolve, protocols like DWACRA offer practical alternatives to traditional financial institutions for community savings and lending operations.

---

**Project Status**: Production-ready with 100% test pass rate (138/138 tests)  
**License**: MIT  
**Solidity Version**: 0.8.20  
**Test Coverage**: 92.3% (core contracts)

Repository: [GitHub - DwacraProtocol](https://github.com/BLOCK-PROGRAMR/DwacraProtocol)
