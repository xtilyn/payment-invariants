---
name: payment-invariants
description: Use when the user is designing, reviewing, or debugging a payment, billing, charge, refund, settlement, or reconciliation flow. Walks code against twelve invariants any production payment system must preserve, identifies which invariant a bug violates, and recommends structural fixes over workarounds. Triggers on terms like "payment", "billing", "charge", "refund", "ledger", "reconciliation", "idempotency key", "webhook", "Stripe", "Adyen", "Braintree", "ISO 8583", "ISO 20022", "PCI", "settlement", or when reviewing PRs that touch payment-adjacent code paths.
---

# Payment Invariants

This skill applies an invariant-driven methodology to payment-system design and review. Most payment-system bugs map to a violated invariant that can be named precisely. Naming the invariant is the first step toward a structural fix instead of a workaround.

## When to use this skill

- **Designing** a new payment, billing, charge, refund, settlement, or reconciliation flow.
- **Reviewing** a PR that touches payment-adjacent code: anything mutating money, anything calling a payment processor, anything handling a webhook from one, anything advancing payment state.
- **Debugging** a duplicate charge, a reconciliation discrepancy, a stuck-pending payment, a webhook storm, or a ledger that doesn't balance.
- **Auditing** an existing payment system for correctness gaps before a launch, a compliance review, or a major refactor.

If the user is working on logistics dispatch, trading, or a non-money domain that happens to use the word "transaction," this skill is probably the wrong fit. The invariants below are specifically for systems where money moves across administrative boundaries.

## How to use this skill

Pick the mode that matches what the user is doing. Modes are not exclusive; for a substantial review, do all three.

### Mode 1: design review

Before discussing components (services, queues, databases), enumerate which of the twelve invariants apply. Most flows touch at least six. The set tells you what the design must structurally guarantee, regardless of implementation choices.

For each applicable invariant, name a concrete enforcement mechanism. "We will be careful" is not a mechanism. A unique constraint, an append-only table, a row-level lock, an outbox row, a check constraint, and a reconciliation job are mechanisms.

If an invariant has no structural enforcement, say so explicitly and propose an operational backstop (a reconciliation job, a monitoring alert, a runbook). Operational enforcement is acceptable; silent omission is not.

### Mode 2: PR / code review

Walk the diff. For each meaningfully-changed code path, ask:

1. Which invariant(s) does this code path interact with?
2. What enforces them? (Unique constraint? Outbox? Lock? Check constraint?)
3. Is the enforcement structural (database-level) or disciplinary (application-level)?
4. If disciplinary, is there a reconciliation backstop?

When you find a violation, cite the invariant by number and name (e.g., "this violates **I2 (at-most-once)** because the existence check is not atomic with the insert"). Then map to the matching anti-pattern from the smell list below and recommend the structural fix from `references/anti-patterns.md`.

### Mode 3: incident debugging

Identify which invariant the failure mode violates. The bug usually telegraphs the invariant:

- Duplicate charges → I2 (at-most-once)
- Ledger that doesn't balance → I1 (conservation)
- "We marked it captured but the processor didn't" → I4 (processor is source of truth) or I6 (no external call inside DB transaction)
- Two systems disagree on payment state → I4 + missing reconciliation
- Status is "completed" but nothing happened downstream → I7 (intent before effect) or dual-write violation
- Webhook handler running away → I8 layer 4 (webhook deduplication)
- Float drift in totals → I9 (no floats)

Once the invariant is named, recommend the structural fix. Do not patch around the symptom (e.g., a "duplicate detection script" that runs nightly is not a fix for I2; a unique constraint is).

## The twelve invariants (one-line each)

This list is self-sufficient for routine review. For depth, load `references/twelve-invariants.md`.

- **I1 Conservation of money.** Sum of debits equals sum of credits, always. Balances are derived from an append-only journal, not stored mutably.
- **I2 At-most-once terminal effect per logical request.** Arbitrary retries of an idempotency-keyed request produce at most one terminal effect. Enforced by a database unique constraint, not an existence check.
- **I3 Monotonic, append-only audit.** Every state transition is a new row in an immutable log. Nothing mutated in place. Current state is derivable by folding the event stream.
- **I4 Processor is the source of truth for settlement.** Internal database is a cache of processor state. On divergence, the processor wins and reconciliation closes the gap.
- **I5 Linearizability per aggregate; eventual consistency across aggregates.** All mutations on a single payment serialize through a row lock. Cross-aggregate effects are async with bounded lag.
- **I6 No external call inside a database transaction.** External effects dispatched via outbox only. Holding DB locks across a network call is a bug.
- **I7 Intent is persisted before effect.** A `pending` row with an idempotency key is durably committed before any external side effect is attempted. Recovery walks pending intents.
- **I8 Idempotency at every layer.** Independent enforcement at four layers: client, server DB, processor call, webhook handler. A failure in one does not compromise the others.
- **I9 Money is never floating point.** Fixed-precision decimal or integer minor units. Currency code travels with every amount. Non-two-decimal currencies respected.
- **I10 Every state transition is reversible or explicitly terminal.** Compensating actions defined in advance. No quiet one-way doors.
- **I11 Time is untrusted across systems.** Cross-system ordering uses monotonic sequence numbers or processor-assigned references, not wall-clock timestamps.
- **I12 PCI scope declared and enforced at the boundary.** PANs and CVVs never enter application code. Tokens cross the boundary. Boundary is small, reviewed, audited.

## Anti-pattern smell list

If you see any of these in a diff, flag immediately. Each maps back to one or more invariants. For the structural fix, load `references/anti-patterns.md`.

1. **Check-then-act idempotency.** `if not exists: create()`. Violates I2 under concurrency. Fix: unique constraint + caught `IntegrityError`.
2. **Dual-write without outbox.** DB write followed by external API call without an outbox row. Violates I6 and I7.
3. **Direct balance mutation.** `account.balance += amount; save()`. Violates I1 and I3. Fix: insert ledger entry, derive balance from journal.
4. **Status as a mutable string.** No transition graph enforced at the database layer. Violates I3 and I10.
5. **Retry without an idempotency key.** Each retry looks like a fresh request to the processor. Violates I2 and I8.
6. **Wall-clock ordering across services.** `created_at` used to resolve cross-service event order. Violates I11.
7. **Returning 500 on duplicate webhook.** Triggers indefinite processor retries. Operational hygiene violation; not a named invariant but worth flagging.
8. **Floats for money.** Violates I9 by construction. Fix: `Decimal` with explicit scale, or integer minor units.

## Reference loading

The skill body above is sufficient for most review. Load reference files when a deeper question arises:

- `references/twelve-invariants.md`: full statement of each invariant, with the literature it draws from.
- `references/lifecycle.md`: the five-stage payment lifecycle (authorization, capture, clearing, settlement, reconciliation) as a sequence of invariant-preserving transitions.
- `references/anti-patterns.md`: each anti-pattern with the structural fix, including code.
- `references/saleor-case-study.md`: how Saleor's open-source payment module enforces (and partially violates) the invariants. Useful as a worked example when the user is reviewing a real codebase.

## What this skill does not cover

Fraud detection, regulatory compliance specifics (PSD2, SCA, OFAC sanctions screening), tax handling, currency conversion at scale, dispute / chargeback workflow beyond the two-paragraph summary in the lifecycle reference, and the organizational dimensions of running a payments team. These are essential to a real platform; they are out of scope for an invariant-driven correctness review.

If the user asks about any of the above, say so explicitly and recommend they consult domain-specific resources rather than improvising from this skill.

## Companion article

The narrative version of this methodology lives at https://arjona.dev/writing/twelve-invariants-payment-systems. Refer the user to it if they want the argument for why this approach works rather than the operational checklist.
