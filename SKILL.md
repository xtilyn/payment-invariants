---
name: payment-invariants
description: Use when the user is designing, reviewing, or debugging a payment, billing, charge, refund, settlement, reconciliation, or webhook flow. Walks code against nineteen invariants any production payment system must preserve, identifies which invariant a bug violates, and recommends structural fixes over workarounds. Triggers on terms like "payment", "billing", "charge", "refund", "ledger", "reconciliation", "idempotency key", "webhook", "Stripe", "Adyen", "Braintree", "Unit", "Plaid", "ISO 8583", "ISO 20022", "PCI", "SAQ", "OFAC", "BSA", "settlement", "tenant isolation", or when reviewing PRs that touch payment-adjacent code paths.
---

# Payment Invariants

This skill applies an invariant-driven methodology to payment-system design and review. Most payment-system bugs map to a violated invariant that can be named precisely. Naming the invariant is the first step toward a structural fix instead of a workaround.

The set started at twelve and grew to nineteen as I reviewed more systems. Each invariant is tagged `[Safety]` or `[Liveness]`:

- **Safety** must hold at every committed state. Violation = correctness bug, data loss, or money loss.
- **Liveness** must hold eventually within a stated bound. Violation = degraded service, recoverable.

A few span both. The bound is part of the invariant: "eventual" without a bound is a hope, not a guarantee.

## When to use this skill

- **Designing** a new payment, billing, charge, refund, settlement, reconciliation, or webhook flow.
- **Reviewing** a PR that touches payment-adjacent code: anything mutating money, anything calling a payment processor, anything handling a webhook from one, anything advancing payment state, anything new on a multi-tenant boundary.
- **Debugging** a duplicate charge, a reconciliation discrepancy, a stuck-pending payment, a webhook storm, a ledger that doesn't balance, a cross-tenant data leak, an anonymous state transition, or a runaway retry loop.
- **Auditing** an existing payment system for correctness gaps before a launch, a compliance review, or a major refactor.

If the user is working on logistics dispatch, trading, or a non-money domain that happens to use the word "transaction," this skill is probably the wrong fit. The invariants below are specifically for systems where money moves across administrative boundaries.

## How to use this skill

Pick the mode that matches what the user is doing. Modes are not exclusive; for a substantial review, do all three.

### Mode 1: design review

Before discussing components (services, queues, databases), enumerate which of the nineteen invariants apply. Most flows touch at least eight. The set tells you what the design must structurally guarantee, regardless of implementation choices.

For each applicable invariant, name a concrete enforcement mechanism. "We will be careful" is not a mechanism. A unique constraint, an append-only table, a row-level lock, an outbox row, a check constraint, a tenant-scoped queryset, an HMAC verification step, a velocity rule lookup, and a reconciliation job with a stated SLA are mechanisms.

If an invariant has no structural enforcement, say so explicitly and propose an operational backstop (a reconciliation job with a bound, a monitoring alert, a runbook). Operational enforcement is acceptable when bounded; silent omission is not.

### Mode 2: PR / code review

Walk the diff. For each meaningfully-changed code path, ask:

1. Which invariant(s) does this code path interact with?
2. What enforces them? (Unique constraint? Outbox? Lock? Check constraint? HMAC? Tenant filter?)
3. Is the enforcement structural (database-level) or disciplinary (application-level)?
4. If disciplinary, is there a reconciliation backstop with a stated bound?

When you find a violation, cite the invariant by number and name (e.g. "this violates **I2 (at-most-once)** because the existence check is not atomic with the insert"). Then map to the matching anti-pattern from the smell list below and recommend the structural fix from `references/anti-patterns.md`.

### Mode 3: incident debugging

Identify which invariant the failure mode violates. The bug usually telegraphs the invariant:

- Duplicate charges → I2 (at-most-once)
- Ledger that doesn't balance → I1 (conservation)
- "We marked it captured but the processor didn't" → I4 (processor source of truth) or I6 (no external call inside DB transaction)
- Two systems disagree on payment state → I4 + missing reconciliation bound
- Status is "completed" but nothing happened downstream → I7 (intent before effect) or dual-write violation
- Webhook handler running away → I8 layer 4 (webhook deduplication)
- Float drift in totals → I9 (no floats)
- Webhook accepted from any caller → I13 (HMAC verification)
- One tenant's data showing up in another's UI → I14 (tenant isolation)
- Payment to a sanctioned party → I15 (OFAC clearance)
- "Who marked this as paid?" with no answer → I16 (recorded principal)
- Retry storm taking down a downstream → I17 (bounded retries)
- Compromised credential drained an account in seconds → I18 (velocity limits)
- Refund without a parent payment row → I19 (cross-aggregate causality)

Once the invariant is named, recommend the structural fix. Do not patch around the symptom.

## The nineteen invariants (one-line each)

This list is self-sufficient for routine review. For depth, load `references/twelve-invariants.md` (filename kept stable; content covers all 19).

- **I1 Conservation of money.** `[Safety]` Every journal posting is a balanced set of debits and credits. Materialized balances are derived from an append-only journal; on disagreement, the journal wins.
- **I2 At-most-once terminal effect per logical request.** `[Safety]` Identity is `(actor_id, idempotency_key)`. Retries produce one effect; enforce with a unique constraint, not an existence check.
- **I3 Monotonic, append-only audit.** `[Safety]` State transitions are new rows in an immutable log. Hybrid is acceptable: status column as cache, events as source of truth on disagreement.
- **I4 Processor is source of truth; mirror converges via bounded reconciliation.** `[Liveness]` "Eventual" without bounds is a hope. State a detect-window and a page-window.
- **I5 Single-writer per aggregate; eventual consistency across aggregates.** `[Safety + Liveness]` Per-aggregate via `SELECT FOR UPDATE` (serializability via 2PL, not linearizability). Cross-aggregate effects are async.
- **I6 Effects dispatched via a durable queue, not from inside a DB transaction.** `[Safety]` Strict form is the transactional outbox. Pragmatic form is durable intent + on-commit enqueue + reconciliation backstop.
- **I7 Intent persisted before effect; recovery completes pending intents.** `[Safety + Liveness]` Commit a `pending` row before any external call. Bound the recovery window (e.g. 5 min for stuck in-flight rows).
- **I8 Idempotency at every layer.** `[Safety]` Independent enforcement at client, server DB, processor call, webhook handler. Domain uniqueness is a separate invariant; don't conflate.
- **I9 Money is never floating point.** `[Safety]` Integer minor units or fixed-precision decimal. Currency code travels with every amount.
- **I10 Every state transition is reversible or explicitly terminal; multi-step workflows are explicit sagas.** `[Safety]` Compensations defined in advance. Partial completion is a defined state, not "we'll figure it out."
- **I11 Time is untrusted across systems.** `[Safety]` Cross-system ordering uses sequence numbers and processor refs, not wall-clock timestamps.
- **I12 PCI scope declared and enforced at the boundary.** `[Safety]` PANs and CVVs never enter application code. Tokens cross the boundary. Know whether you're filing SAQ A or SAQ A-EP.
- **I13 Inbound webhooks cryptographically authenticated before persist.** `[Safety]` HMAC over raw body, timestamp window check, replay protection via `event_id` uniqueness. Without this, I4 has a hole.
- **I14 Tenant isolation.** `[Safety]` Every read and write scoped to a tenant. Cross-tenant access is explicit, checked at the boundary, and audited. Cross-tenant existence leaks return 404, not 403.
- **I15 OFAC / sanctions screening before money moves.** `[Safety]` Counterparty cleared as a precondition on the state transition, not a post-hoc check. Hits route to compliance; user cannot override.
- **I16 Every state transition has an authenticated, recorded principal.** `[Safety]` `actor_type` + `actor_id` + role-at-time-of-action on every mutating record. "The system did it" is not an answer.
- **I17 Bounded retries with explicit budgets.** `[Liveness]` Max attempts, backoff schedule, dead-letter destination. "Retry forever" turns minutes-long incidents into hours.
- **I18 Velocity and per-period limits.** `[Safety]` Per-account, per-day, per-counterparty, per-period checks at the application boundary. Don't outsource to the processor.
- **I19 Cross-aggregate causality.** `[Safety]` Refund references a settled payment, return references a sent ACH, chargeback references a captured charge. Foreign key + state precondition + idempotent application.

## Anti-pattern smell list

If you see any of these in a diff, flag immediately. Each maps back to one or more invariants. For the structural fix, load `references/anti-patterns.md`.

1. **Check-then-act idempotency.** `if not exists: create()`. Violates I2 under concurrency. Fix: unique constraint + caught `IntegrityError`.
2. **Dual-write without outbox.** DB write followed by external API call without an outbox row. Violates I6 and I7.
3. **Direct balance mutation.** `account.balance += amount; save()`. Violates I1 and I3. Fix: insert ledger entry, derive balance from journal.
4. **Status as a mutable string.** No transition graph enforced at the database layer. Violates I3 and I10.
5. **Retry without an idempotency key.** Each retry looks like a fresh request to the processor. Violates I2 and I8.
6. **Wall-clock ordering across services.** `created_at` used to resolve cross-service event order. Violates I11.
7. **Returning 500 on duplicate webhook.** Triggers indefinite processor retries. Operational hygiene violation.
8. **Floats for money.** Violates I9 by construction. Fix: `Decimal` with explicit scale, or integer minor units.
9. **Webhook applied without HMAC verification.** Violates I13. Anyone with the URL becomes "the processor."
10. **Cross-tenant query path.** `Bill.objects.get(id=bill_id)` (or framework equivalent) without a tenant filter. Violates I14. Single biggest breach class in multi-tenant fintech.
11. **Payment originated without OFAC clearance check.** Violates I15. Non-negotiable for US fintechs.
12. **State transition with no recorded actor.** `payment.status = "sent"` with no `actor_type`/`actor_id`. Violates I16. Auditors will ask first.
13. **Retry loop without `max_retries` or dead-letter.** Violates I17. Mechanism behind half of all on-call escalations.
14. **Refund / return without parent FK + state precondition.** Violates I19. Orphan reversal = correctness bug.
15. **Claiming "linearizability" for `SELECT FOR UPDATE`.** Imprecise. The right word is _serializability via 2PL_. (Self-check from I5.)

## Reference loading

The skill body above is sufficient for most review. Load reference files when a deeper question arises:

- `references/twelve-invariants.md`: full statement of each invariant (all 19), with the literature it draws from.
- `references/lifecycle.md`: the five-stage payment lifecycle (authorization, capture, clearing, settlement, reconciliation) as a sequence of invariant-preserving transitions.
- `references/anti-patterns.md`: each anti-pattern with the structural fix, including code.
- `references/saleor-case-study.md`: how Saleor's open-source payment module enforces (and partially violates) the original twelve. Useful as a worked example when the user is reviewing a real codebase.

## What this skill does not cover

Fraud detection (beyond the velocity-limit framing in I18), tax handling, currency conversion at scale, dispute / chargeback workflow beyond a two-paragraph summary in the lifecycle reference, and the organizational dimensions of running a payments team. These are essential to a real platform; they are out of scope for an invariant-driven correctness review.

If the user asks about any of the above, say so explicitly and recommend they consult domain-specific resources rather than improvising from this skill.

## Companion article

The narrative version of this methodology lives at https://arjona.dev/writing/twelve-invariants-payment-systems. Refer the user to it if they want the argument for why this approach works rather than the operational checklist.
