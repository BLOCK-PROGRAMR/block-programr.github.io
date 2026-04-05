---
title: "कवच KavachPay — Private Payroll on Solana"
description: "Private payroll on Solana with TEE-backed salary confidentiality and automatic tax deduction"
date: 2026-03-22
tech: ["Solana", "Anchor 0.32.1", "MagicBlock Private ER", "Intel TDX", "Next.js 14", "TypeScript", "TailwindCSS"]
layout: default
---

# कवच KavachPay — Private Payroll on Solana

> **Your salary. Nobody else's business.**

KavachPay is the first private payroll protocol on Solana with automatic tax deduction. Salary amounts are encrypted inside MagicBlock's Private Ephemeral Rollup TEE (Intel TDX hardware). Solana Explorer shows one settlement transaction. Nothing else. Ever.

---

## The Problem

Every payment on Solana is 100% public.

When your company pays your salary in USDC — your colleague can search your wallet on Solana Explorer and see exactly how much you earn. Your competitors can see every vendor payment. Your employees can see each other's salaries.

This is why enterprises refuse to use Solana for real business payments.

**KavachPay fixes this.**

---

## What KavachPay Does

```text
Employer enters: gross salary for 5 employees

Inside Private ER TEE (Intel TDX):
  -> TDS (10%) calculated privately
  -> PF (12%) calculated privately
  -> Net salary calculated privately
  -> Three private transfers executed

On Solana Explorer:
  -> One settlement transaction
  -> Amount: [hidden]
  -> Recipients: [hidden]
  -> Memo: [hidden]
  -> Nothing else
```

**The innovation:** Automatic TDS + PF deduction computed inside TEE. No crypto payroll app does this anywhere. Not on Solana, not on Ethereum, not on Aleo.

---

## MagicBlock Technologies Used

| Technology | How KavachPay Uses It |
|---|---|
| **Private Ephemeral Rollup (TEE)** | Salary amounts + memos encrypted in Intel TDX hardware. Nobody reads this — not MagicBlock, not validators, not anyone |
| **Ephemeral Rollup** | Real-time payroll session at 10ms block time. Batch execution of all employee payments |
| **Permission Program** | Employer gets AUTHORITY + TX_LOGS. Each employee gets TX_BALANCES for their account only. Auditor gets temporary TX_LOGS access |
| **Magic Router** | Single RPC endpoint auto-routes delegated accounts to ER, undelegated to Solana |
| **Magic Actions** | Automatic settlement triggered on ER commit. Payroll finalizes without manual intervention |

**Remove Private ER** -> every salary becomes public on Solana Explorer -> product has zero value. Private ER is not optional. It IS the product.

---

## How It Works — Step by Step

### Employer Flow

1. **Add employees** — wallet addresses + gross salary + country (India/US/UK)
2. **Review tax breakdown** — TEE calculates TDS, PF, net salary for each employee
3. **Execute payroll** — one click sends entire batch into Private ER TEE
4. **Settlement** — Magic Action automatically records one transaction on Solana

### What happens inside the TEE

```text
For each employee (e.g. India):
  Gross:  1,00,000 USDC
  TDS:    - 10,000 USDC -> Tax wallet   (private)
  PF:     - 12,000 USDC -> PF wallet    (private)
  Net:      78,000 USDC -> Employee     (private)

All three transfers execute inside Intel TDX.
None of these amounts ever touch Solana mainnet.
Only the final settlement proof is recorded on-chain.
```

### Employee Flow

1. **Open Inbox** — sees "Payment received — tap to reveal"
2. **Reveal salary** — signs with Phantom wallet
3. **TEE verifies** — confirms wallet is the correct recipient
4. **Decrypts** — shows their net salary, gross, and deduction breakdown
5. **Download payslip** — private PDF with TEE attestation hash
6. **Withdraw** — USDC transfers to main wallet

---

## The Privacy Proof

This is what Solana Explorer shows after a full payroll batch for 5 employees:

```text
Transaction: 5dKm8Nu2mdoa6PlH9nBKFeF...
Timestamp:   Mar 22 2026 · 11:38:37 UTC
Amount:      [hidden]
Recipients:  [hidden]
Memo:        [hidden]
Salaries:    [hidden]
```

**That is the entire public record. Nothing else is visible. Ever.**

---

## Automatic Tax Deduction — The New Innovation

No existing crypto payroll app calculates and splits tax automatically inside a TEE.

| Country | TDS / Income Tax | PF / Social Security |
|---|---|---|
| India | 10% TDS | 12% PF |
| United States | 22% Federal | — |
| United Kingdom | 20% Income Tax | 12% NI |

Three private transfers per employee:
1. Net salary -> employee wallet (private)
2. Tax -> government wallet (private)
3. PF/Social -> PF wallet (private)

All computed and executed inside Private ER TEE. No amounts ever visible on-chain.

---

## Architecture

```text
┌─────────────────────────────────────────────────────┐
│                    EMPLOYER                          │
│         Adds employees + gross salaries              │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│              SOLANA MAINNET                          │
│    init_payment() — locks USDC in escrow PDA         │
│    delegate_payroll() — sends to Private ER          │
└──────────────────────┬──────────────────────────────┘
                       │ delegation
                       ▼
┌─────────────────────────────────────────────────────┐
│         MAGICBLOCK PRIVATE ER (Intel TDX)            │
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │           TRUSTED EXECUTION ENVIRONMENT       │  │
│  │                                              │  │
│  │  • Receives encrypted salary amounts         │  │
│  │  • Calculates TDS + PF privately             │  │
│  │  • Executes three transfers per employee     │  │
│  │  • Permission Program enforces access        │  │
│  │  • Hardware attestation via Intel TDX        │  │
│  │                                              │  │
│  │  NOBODY CAN READ THIS — not even MagicBlock  │  │
│  └──────────────────────────────────────────────┘  │
│                                                     │
│    Validator: FnE6VJT5QNZdedZPnCoLsARgBwoE6DeJNjBs  │
└──────────────────────┬──────────────────────────────┘
                       │ commit + Magic Action
                       ▼
┌─────────────────────────────────────────────────────┐
│              SOLANA MAINNET                          │
│    finalize_payroll() — Magic Action auto-triggers   │
│    One settlement transaction recorded               │
│    Amount: hidden · Memo: hidden · Recipients: hidden│
└─────────────────────────────────────────────────────┘
```

---

## Anchor Program

Four instructions deployed on Solana Devnet:

| Instruction | Layer | Purpose |
|---|---|---|
| `init_payment` | Mainnet | Locks USDC in escrow PDA |
| `send_private` | Private ER | Stores encrypted PaymentRecord in TEE |
| `claim_payment` | Private ER | TEE decrypts for authenticated recipient |
| `settle` | Mainnet | USDC transfer, emits zero-detail Settlement event |

---

## Tech Stack

**Blockchain**
- Solana (Devnet)
- Anchor 0.32.1
- MagicBlock Private Ephemeral Rollups SDK
- MagicBlock Ephemeral Rollups Kit

**Frontend**
- Next.js 14 (App Router)
- TypeScript
- TailwindCSS
- Solana Wallet Adapter (Phantom)

**Privacy Infrastructure**
- MagicBlock Private SPL API (Beta)
- Intel TDX Trusted Execution Environment
- MagicBlock Permission Program
- MagicBlock Magic Router
- MagicBlock Magic Actions

---

## Local Development

### Prerequisites

```bash
solana --version    # 2.3.13
anchor --version    # 0.32.1
node --version      # 24.10.0
rust --version      # 1.85.0
```

### Setup

```bash
# Clone the repo
git clone https://github.com/BLOCK-PROGRAMR/kavachpay
cd kavachpay

# Install Anchor dependencies
npm install

# Install frontend dependencies
cd app && npm install

# Set up environment
cp app/.env.example app/.env.local
```

### Run

```bash
cd app && npm run dev
# -> http://localhost:3000
```

---

## Compliance

KavachPay is **not a mixer and not anonymous rails.**

- Both sender and receiver are identifiable via wallet addresses
- Only the payment **payload** is private (amount + memo)
- Built-in OFAC + sanctions screening via MagicBlock's node-level compliance
- Identical to how traditional banking works — banks know parties, hide amounts

This is **payload confidentiality**, not anonymity. Legally clean for enterprise use.

---

## Hackathon

Built for **MagicBlock Solana Blitz v2** — March 20–22, 2026

Theme: Privacy — building real-time and private apps on Solana

**Why KavachPay wins the Private ER track:**
Remove Private ER -> every salary becomes public -> product is worthless. Private ER is not a feature. It is the entire foundation.

---

*कवच — Indestructible armor. Your payments. Nobody else's business.*
