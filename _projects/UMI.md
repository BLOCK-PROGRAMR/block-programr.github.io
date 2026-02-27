---
title: "UMI (U and Me)"
description: "Peer-to-peer messaging over local WiFi with no central server — just you and me"
date: 2025-11-20
tech: ["Rust", "libp2p", "mDNS", "Request-Response Protocol"]
layout: default
---

# UMI (U and Me) — P2P Messaging with Rust

## Overview

**UMI** is a peer-to-peer chat system where devices talk directly to each other over local WiFi — no servers, no middlemen, no surveillance. Just you and me.

Most messaging apps (WhatsApp, Telegram, Discord) route your messages through their servers. They see everything. UMI cuts them out entirely. It's built for those moments when you want to chat privately with someone nearby without big tech watching.

---

## How P2P Works

**Normal messaging (Client-Server):**
```
You → App Server (WhatsApp/Google) → Your Friend
      ↑ Server sees everything
```

**UMI (Peer-to-Peer):**
```
You → Direct Connection → Your Friend
      ↑ No middleman
```

In a P2P network:
- Every device acts as both client and server
- Peers discover each other directly (no central directory)
- Messages travel point-to-point (faster, private, censorship-resistant)

**Real-world examples:** BitTorrent (file sharing), Blockchain (decentralized ledgers), early Skype (direct calling).

---

## How It Works

1. **Identity** — Each peer generates a unique Peer ID when they start
2. **Discovery** — Peers broadcast their existence using mDNS (multicast DNS) on local WiFi
3. **Connection** — When peers discover each other, they open encrypted communication channels
4. **Messaging** — Type a message, hit enter, and it broadcasts directly to all discovered peers

```
┌─────────┐         ┌─────────┐         ┌─────────┐
│ Peer A  │◄───────►│ Peer B  │◄───────►│ Peer C  │
└─────────┘    ↕    └─────────┘    ↕    └─────────┘
               │                     │
           mDNS Discovery (Local WiFi)
```

**No server. No database. No logs.**

---

## Architecture

UMI uses **libp2p**, a modular networking stack originally built for IPFS:

- **Transport** — TCP for reliable message delivery
- **Identity** — Ed25519 keypairs for peer authentication
- **Discovery** — mDNS broadcasts to find local peers
- **Protocol** — Request/Response pattern for messaging
- **Swarm** — Manages all peer connections and events

When you run UMI, it:
1. Generates your Peer ID (e.g., `12D3KooW...`)
2. Listens on available ports (random)
3. Broadcasts to local network via mDNS
4. Auto-connects when peers are discovered
5. Routes messages directly peer-to-peer

---

## Example Output (4 Peers on Same WiFi)

**Peer A** sends "hii":
```yaml
Your Peer ID: 12D3KooWGHktar8HC2q1GhPazDNSZrehimNYS9Xgy23D4ch47Aag
Discovered peer: 12D3KooWQmFxzswRsKwES8xwuB7CVm2jM49jJijAsBSTunKhPFt3
Discovered peer: 12D3KooWE9LufgKDyy6HLRMLyUH4ms3eAqC8MxvyghN2Nw1cXZ3o
Received request: 'hii'
```

**Peer B, C, D** all receive instantly:
```yaml
Discovered peer: 12D3KooWGHktar8HC2q1GhPazDNSZrehimNYS9Xgy23D4ch47Aag
Received request: 'hii'
```

All communication happens locally. Your router sees encrypted packets, but has no idea what you're saying.

---

## Tech Stack

- **Rust** — Memory-safe, concurrent, blazing fast
- **libp2p** — Modular P2P networking framework (same tech powering IPFS and Polkadot)
- **mDNS** — Local peer discovery without DNS servers
- **Request/Response** — Simple message protocol for chat
- **TCP** — Reliable transport layer for message delivery

---

## Features

- **No Server** — All communication is peer-to-peer
- **Local Network** — Works on WiFi/LAN without internet
- **Encrypted Channels** — libp2p handles secure transport
- **Auto-Discovery** — Peers find each other automatically
- **Broadcast Messages** — Send to all discovered peers at once
- **No Logs** — Nothing is stored; messages live only in memory

---

## Limitations

- **Range** — Only works on same local network (WiFi/LAN)
- **No History** — Messages disappear when peers disconnect
- **No Identity** — Peer IDs change every session
- **No Global Reach** — Can't message across the internet (yet)

Future versions will add NAT traversal and DHT for global P2P routing.

---

## Getting Started

**Install Rust:**
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

**Clone & Run:**
```bash
git clone https://github.com/SCATERLABs/UMI.git
cd UMI
cargo run
```

Open multiple terminals, run `cargo run` in each, and watch peers discover each other. Type messages and see them broadcast instantly.

---

## Why This Matters

Sometimes you just want to talk to someone nearby without Google, Meta, or your ISP knowing what you're saying. UMI is a proof of concept that **private, direct communication is possible** — no permissions needed.

---

🔗 [GitHub Repository](https://github.com/SCATERLABs/UMI)
