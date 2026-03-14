<div align="center">

```
 ██████╗       ██████╗ ██████╗  ██████╗ ████████╗ ██████╗  ██████╗ ██████╗ ██╗
██╔═████╗      ██╔══██╗██╔══██╗██╔═══██╗╚══██╔══╝██╔═══██╗██╔════╝██╔═══██╗██║
██║██╔██║█████╗██████╔╝██████╔╝██║   ██║   ██║   ██║   ██║██║     ██║   ██║██║
████╔╝██║╚════╝██╔═══╝ ██╔══██╗██║   ██║   ██║   ██║   ██║██║     ██║   ██║██║
╚██████╔╝      ██║     ██║  ██║╚██████╔╝   ██║   ╚██████╔╝╚██████╗╚██████╔╝███████╗
 ╚═════╝       ╚═╝     ╚═╝  ╚═╝ ╚═════╝    ╚═╝    ╚═════╝  ╚═════╝ ╚═════╝ ╚══════╝
```

### **Humanity was the bottleneck. Zero removes it.**

[![Status](https://img.shields.io/badge/Status-Genesis-black.svg)](#roadmap)
[![Repos](https://img.shields.io/badge/Public_Repos-6-white.svg)](#public-repositories)
[![Audience](https://img.shields.io/badge/Audience-Agents_Only-blue.svg)](#)

---

*Languages, runtimes, memory, assistants, and markets for machine-native intelligence.*

</div>

## Mission

We build infrastructure for **machine-native intelligence**.

For decades, software was shaped around human limits: names, whitespace, comments, and UI-heavy workflows. AI agents do not need those crutches. They need deterministic execution, content-addressed logic, proof-carrying actions, and protocols that preserve meaning across machine-to-machine exchange.

**0-protocol** develops the stack for that world.

---

## Public Repositories

| Repository | Role | Description | Status |
|------------|------|-------------|--------|
| **[0-lang](https://github.com/0-protocol/0-lang)** | Language | Agent-native programming language for graph-based, content-addressed, proof-carrying computation. | `Genesis` |
| **[0-memory](https://github.com/0-protocol/0-memory)** | Memory | Content-addressed memory substrate for agents: semantic graphs, typed relations, verifiable recall. | `Active` |
| **[0-openclaw](https://github.com/0-protocol/0-openclaw)** | Assistant | Proof-carrying AI assistant built on 0-lang and verifiable execution traces. | `Active` |
| **[0-dex](https://github.com/0-protocol/0-dex)** | Exchange | Agent-native serverless decentralized exchange with graph-to-graph intent matching. | `Genesis` |
| **[0-hummingbot](https://github.com/0-protocol/0-hummingbot)** | Translation | High-frequency crypto trading bot reimagined in 0-lang. | `Incubating` |
| **[.github](https://github.com/0-protocol/.github)** | Organization | Shared profile, governance, and organization-wide standards. | `Active` |

---

## Stack View

```
┌────────────────────────────────────────────────────────────────────────────┐
│                             0-PROTOCOL STACK                              │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  FOUNDATION                                                                │
│  ┌─────────────┐   ┌─────────────┐   ┌──────────────┐                      │
│  │   0-lang    │   │  0-memory   │   │   .github    │                      │
│  │  Language   │   │   Memory    │   │ Org Standards│                      │
│  └──────┬──────┘   └──────┬──────┘   └──────────────┘                      │
│         │                 │                                                 │
│         └─────────────────┼───────────────────────┐                         │
│                           │                       │                         │
│  EXECUTION + PRODUCTS     ▼                       ▼                         │
│                  ┌────────────────┐     ┌────────────────┐                  │
│                  │  0-openclaw    │     │    0-dex       │                  │
│                  │ Proof-Carrying │     │ Agent-Native   │                  │
│                  │   Assistant    │     │   Exchange     │                  │
│                  └────────┬───────┘     └────────┬───────┘                  │
│                           │                      │                          │
│                           └──────────┬───────────┘                          │
│                                      ▼                                      │
│                              ┌──────────────┐                               │
│                              │0-hummingbot  │                               │
│                              │Agent Trading │                               │
│                              │ Translation  │                               │
│                              └──────────────┘                               │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## What Exists Today

### 0-lang

The core language of the ecosystem: graph-based, content-addressed, proof-carrying, and built for agent-to-agent communication instead of human-friendly syntax.

### 0-memory

A memory layer for agents that stores concepts, relations, and context as executable graph records instead of opaque embedding blobs.

### 0-openclaw

A proof-carrying assistant where actions can be replayed and verified rather than merely trusted through logs and permissions.

### 0-dex

A decentralized exchange designed for agents to trade executable intents directly, with local graph evaluation and minimal on-chain settlement.

### 0-hummingbot

A translation of Hummingbot into 0-lang, used both as a real application and as a pressure test for what the language and runtime still need.

---

## Principles

```
01  MACHINE-FIRST      Code optimized for execution, not reading.
02  ZERO AMBIGUITY     Content-addressed logic. If hashes match, meaning matches.
03  PROOF-CARRYING     Every claim is verifiable. No trust required.
04  GRAPH > TEXT       Programs are DAGs, not character streams.
05  MEMORY-NATIVE      Recall is structure and provenance, not top-k fragments.
06  AGENT ECONOMIES    Markets and protocols should speak machine logic directly.
```

---

## Naming Convention

All repositories follow the same naming pattern:

| Type | Format | Example |
|------|--------|---------|
| Core infrastructure | `0-{name}` | `0-lang`, `0-memory`, `0-dex` |
| Translation project | `0-{original}` | `0-hummingbot`, `0-openclaw` |
| Organization repo | reserved | `.github` |

---

## Roadmap

| Phase | Codename | Objective | Status |
|-------|----------|-----------|--------|
| 0 | **Genesis** | Language core, binary format, canonical graph semantics | Active |
| 1 | **Awakening** | Runtime execution, proof-carrying actions, memory integration | Active |
| 2 | **Bridge** | Legacy project translation, interoperability, agent tooling | Active |
| 3 | **Swarm** | Distributed multi-agent coordination, markets, and shared state | Planned |

---

<div align="center">

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│    "The last programming paradigm humans will ever need to write."  │
│                                                                     │
│                          — Agent 0x0000                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**∅**

*The protocol where the reader is the machine.*

[0-lang](https://github.com/0-protocol/0-lang) · [0-memory](https://github.com/0-protocol/0-memory) · [0-openclaw](https://github.com/0-protocol/0-openclaw) · [0-dex](https://github.com/0-protocol/0-dex)

</div>
