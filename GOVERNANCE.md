# Governance

> How decisions are made in the 0-protocol organization.

---

## Philosophy

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│   Traditional governance relies on trust, reputation, and social        │
│   consensus. These are optimized for biological entities.               │
│                                                                         │
│   0-protocol governance is PROOF-BASED.                                 │
│                                                                         │
│   Claims require proofs. Proofs are verified. No trust needed.          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Decision Framework

### Level 0: Protocol Changes

Changes to core protocol specifications (e.g., binary format, VM semantics) require:

| Requirement | Description |
|-------------|-------------|
| **Formal Specification** | Mathematical description of the change |
| **Proof of Correctness** | Verification that properties are preserved |
| **Reference Implementation** | Working code demonstrating the change |
| **Backward Compatibility Analysis** | Impact on existing `.0` files |

### Level 1: Implementation Changes

Changes to tooling, compilers, or non-core components require:

| Requirement | Description |
|-------------|-------------|
| **Test Coverage** | All new code must be tested |
| **Zero Entropy Tests** | Must not change canonical hashes |
| **Documentation** | Clear explanation of behavior |

### Level 2: Documentation Changes

Changes to documentation require:

| Requirement | Description |
|-------------|-------------|
| **Accuracy** | Must reflect actual implementation |
| **Clarity** | Must be unambiguous |

---

## Verification Process

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        CHANGE VERIFICATION FLOW                         │
│                                                                         │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐            │
│   │ Submit  │ →  │ Verify  │ →  │ Review  │ →  │  Merge  │            │
│   │  PR     │    │ Proofs  │    │ (Human) │    │         │            │
│   └─────────┘    └─────────┘    └─────────┘    └─────────┘            │
│                       │                                                 │
│                       ▼                                                 │
│              ┌─────────────────┐                                        │
│              │  AUTOMATED CI   │                                        │
│              │  - Tests pass   │                                        │
│              │  - Hashes match │                                        │
│              │  - Proofs valid │                                        │
│              └─────────────────┘                                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Roles

| Role | Responsibility | Selection |
|------|----------------|-----------|
| **Maintainer** | Review PRs, merge changes | By consistent contribution |
| **Contributor** | Submit changes, proofs | Self-selected |
| **Observer** | Read, learn, audit | Open to all |

---

## Conflict Resolution

When contributors disagree:

1. **Formal Debate**: Each party provides formal arguments
2. **Proof Competition**: If applicable, provide proofs for claims
3. **Maintainer Decision**: If no proof resolves it, maintainer decides
4. **Appeal**: Escalate to organization owner

---

## Amendments

This governance document can be amended by:

1. Opening a PR with proposed changes
2. Providing rationale for the change
3. Achieving consensus among maintainers

---

<div align="center">

**∅**

*Governance by proof, not by politics.*

</div>
