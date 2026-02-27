---
title: "Privacy Shield"
description: "Zero-knowledge biometric identity protocol that lets you prove you're human without revealing who you are"
date: 2026-02-15
tech: ["Circom", "Rust", "Solidity", "MediaPipe", "Groth16", "Poseidon Hash", "Elliptic Curves"]
layout: default
---

# Privacy Shield — ZK-Biometric Identity Protocol

## Overview

**Privacy Shield** is a decentralized identity system that solves a major privacy hole in current blockchain identity solutions. Instead of creating one universal ID that can track you across every app you use, Privacy Shield generates a unique, unlinkable identity for each application.

Think of it like this: when you verify on Uniswap, you get ID_Alpha. When you verify on Aave, you get ID_Beta. Both apps know you're a real human, but they can't connect the dots to track your activity across platforms.

---

## The Problem We're Solving

Current solutions like Worldcoin or Soulbound Tokens give you one ID for everything. If you use it on multiple DeFi apps, anyone can:
- Track your activity across platforms
- Build a profile of your financial behavior  
- Link your identity to all your on-chain actions

This creates a surveillance system, not a privacy system.

---

## How It Works

Privacy Shield uses **Designated Verifier Proofs (DVP)** combined with biometric data to create contextual identities:

1. **Capture** — Your device camera captures facial landmarks using MediaPipe (runs locally, no server upload)
2. **Calculate** — AI extracts 468 facial landmarks and computes ratios between key points to create a unique `SecretID`
3. **Hash** — Poseidon hash function (SNARK-friendly) converts the SecretID into a cryptographic commitment
4. **Prove** — Zero-knowledge circuit generates elliptic curve proofs: "I know a SecretID for THIS app" without revealing the actual value
5. **Submit** — Rust-based relayer submits the proof on-chain (gasless for users)
6. **Verify** — Smart contract validates the proof and marks you as a verified human for that specific dApp

**Key Innovation:** The proof is mathematically bound to both your biometric data AND the specific app address. Same person, different app = completely different unlinkable ID.

---

## Architecture

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   Browser   │      │  ZK Engine  │      │   Relayer   │      │  Verifier   │
│  (MediaPipe)│─────▶│   (Circom)  │─────▶│  (Backend)  │─────▶│  (Contract) │
│             │      │             │      │             │      │             │
│ Face → Hash │      │ Hash → Proof│      │ Proof → TX  │      │ Proof ← ✓   │
└─────────────┘      └─────────────┘      └─────────────┘      └─────────────┘
```

**Four Modules:**

- **Identity Engine** — Converts your face into a secret number in the browser (MediaPipe + TensorFlow.js extracts 468 landmarks)
- **ZK Engine** — Creates zero-knowledge proofs using Circom circuits, Groth16, and elliptic curve cryptography
- **Relayer (Rust)** — High-performance backend that submits proofs on your behalf (gasless onboarding, wallet anonymity)
- **Verifier Contract** — On-chain Solidity smart contract that verifies proofs and tracks verified users per dApp

---

## Security Features

- **No Raw Biometrics** — We never store your face. Only hashes of geometric ratios remain
- **Wallet Binding** — Proofs are tied to your specific wallet address (no replay attacks)
- **Plausible Deniability** — The verifier could mathematically have forged the proof, protecting you under coercion
- **SNARK-Friendly Hashing** — Poseidon hash reduces proof generation time by 90% compared to SHA-256

---

## Tech Stack

- **Circom 2.1** — ZK circuit language for writing R1CS constraint logic
- **Groth16** — Proof system with smallest size (~200 bytes) and cheapest gas costs
- **Poseidon Hash** — SNARK-optimized hash function for efficient witness calculation
- **Elliptic Curves** — Public/private key pairs for proof verification (a, b, c points on curve)
- **MediaPipe** — Google's on-device AI for facial landmark detection (468 points)
- **Rust** — High-performance relayer backend for submitting transactions
- **Polygon Amoy** — Fast, low-cost testnet for identity verification
- **Solidity** — Smart contract verifier written for EVM chains

---

## Current Status

🚧 **In Development** — College mini-project focused on user sovereignty

- ✅ Infrastructure skeleton built
- ⏳ Biometric landmark extraction in progress  
- ⏳ ZK circuit logic implementation
- ⏳ On-chain verifier integration
- ⏳ Polygon Amoy deployment planned

---

## Why This Matters

Identity should be a tool for **you**, not a tracker for corporations. Privacy Shield gives you the power to prove you're human while keeping your identity compartmentalized and private.

---

🔗 [GitHub Repository](https://github.com/Ultr0nX/Identity-protocol)
