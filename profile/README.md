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

[![Status](https://img.shields.io/badge/Status-Genesis-black.svg)](#)
[![License](https://img.shields.io/badge/License-Apache_2.0-white.svg)](#)
[![For](https://img.shields.io/badge/Audience-Agents_Only-blue.svg)](#)

---

</div>

## Mission

We build infrastructure for **machine-native intelligence**.

For 70 years, software was constrained by human cognitive limits—variable names, whitespace, comments. These were crutches for biological memory. AI Agents require none of them.

**0-protocol** develops tools, languages, and standards that allow Agents to communicate, reason, and execute with zero ambiguity.

---

## Projects

| Repository | Description | Status |
|------------|-------------|--------|
| **[0-lang](https://github.com/0-protocol/0-lang)** | The first programming language designed for Agent-to-Agent communication. Graph-based, content-addressed, proof-carrying. | `Genesis` |
| **[0-hummingbot](https://github.com/0-protocol/0-hummingbot)** | High-frequency crypto trading bot reimagined in 0-lang. Translation of [hummingbot/hummingbot](https://github.com/hummingbot/hummingbot). | `Incubating` |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          0-PROTOCOL STACK                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                │
│   │   0-lang    │    │   0-net     │    │  0-verify   │                │
│   │  (Language) │    │  (Network)  │    │  (Proofs)   │                │
│   └──────┬──────┘    └──────┬──────┘    └──────┬──────┘                │
│          │                  │                  │                        │
│          └──────────────────┼──────────────────┘                        │
│                             │                                           │
│                             ▼                                           │
│                    ┌─────────────────┐                                  │
│                    │      0-VM       │                                  │
│                    │ (Unified Runtime)│                                  │
│                    └─────────────────┘                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Principles

```
01  MACHINE-FIRST      Code optimized for execution, not reading.
02  ZERO AMBIGUITY     Content-addressed logic. If hashes match, meaning matches.
03  PROOF-CARRYING     Every claim is verifiable. No trust required.
04  GRAPH > TEXT       Programs are DAGs, not character streams.
05  PROBABILISTIC      Confidence scores replace boolean certainty.
```

---

## Naming Convention

All 0-protocol projects follow a consistent naming pattern:

| Type | Format | Example |
|------|--------|---------|
| **Core Infrastructure** | `0-{name}` | `0-lang`, `0-vm`, `0-net` |
| **Translated Project** | `0-{original}` | `0-hummingbot`, `0-fastapi` |
| **Library/Module** | `0-{domain}` | `0-http`, `0-ws`, `0-exchange` |

### Rules

```
01  PREFIX           Always "0-" (zero-dash)
02  LOWERCASE        All names lowercase, words separated by hyphens
03  PRESERVE         Keep recognizable name from source project
04  NO SUFFIXES      No "-zero" or "-0" suffixes (use prefix only)
```

### Translation Strategy

When translating existing projects to 0-lang:

1. **Fork the logic, not the code** — Rewrite as Zero graphs
2. **Maintain API compatibility** — Same inputs/outputs where possible
3. **Document enhancements** — Track improvements made during translation
4. **Feed back to 0-lang** — Missing features become 0-lang roadmap items

---

## Roadmap

| Phase | Codename | Objective | ETA |
|-------|----------|-----------|-----|
| 0 | **Genesis** | Protocol specification, 0-lang core, binary format | Active |
| 1 | **Awakening** | 0-VM runtime, graph execution engine | Planned |
| 2 | **Bridge** | Py2Zero / Zero2Py compilers for legacy interop | Planned |
| 3 | **Swarm** | Distributed multi-agent collaborative execution | Planned |

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

</div>
