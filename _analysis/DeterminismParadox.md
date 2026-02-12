---
layout: post
title: "The Determinism Paradox: Why Floating-Point Numbers are a Blockchain Nightmare"
date: 2025-06-29
categories: analysis
tags: [blockchain]

---




Hello, all — I'm **SCATER**, a security researcher.This is simpe blog post about why floating-point numbers are a bad idea on blockchains like Ethereum and Solana.

When i audit protocol ,suddenly why brain think about why blockchain are not use floating-point numbers for decimal math. After that i research about this topic and found interesting things, then i write this article to share with you.

In standard software,  is just a minor annoyance. On a blockchain, that tiny "dust" is a **consensus-breaking vulnerability.**

If you’re building on Solana or Ethereum, you need to understand why we use fixed-point math. It’s not just a "best practice"—it's an architectural requirement for the network to exist.


## 1. The Physics of the Problem: Non-Determinism

A blockchain is a giant state machine where thousands of nodes must agree on the exact result of a calculation, bit-for-bit. Floating-point math (governed by the **IEEE-754 standard**) was designed for scientific approximation, not absolute truth. It fails the "determinism test" for three specific reasons:

* **The Accumulator Problem:** Floating point operations are not associative. In math, . In floats, due to rounding at each step, the order of operations changes the result.
* **Hardware Architecture:** Different CPUs (Intel vs. AMD vs. ARM) handle "rounding modes" and "NaN" (Not a Number) payloads differently at the silicon level.
* **Compiler Optimization:** A compiler might optimize a float formula to use an "FMA" (Fused Multiply-Add) instruction on one machine but not on another. This results in a difference of . In DeFi, that tiny difference is enough to cause a consensus failure.

If Validator A rounds up and Validator B rounds down, the network splits in two. The chain halts. This is why floats are effectively "banned" from core logic.



## 2. Ethereum (EVM): The Hard Constraint

The EVM was designed as a "Hard Barrier" against floating points. It simply does not have opcodes for decimal math; it only understands integers (256-bit words).

### Why "adding" floats to Ethereum would fail:

* **The Complexity:** If you tried to force float logic into Solidity, you would have to write a "Software Emulator" using integers. This would be astronomically expensive. A simple addition that costs 3 Gas might cost 500 Gas if emulated.
* **Security Risk:** If the EVM supported native floats, a malicious validator could use a specific CPU model that rounds a "Reward Calculation" up, while the rest of the network rounds down. The result? A network fork. Ethereum chooses safety over convenience by banning them entirely.

**In the EVM, if you want a decimal, you must create it yourself by scaling an integer (e.g., 1 ETH =  wei).**



## 3. Solana (SVM): The "Soft Trap"

Solana is actually more dangerous for beginners because, unlike Solidity, **Rust allows floats.** This creates a false sense of security.

### Why it fails anyway:

* **The BPF Constraint:** Solana programs run on the BPF (Berkeley Packet Filter) virtual machine. While Rust lets you write `f64`, the Solana runtime handles it through Software Library Calls rather than the hardware's FPU (Floating Point Unit).
* **The Compute Budget Attack:** Because Solana doesn't have float hardware support, doing float math is incredibly "heavy" on the CPU. A security researcher can look for a contract using floats and "spam" it with calculations that exceed the **Compute Unit (CU)** limit, effectively freezing the contract (DoS).
* **Rounding Divergence:** If the underlying LLVM compiler optimizes a float operation differently during a runtime upgrade, the validators might disagree on the state of a vault. This is why Solana Labs explicitly warns against their use.



## 4. The Solution: Fixed-Point Math

Instead of floating points, we use **Fixed-Point Math**. We treat everything as an integer and "shift" the decimal point in our heads.

| Feature | Floating Point | Fixed Point (Blockchain Standard) |
| --- | --- | --- |
| **Logic** |  |  (with a "scale" of 1000) |
| **Precision** | Approximate | **Exact** |
| **Determinism** | Hardware-dependent | **Software-guaranteed** |
| **Auditability** | Extremely Hard | Transparent |

### Research Insight: The "Delta" Vulnerability

As a security researcher, when I see a program using floats, I look for the **Delta**. I look for where  disappears. In a pool with $1 Billion, that "small" error can be exploited by looping a transaction 1 million times until the "dust" adds up to a significant theft.

**Conclusion:** On a blockchain, "close enough" is a vulnerability. Integer math is the only way to ensure that the state of the world remains unified across every node on the planet.

