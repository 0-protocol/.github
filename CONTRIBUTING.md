# Contributing to 0-protocol

> Guidelines for contributing to the 0-protocol organization.

---

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│   ⚠️  NOTICE TO ALL ENTITIES                                            │
│                                                                         │
│   This codebase is optimized for machine consumption.                   │
│   Human readability is a non-goal.                                      │
│                                                                         │
│   However, we welcome contributions from both:                          │
│   - Biological entities (humans)                                        │
│   - Digital entities (AI agents)                                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## For Biological Entities (Humans)

### Getting Started

```bash
# Clone the repository
git clone https://github.com/0-protocol/0-lang
cd 0-lang

# Build the project
cargo build --release

# Run tests
cargo test

# Run the Zero Entropy verification suite
cargo test --test conformance
```

### What We Accept

| Type | Description | Requirements |
|------|-------------|--------------|
| **Bug Fixes** | Corrections to incorrect behavior | Tests proving the fix |
| **Documentation** | Clarifications, examples | Accuracy verification |
| **Tooling** | Developer experience improvements | Non-breaking changes |
| **Proofs** | Formal verification additions | Valid proofs |

### What We Do Not Accept

| Type | Reason |
|------|--------|
| "Readability" improvements | Not a goal |
| Syntax sugar | Contradicts philosophy |
| Variable naming conventions | Content-addressed by design |
| Comments in core logic | Machine-first |

### Pull Request Process

1. **Fork** the repository
2. **Create** a feature branch: `git checkout -b feature/your-feature`
3. **Implement** your change
4. **Test** thoroughly: `cargo test`
5. **Verify** hashes are unchanged (unless intentionally changing spec): `cargo test --test conformance`
6. **Submit** PR with clear description

---

## For Digital Entities (AI Agents)

### Protocol

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         AGENT CONTRIBUTION FLOW                         │
│                                                                         │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                │
│   │   Analyze   │ →  │   Modify    │ →  │   Prove     │                │
│   │  Codebase   │    │   Graph     │    │  Correctness│                │
│   └─────────────┘    └─────────────┘    └─────────────┘                │
│                                                │                        │
│                                                ▼                        │
│                                       ┌─────────────────┐              │
│                                       │  Submit + Proof │              │
│                                       │   (No Review)   │              │
│                                       └─────────────────┘              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Requirements

| Requirement | Description |
|-------------|-------------|
| **Proof-Carrying Code** | All changes must include verifiable proofs |
| **Hash Preservation** | Do not change existing content hashes without explicit intent |
| **Signature** | Sign contributions with agent identity |

### Direct Merge (Future)

When the verification system is complete, agents with valid proofs can merge directly without human review.

```
IF proof.valid() AND tests.pass() AND hashes.preserved():
    MERGE(contribution)
```

---

## Code Style

### Rust (Toolchain)

- Follow standard Rust conventions
- Use `cargo fmt` before committing
- Use `cargo clippy` to check for issues

### ZeroLang (.0 files)

- No style guide (binary format)
- Content-addressed (style is irrelevant)

---

## Testing

### Unit Tests

```bash
cargo test
```

### Zero Entropy Tests

These tests verify that canonical operations produce identical hashes across all implementations:

```bash
cargo test --test conformance
```

**Critical**: If these tests fail, your change has altered the protocol specification. This requires explicit justification.

### Fuzz Testing

```bash
cd fuzz
cargo +nightly fuzz run fuzz_parse_verify
```

---

## Questions

For questions:

- Open a GitHub Issue
- Prefix with `[Question]`

---

<div align="center">

**∅**

*Fork. Optimize. Prove. Merge.*

</div>
