# The Invariants

A working set of invariants that, if preserved, is sufficient to catch the most common classes of payment-domain bugs. The set started at twelve and grew to nineteen as I reviewed more systems. The filename stays `twelve-invariants.md` for URL stability; the content is the current set.

Each invariant is tagged `[Safety]` or `[Liveness]`. A few span both:

- **Safety**: must hold at every committed state. Violation = correctness bug, data loss, or money loss. Unit-testable.
- **Liveness**: must hold eventually within a stated bound. Violation = degraded service, recoverable. Tested with chaos drills, monitored as SLOs.

## I1. Conservation of money. [Safety]

Every journal posting is a balanced set of debits and credits; the journal is in balance after every committed transaction. State changes that don't move money (e.g. `scheduled` to `dispatching`) post nothing. The balanced-pair claim is per posting, not per state change. Materialized balances are derived from the append-only journal and can always be recomputed from it. If a materialized balance diverges from the journal's fold, the journal wins.

A system designed around a mutable balance column is structurally incapable of preserving I1 under concurrency. The canonical implementation is double-entry accounting: every transaction is represented as a pair of postings, each to a distinct account, with opposite signs. Balances are derived by summing postings.

Classical transaction processing literature treats journaling as a durability mechanism (Gray and Reuter; Bernstein and Newcomer). In payment systems it is additionally an audit and conservation mechanism.

## I2. At-most-once terminal effect per logical request. [Safety]

Given a request identity `(actor_id, idempotency_key)`, arbitrarily many retries must produce at most one successful terminal effect. This is the defining correctness property of charge operations: a duplicate charge is worse than a failed charge.

Exactly-once semantics, in the delivery sense, is provably unattainable in an asynchronous system with failures. What is attainable is _effectively-once_ semantics, built from at-least-once delivery composed with idempotent consumers (Helland 2012). The implementation layer to rely on is a database unique constraint, with duplicate-key errors caught and mapped to an idempotent replay of the original response.

The idempotency key is generated once at intent creation and persisted on the row. Never regenerated per execute attempt. If the executor crashes after the processor accepts the request but before the local DB write commits, the recovery worker reruns with the same key; the processor returns a conflict; the executor reads the existing resource and reconciles.

## I3. Monotonic, append-only audit. [Safety]

Every state transition produces a new row in an immutable event log. Nothing is mutated in place; nothing is deleted. The current state of any aggregate _could be reconstructed_ by folding its event stream.

In practice, most production systems run a hybrid: current state is also stored on the aggregate row for query speed. The mutable `status` column is a cache of the latest event; the event log is the source of truth on disagreement. Pure event sourcing adds latency for every read; the hybrid pays storage cost for fast reads plus auditable history. The invariant is that the cache _can be derived_ from the events, not that it is re-derived on every read.

The most common violation in practice is a system that stores current state in a mutable status column and treats the event log as a supplementary artifact rather than the authority on disagreement. When the two diverge under concurrency, the log ceases to function as a source of truth.

## I4. Processor is the source of truth; mirror converges via bounded reconciliation. [Liveness]

The internal database is a cache of processor state, not an independent ledger of external reality. On any divergence between the local mirror and the processor's state, the processor wins and a reconciliation process closes the gap. Settlement is never decided based on internal state alone.

**The bound is the invariant.** "Eventual" without bounds is a hope, not a guarantee. Concrete bounds for a typical reconciliation system: drift between local mirror and processor detected within ≤ 60 minutes (hourly reconciler tick); unresolved drift older than 4 hours pages on-call; drift on > 1% of accounts in any single pass also pages, because the drift rate is itself a health metric.

This invariant captures the structural fact that distinguishes payment systems from purely internal distributed systems. An order management system can define order state arbitrarily because it owns the order. A payment system cannot define settlement state arbitrarily because settlement is an externally-observable fact about money movement that we neither originate nor control.

## I5. Single-writer per aggregate; eventual consistency across aggregates. [Safety + Liveness]

All mutations on a single aggregate (e.g. one payment identity) are serially ordered through row-level locks via `SELECT ... FOR UPDATE`. This gives **serializability for that aggregate via two-phase locking**, strictly weaker than linearizability (which is about real-time ordering across operations on a single object), but indistinguishable from it for the typical single-row contention pattern.

Cross-aggregate effects (notifications, analytics, ledger postings into other accounts) are asynchronous with bounded lag.

A precision note: it is common to see `SELECT FOR UPDATE` described as providing linearizability. That is imprecise. Linearizability is a real-time-ordering property over operations on a single object; 2PL gives serializability over the transactions that touched the locked rows. Both are useful framings, but the right word in code review is "serializability via 2PL." Kleppmann (Ch. 9) is the place to look up the formal distinction.

The aggregate here is in the domain-driven sense: a consistency boundary. For a payment, the boundary is the payment identity and its owned events. For a merchant balance, the boundary is the account. Linearizability across aggregates is both unnecessary and expensive, and attempting it via distributed transactions imports the well-known availability costs of two-phase commit (Helland 2007).

## I6. Effects dispatched via a durable queue, not from inside a DB transaction. [Safety]

Processor calls, outbound webhooks, email, and Slack pings never happen inside an open `transaction.atomic()` block (or whatever the framework's equivalent is). Holding row locks across an external roundtrip blocks the connection pool. Performing the external call before the commit creates the classic dual-write problem: DB updated and side effect lost on crash, or side effect succeeded and DB rolled back.

There are two implementations, with different operational properties:

**Strict form (transactional outbox).** Write the message to an `outbox` DB table inside the transaction; a separate poller drains it. The DB commit is the message commit; broker liveness is irrelevant. The strongest form, with no message loss window. Reference: Richardson, _Transactional Outbox Pattern_.

**Pragmatic form (on-commit enqueue + reconciliation backstop).** Persist the durable _intent_ row inside the transaction (a `WebhookEvent`, a payment-intent row, an outbound message row, etc.). After commit, enqueue the worker via the framework's `transaction.on_commit` hook. The broker is a separate system, so a window exists between DB commit and broker accept where a process kill loses the enqueue. This is acceptable when reconciliation finds and reapplies anything the broker dropped: the inbound durable row exists, and a sweeper task re-enqueues by status. Loss is bounded, not invisible.

**When to upgrade to strict.** Pure outbound effects where the durable intent isn't pre-persisted somewhere reconcilable. For one-shot operational pings or non-money emails, the pragmatic form is fine.

## I7. Intent is persisted before effect; recovery completes pending intents. [Safety + Liveness]

**Safety side.** Before any external side effect is attempted, a row representing the intent to perform that effect is durably committed. This row carries an idempotency key and initial state `pending`. No effect is ever performed whose intent we did not first durably record.

**Liveness side.** On crash recovery, the recovery process sees the pending intent and resumes or compensates within a stated bound. A reasonable default for in-flight rows is 5 minutes; without a bound, "recovery completes" is a hope, not an invariant.

This is the payment-domain specialization of write-ahead logging. The broader principle is recoverability (Bernstein and Newcomer). The combination of I6 and I7 is structurally equivalent to the outbox pattern: an intent row and an outbox row are the same artifact.

The antipattern is calling the processor first and writing the row from the response. That loses the intent on crash, can't safely retry, and ships duplicate charges under specific failure modes.

## I8. Idempotency at every layer. [Safety]

Idempotency is enforced as four independent layers, each defending the same invariant on its own:

1. **Client** generates a UUID `Idempotency-Key` per logical request and sends it on every retry. Missing key on a write request is a 400.
2. **Server** has a unique constraint on `(actor_id, idempotency_key)` so duplicate requests collapse to one logical operation. Duplicates return the existing resource via a caught `IntegrityError`.
3. **Processor** call passes the same key through; the processor returns the same resource on retry.
4. **Webhook** handler dedupes on `(provider, event_id)` with its own unique constraint so retries from the processor are idempotent on the receiving side too.

Each layer is independently sufficient on the happy path; together they survive crashes at any step. This is the layered defense Helland advocates (2012).

**Domain uniqueness is a separate invariant; don't conflate.** A constraint like "one active scheduled payment per bill" prevents two distinct user actions from creating conflicting state, but it is not idempotency. Both belong; both should be named separately.

## I9. Money is never floating point. [Safety]

Amounts are represented as integer minor units (or fixed-precision `Decimal`); currency codes travel with every amount. Amounts are never compared or summed across currencies without explicit conversion at a recorded FX rate. Non-two-decimal currencies are respected: Japanese yen has zero decimals, Kuwaiti dinar has three.

IEEE 754 binary floating point cannot represent most decimal values exactly, and the accumulated rounding error in financial arithmetic is detectable by users and, eventually, by auditors. A `FloatField` for money in code review is reject-on-sight.

## I10. Every state transition is reversible or explicitly terminal; multi-step workflows are explicit sagas. [Safety]

**Per-aggregate.** If a transition is reversible, the compensating action is defined: authorization has authorization reversal, capture has refund, sent has returned (within the ACH return window, typically 60 days). If terminal in the sense that no compensating action exists (a completed wire transfer past the return window, for example), the precondition for entering the terminal state is heightened: fraud check, dual approval, or explicit user confirmation. No quiet one-way doors.

**Across aggregates / multi-step external workflows.** If a workflow has more than one external effect (an FX-converted payment, an ACH-then-card-fee, a KYB-application-then-account-create), the saga is explicit. Each step has a defined forward action and a defined compensation, and partial completion is a defined state, not "we'll figure it out." The Garcia-Molina and Salem (1987) saga formalism applies; an unhandled mid-workflow failure is a bug, not an unknown unknown.

The design discipline is to declare reversibility as a property of each transition, not to discover it retroactively.

## I11. Time is untrusted across systems. [Safety]

Timestamps are for human audit, not for ordering. Cross-system ordering uses monotonic transaction IDs or causal references (event IDs, processor refs). Never resolve a race by comparing `created_at` across services.

The concrete failure mode: webhooks may deliver `payment.returned` before `payment.sent` if retries reorder them on the network. The handler must not write `if event.received_at > payment.updated_at: apply()`. Instead: `if new_status reachable from payment.status by the state machine: apply()`, and let reconciliation correct rare ordering issues (per I4).

Clock skew, NTP drift, and leap seconds all produce subtle reordering bugs that are expensive to debug.

## I12. PCI scope is declared and enforced at the boundary. [Safety]

Primary account numbers and card verification values never enter application code. Tokens alone cross the PCI boundary. The boundary itself is small, reviewed, and audited; the rest of the system operates in a PCI-scope-reduced regime. Tokenization shifts scope; it does not eliminate it.

**Which SAQ.** Two SAQs are relevant for an iframe-based architecture:

- **SAQ A** applies if the cardholder data iframe is hosted entirely on the PSP's domain and the merchant page only embeds or redirects to it. Lightest controls.
- **SAQ A-EP** applies if the merchant page controls the surrounding context (e.g. JS manipulates the iframe's URL or the DOM around it at runtime). Requires quarterly ASV scans plus several additional controls. Meaningfully more onerous.

The architecture decides which SAQ gets filed. A hosted card-display iframe with no runtime manipulation by the surrounding page is achievable as SAQ A; A-EP kicks in the moment the surrounding page manipulates the iframe. Worth confirming with a QSA before assuming the cheaper SAQ.

## I13. Inbound webhooks are cryptographically authenticated before persist. [Safety]

Webhook payloads are verified (HMAC over the raw request body using a shared secret, timestamp checked against a clock-skew window, replay-protected via `event_id` uniqueness) _before_ the payload is persisted, let alone applied. Without this, I4 has a hole: anyone who can hit the webhook URL can claim to be the processor.

Implementation notes:

- Read the raw request body, not the parsed body. Parsing alters whitespace and breaks the HMAC.
- Use a constant-time comparison (e.g. `hmac.compare_digest`) to avoid timing side channels.
- The unique constraint on `event_id` provides replay protection. Re-delivery returns 200 (the duplicate detection from I8 layer 4).

Reference: OWASP API Security Top 10 (2023), API2 Broken Authentication. Standard pattern across Stripe, Adyen, Plaid.

## I14. Tenant isolation. [Safety]

Every read and every write is scoped to a tenant. Cross-tenant references are explicit, checked at the application boundary, and audited. There is no global query path that returns rows from more than one tenant unless the caller is a system actor with explicit cross-tenant privilege and the action is logged.

The canonical multi-tenant breach class is "Authorization Bypass Through User-Controlled Key" (CWE-639): an endpoint accepts an identifier from the request and looks it up without scoping to the requesting tenant. A request with another tenant's identifier should return 404, not 403, so the system does not even leak existence.

Patterns that enforce I14 structurally:

- Every model that holds tenant data carries a `tenant` foreign key.
- Every API view's base queryset starts with `.filter(tenant=request.user.tenant)` (or framework equivalent).
- Tenant scoping is applied before role scoping.
- Cross-tenant administrative operations bypass the filter explicitly and write to an audit log with an explicit `actor_kind`.

Reference: Kleppmann, _DDIA_, Ch. 6 (Partitioning).

## I15. OFAC / sanctions screening before money moves. [Safety]

For US fintechs and any platform moving money across US-touched rails, every counterparty is screened against the OFAC SDN list before funds can be sent or received. Screening is a precondition on the state transition, not a post-hoc check. Hits are quarantined and routed to compliance; the user cannot override.

Defense in depth: the processor typically also screens at their layer, but local enforcement makes rejection fast and the audit log shows a deliberate decision rather than a processor-side mystery 4xx. A blocked counterparty routes the request to a compliance queue; the user sees "under review," never "denied," to avoid tipping off in case of a true positive.

References: OFAC SDN List (treasury.gov); FinCEN BSA/AML guidance, 31 CFR § 1010.

## I16. Every state transition has an authenticated, recorded principal. [Safety]

No transition is anonymous. The mutating record carries `actor_type` (user / system / webhook / scheduled job), `actor_id`, and the authorization check that permitted the transition. "The system did it" is never an acceptable answer to "who initiated this?" There is always a system actor identity (e.g. `webhook:provider:event_<id>`, `cron:reconcile_accounts`, `worker:dispatch_due:<host>:<pid>`).

For user-initiated actions, record the role at the time of action, not just the user identity. Roles change over time; the audit log should preserve what the user _was_ when they acted.

References: NIST SP 800-53 AU-3 (Audit Record Content); SOX § 404 internal-control requirements for financial systems.

## I17. Bounded retries with explicit budgets. [Liveness]

Every retry loop has a max attempt count, a backoff schedule, and a defined dead-letter destination. "Retry forever" amplifies downstream incidents: an unbounded retry against a flapping dependency is the mechanism that turns a 5-minute incident into a 5-hour outage. Budgets live in code, not in a runbook.

A reasonable default for a payment-execution worker: max 5 attempts, exponential backoff capped at ~10 minutes (`countdown=min(2 ** retries * 30, 600)`). After exhaustion, the row goes to `failed` with a populated `failure_reason`. A daily report surfaces stuck rows for ops review.

Continuous reconciliation jobs typically need no in-loop retry: the next tick covers a missed one. Bounded retries are for in-flight forward operations that can fail mid-step.

Reference: Marc Brooker, _Exponential Backoff and Jitter_, AWS Architecture Blog (2015).

## I18. Velocity and per-period limits. [Safety]

No payment leaves the system without passing per-account, per-day, per-counterparty, and per-period velocity checks. Limits are enforced at the application boundary, not delegated entirely to the processor.

The reasons are operational and regulatory. Operational: the blast radius of a compromised credential is whatever fits inside the daily ceiling. Regulatory: BSA/AML expects monitoring; a visible local policy beats an inferred policy that lives only at the processor.

Patterns:

- A `velocity_rule` table holds rows like `(scope, scope_id, window_seconds, max_amount_cents, max_count)`.
- The originating endpoint queries the relevant rules in a single transaction, sums in-window committed payments, and rejects with 429 + reason if any rule is breached.
- A separate per-counterparty rule catches the "scammed bookkeeper" case: a single new vendor receiving an outsized share of the day's spend.

References: BSA / AML, Title 31 CFR § 1020 (BSA Compliance Program); FinCEN Suspicious Activity Report thresholds.

## I19. Cross-aggregate causality. [Safety]

A refund references a settled payment; a reversal references an authorized capture; a return references a sent ACH; a chargeback references a captured charge. The state machine encodes per-aggregate transitions; cross-aggregate causality is enforced by foreign key + state precondition + idempotent application. Orphan refunds and reversals-without-parent are correctness bugs, not edge cases.

Patterns that enforce I19:

- The dependent row carries a foreign key to the parent (`payment_id`, `capture_id`, etc.) with `on_delete=PROTECT`.
- A check constraint or application-level guard requires the parent's status to be in a valid set (e.g. a refund's parent payment must be `sent` or `returned`).
- Webhook handlers that arrive out of order must look up the parent and refuse to apply if it is missing; requeue with backoff (per I8 layer 4) rather than create an orphan.
- Amount preconditions are explicit: a refund larger than the parent's amount returns an explicit 422 with a reason, not a silent clamp.

Reference: Kleppmann, _DDIA_, Ch. 5 (causality, version vectors, happens-before).

## Sources

- Pat Helland, [_Idempotence Is Not a Medical Condition_](https://queue.acm.org/detail.cfm?id=2187821), ACM Queue 10(4), 2012. Cite for I2, I8.
- Pat Helland, _Life Beyond Distributed Transactions: An Apostate's Opinion_, CIDR 2007. Cite for I4, I6 (why not 2PC across heterogeneous systems).
- Martin Kleppmann, [_Designing Data-Intensive Applications_](https://dataintensive.net/), O'Reilly, 2017.
  - Ch. 5 (Replication, causality, version vectors). Cite for I19.
  - Ch. 6 (Partitioning). Cite for I14.
  - Ch. 7 (Transactions, serializability via 2PL, write skew, isolation anomalies). Cite for I5.
  - Ch. 9 (Consistency and consensus, linearizability vs. serializability). Cite for I4, I5; the right place to be precise about the linearizability vs. serializability distinction.
- Hector Garcia-Molina and Kenneth Salem, _Sagas_, SIGMOD 1987. Cite for I10, especially the multi-step external workflow clause.
- Chris Richardson, [_Transactional Outbox Pattern_](https://microservices.io/patterns/data/transactional-outbox.html). Cite for I6 (strict form) and I7.
- Jim Gray and Andreas Reuter, _Transaction Processing: Concepts and Techniques_, Morgan Kaufmann, 1992. Cite for I1, I7 (ACID, durability, write-ahead logging).
- Philip Bernstein and Eric Newcomer, _Principles of Transaction Processing_, 2nd ed., Morgan Kaufmann, 2009. Cite for I1, I7 (recoverability, transaction manager design).
- Marc Brooker, _Exponential Backoff and Jitter_, AWS Architecture Blog (2015). Cite for I17.
- OWASP, _API Security Top 10_ (2023). Cite for I13.
- NIST SP 800-53, AU-3 (Audit Record Content). Cite for I16.
- OFAC SDN List (treasury.gov); FinCEN BSA/AML guidance, 31 CFR § 1010 / § 1020. Cite for I15, I18.
- CWE-639 (Authorization Bypass Through User-Controlled Key). Cite for I14.
