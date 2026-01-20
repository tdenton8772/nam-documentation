# Contributions & Collaboration

NAM is a **collaborative system**, but it is not an open-contribution project.

Contributions are welcome **by alignment and agreement**, not by default pull request.

This document defines how collaboration works, what kinds of contributions are appropriate, and how architectural integrity is preserved.

---

## Guiding Principle

NAM prioritizes **coherence over velocity**.

The system is built around a small set of non-negotiable invariants:
- Determinism
- Explicit semantics
- Geometric retrieval
- Separation of training from execution
- Rebuildability

Any contribution that weakens these principles is not a contribution — even if it “adds features.”

---

## Who Can Contribute

Contributions are accepted from individuals or teams who:

- Understand NAM’s design philosophy
- Are aligned with the project’s direction
- Are willing to collaborate openly and deliberately
- Have explicit permission to engage

This includes (but is not limited to):
- Research collaborators
- Early partners
- Design contributors
- Domain experts
- GTM and strategy collaborators (documentation, framing, evaluation guidance)

---

## How Contributions Happen

Contributions are **intentional and conversational**, not transactional.

Typical flow:
1. Discussion of intent and scope
2. Agreement on boundaries
3. Contribution or proposal
4. Review against architectural invariants
5. Integration (or rejection) with clear reasoning

There is no guarantee that any contribution will be accepted.

---

## What Kinds of Contributions Are Welcome

### Strongly Encouraged
- Documentation improvements
- Conceptual clarifications
- Design critiques
- Test cases and evaluation scenarios
- Domain-specific insights
- Training artifacts (under guidance)
- Examples and explanatory material

### Conditionally Accepted
- Encoder heads (with clear contracts)
- Training pipelines (offline only)
- Storage backends (behind abstractions)
- Query strategies (must preserve determinism)

### Explicitly Out of Scope
- Ad-hoc feature additions
- Runtime learning logic
- Heuristic ranking or scoring
- Non-deterministic execution paths
- Changes that collapse ambiguity prematurely
- Forks intended to compete with NAM

---

## Architectural Authority

NAM has a **single architectural authority**.

This is not about control — it is about:
- Maintaining invariants
- Preventing semantic drift
- Ensuring the system remains understandable

Disagreements are expected.
Fragmentation is not.

---

## Intellectual Property

All contributions:
- Fall under the existing licensing model
- Do not grant ownership or usage rights by default
- May be incorporated, modified, or declined

Contributors should assume:
- No implied rights
- No automatic attribution guarantees
- No retroactive licensing changes

If IP terms matter for your contribution, discuss them *before* contributing.

---

## Forks and Derivative Work

Forking NAM for experimentation or learning is acceptable.

Forking NAM to:
- Compete
- Circumvent licensing
- Repackage the system

…is not permitted without agreement.

---

## Why This Structure Exists

NAM is designed to explore a new class of systems.

That requires:
- Thoughtful iteration
- Shared vocabulary
- Mutual trust
- Long-term coherence

This contribution model exists to protect that work — not to exclude people.

---

## Questions or Proposals

If you are interested in contributing:
- Ideas
- Code
- Documentation
- Research
- GTM support

Start with a conversation.

Alignment comes before contribution.

