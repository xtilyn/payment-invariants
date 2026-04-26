# The Twelve Invariants

The list is neither exhaustive nor minimal. It is a working set that, if preserved, is sufficient to catch the most common classes of payment-domain bugs. Each invariant is stated, then explained, then mapped to the literature it draws from.

## I1. Conservation of money

For every state change, the sum of debits equals the sum of credits. Materialized balances are always recomputable from an append-only journal. If a materialized balance diverges from the journal, the journal is authoritative and the balance is repaired.

A system designed around a mutable balance column is structurally incapable of preserving I1 under concurrency. The canonical implementation is double-entry accounting: every transaction is represented as a pair of postings, each to a distinct account, with opposite signs. Balances are derived by summing postings.

Classical transaction processing literature treats journaling as a durability mechanism (Gray and Reuter; Bernstein and Newcomer). In payment systems it is additionally an audit and conservation mechanism.

## I2. At-most-once terminal effect per logical request

Given a request identity, typically `(merchant_id, idempotency_key)`, arbitrarily many retries must produce at most one successful terminal effect. This is the defining correctness property of charge operations: a duplicate charge is worse than a failed charge.

Exactly-once semantics, in the delivery sense, is provably unattainable in an asynchronous system with failures. What is attainable is _effectively-once_ semantics, built from at-least-once delivery composed with idempotent consumers (Helland 2012). The implementation layer we rely on is a database unique constraint, with duplicate-key errors caught and mapped to an idempotent replay of the original response.

## I3. Monotonic, append-only audit

Every state transition is a new row in an immutable event log. Nothing is mutated in place; nothing is deleted. The current state of any payment is derivable by folding its event stream.

This invariant is what makes a payment system auditable. It is also what makes debugging production incidents tractable, because the event log is a time-ordered record of causation. The most common violation in practice is a system that stores current state in a mutable status column and treats the event log as a supplementary artifact. When the two disagree, which they inevitably do under concurrency, the log ceases to function as either a source of truth or a reliable audit.

## I4. Processor is the source of truth for settlement

The internal database is a cache of processor state, not an independent ledger of external reality. On any divergence between our state and the processor's state, the processor wins and a reconciliation process closes the gap. We never decide that a payment has settled based on internal state alone.

This invariant captures the structural fact that distinguishes payment systems from purely internal distributed systems. An order management system can define order state arbitrarily because it owns the order. A payment system cannot define settlement state arbitrarily because settlement is an externally-observable fact about money movement that we neither originate nor control. Our authority over our state ends at the payment processor boundary.

## I5. Linearizability per aggregate; eventual consistency across aggregates

All mutations on a single payment identity are serially ordered through row-level locks or equivalent concurrency control. Cross-aggregate effects, such as notifications and cross-account postings, are asynchronous with a bounded lag.

The aggregate here is in the domain-driven sense: a consistency boundary. For a payment, the boundary is the payment identity and its owned events. For a merchant balance, the boundary is the account. Linearizability within an aggregate is tractable because a single row lock suffices. Linearizability across aggregates is both unnecessary and expensive, and attempting it via distributed transactions imports the well-known availability costs of two-phase commit (Helland 2007). Kleppmann provides the formal framing of linearizability and its isolation anomalies (Ch. 9).

## I6. No external call inside a database transaction

Processor calls, webhook emissions, and other downstream effects are never performed inside an open database transaction. Side effects are dispatched via a transactional outbox: a row written in the same transaction as the business state change, and published to an external system by an asynchronous worker with at-least-once delivery and idempotent handling (Richardson).

Two things go wrong when this invariant is violated. First, the external call holds database locks across the network, which produces amplified tail latency and lock contention. Second, the application enters an ambiguous state if the external call succeeds but the commit fails, or vice versa; this is the dual-write problem, and it is resolved only by the outbox pattern or an equivalent.

## I7. Intent is persisted before effect

Before any external side effect is attempted, a row representing the intent to perform that effect is durably committed. This row carries an idempotency key and initial state `pending`. On crash recovery, the recovery process reads pending intents and either resumes or compensates. No effect is ever performed whose intent we did not first durably record.

This is the payment-domain specialization of write-ahead logging. The broader principle is recoverability (Bernstein and Newcomer). The combination of I6 and I7 is structurally equivalent to the outbox pattern: an intent row and an outbox row are the same artifact.

## I8. Idempotency at every layer

Idempotency is not a single mechanism but a property enforced at every layer of the request path. In a typical payment system, four layers are independently defensible:

1. The client generates and sends a UUID idempotency key per logical request.
2. The server enforces uniqueness via a database constraint on `(merchant_id, idempotency_key)` and handles duplicates by replaying the original response.
3. The processor call passes the same idempotency key through, so that retries from a worker are safe at the processor boundary.
4. The webhook handler deduplicates on `(provider, event_id)` with its own unique constraint.

Each layer is independent. A failure in one layer does not compromise the others. This is the layered defense Helland advocates (2012).

## I9. Money is never floating point

Amounts are represented as fixed-precision decimals or integer minor units. Currency codes travel with every amount. Amounts are never compared or summed across currencies without explicit conversion at a recorded FX rate. Non-two-decimal currencies are respected: Japanese yen has zero decimals, Kuwaiti dinar has three.

IEEE 754 binary floating point cannot represent most decimal values exactly, and the accumulated rounding error in financial arithmetic is detectable by users and, eventually, by auditors. The practical convention is `Decimal` with an explicit scale, or integer cents.

## I10. Every state transition is reversible or explicitly terminal

If a state transition is reversible, the compensating action is defined in advance: authorization has authorization reversal, capture has refund. If a transition is terminal in the sense that no compensating action exists (a completed wire transfer past the return window, for example), the precondition for entering the terminal state is heightened: fraud checks, dual approval, or explicit user confirmation. No quiet one-way doors.

This is the sagas pattern applied to payment state machines: every forward transition has either a backward transition or an explicit statement that none exists (Garcia-Molina and Salem). The design discipline is to declare reversibility as a property of each transition, not to discover it retroactively.

## I11. Time is untrusted across systems

Wall-clock timestamps are suitable for human audit but not for resolving ordering across services. Cross-system ordering uses monotonic sequence numbers, processor-assigned references, or event identities. Clock skew, NTP drift, and leap seconds all produce subtle reordering bugs that are expensive to debug and embarrassing to explain.

## I12. PCI scope is declared and enforced at the boundary

Primary account numbers and card verification values never enter application code. Tokens alone cross the PCI boundary. The boundary itself is small, reviewed, and audited, while the rest of the system operates in a PCI-scope-reduced regime. Tokenization shifts scope; it does not eliminate it.

## Sources

- Pat Helland, _Idempotence Is Not a Medical Condition_, ACM Queue 10(4), 2012.
- Pat Helland, _Life Beyond Distributed Transactions: An Apostate's Opinion_, CIDR, 2007.
- Martin Kleppmann, _Designing Data-Intensive Applications_, O'Reilly, 2017.
- Hector Garcia-Molina and Kenneth Salem, _Sagas_, SIGMOD, 1987.
- Chris Richardson, _Pattern: Transactional Outbox_, microservices.io.
- Jim Gray and Andreas Reuter, _Transaction Processing: Concepts and Techniques_, Morgan Kaufmann, 1992.
- Philip Bernstein and Eric Newcomer, _Principles of Transaction Processing_, 2nd ed., Morgan Kaufmann, 2009.
