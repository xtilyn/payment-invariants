# payment-invariants

A Claude Code skill that reviews payment-system code against nineteen invariants any production payment platform must preserve.

Most payment-system bugs (duplicate charges, ledger drift, stuck payments, reconciliation discrepancies, cross-tenant leaks, anonymous state changes, sanctions failures, runaway retries) map to a violated invariant that can be named precisely. This skill names them.

The set started at twelve and grew to nineteen as I reviewed more systems. The companion article is at [arjona.dev/writing/twelve-invariants-payment-systems](https://arjona.dev/writing/twelve-invariants-payment-systems).

## Install

Clone into your Claude Code skills directory:

```bash
git clone https://github.com/xtilyn/payment-invariants.git ~/.claude/skills/payment-invariants
```

The skill auto-triggers when you work on payment, billing, charge, refund, settlement, reconciliation, or webhook code. You can also invoke it explicitly by mentioning "payment invariants" in a message.

## What it does

Three modes, picked automatically based on what you're doing:

- **Design review.** Enumerate which of the nineteen invariants apply before discussing components. Each invariant must have a concrete enforcement mechanism, structural or operational (with a stated bound).
- **PR / code review.** Walk a diff and flag invariant violations by number. Cite the matching anti-pattern. Recommend the structural fix.
- **Incident debugging.** Identify which invariant a failure mode violates, then recommend the structural fix instead of a workaround.

Each invariant is tagged `[Safety]` (must hold at every committed state, unit-testable) or `[Liveness]` (must hold eventually within a stated bound, monitored as an SLO). A few span both. The bound is part of the invariant; "eventual" without one is a hope.

## The nineteen invariants

| #   | Invariant                                                                   | Tag             | One-line                                                              |
| --- | --------------------------------------------------------------------------- | --------------- | --------------------------------------------------------------------- |
| I1  | Conservation of money                                                       | Safety          | Sum of debits equals sum of credits, always; balances derived from journal |
| I2  | At-most-once terminal effect                                                | Safety          | Identity is `(actor_id, idempotency_key)`; enforce with a unique constraint |
| I3  | Monotonic, append-only audit                                                | Safety          | Hybrid OK: status as cache, events as truth on disagreement           |
| I4  | Processor is source of truth; mirror converges via bounded reconciliation   | Liveness        | "Eventual" needs a stated bound (detect + page windows)               |
| I5  | Single-writer per aggregate; eventual consistency across aggregates         | Safety + Liveness | `SELECT FOR UPDATE` = serializability via 2PL, not linearizability   |
| I6  | Effects dispatched via a durable queue, not from inside a DB transaction    | Safety          | Strict outbox or pragmatic on-commit + reconciliation backstop        |
| I7  | Intent persisted before effect; recovery completes pending intents          | Safety + Liveness | Commit `pending` row before any external call; bound the recovery   |
| I8  | Idempotency at every layer                                                  | Safety          | Independent enforcement at client, DB, processor, webhook             |
| I9  | Money is never floating point                                               | Safety          | Integer minor units or fixed-precision decimal; currency travels      |
| I10 | Every transition is reversible or explicitly terminal                        | Safety          | No quiet one-way doors; multi-step workflows are explicit sagas       |
| I11 | Time is untrusted across systems                                            | Safety          | Sequence numbers and processor refs, not wall-clock timestamps        |
| I12 | PCI scope declared and enforced at the boundary                             | Safety          | PANs / CVVs never enter app code; know your SAQ (A vs A-EP)           |
| I13 | Inbound webhooks cryptographically authenticated before persist             | Safety          | HMAC + timestamp window + replay protection; without it, I4 has a hole |
| I14 | Tenant isolation                                                            | Safety          | Every read and write scoped to a tenant; cross-tenant existence returns 404 |
| I15 | OFAC / sanctions screening before money moves                                | Safety          | Counterparty cleared as a precondition, not a post-hoc check          |
| I16 | Every state transition has an authenticated, recorded principal              | Safety          | `actor_type` + `actor_id` + role-at-time on every mutating record     |
| I17 | Bounded retries with explicit budgets                                       | Liveness        | Max attempts, backoff schedule, dead-letter destination               |
| I18 | Velocity and per-period limits                                              | Safety          | Per-account, per-day, per-counterparty checks at the app boundary     |
| I19 | Cross-aggregate causality                                                   | Safety          | FK + state precondition + idempotent application; no orphan reversals |

Full statements with literature in [`references/twelve-invariants.md`](references/twelve-invariants.md). (The filename stays `twelve-invariants.md` for URL stability; content covers all nineteen.)

## What it doesn't cover

Fraud detection (beyond the velocity-limit framing in I18), tax handling, currency conversion at scale, chargeback workflow beyond a two-paragraph summary, and the organizational side of running a payments team. These are essential to a real platform; they are out of scope for an invariant-driven correctness review.

## Repo structure

```
.
├── SKILL.md                        the body Claude loads
├── README.md                       this file
├── LICENSE                         MIT
└── references/
    ├── twelve-invariants.md        I1-I19 expanded (filename kept stable)
    ├── lifecycle.md                five-stage payment lifecycle
    ├── anti-patterns.md            anti-pattern catalog with fixes
    └── saleor-case-study.md        worked example: Saleor payment module (illustrates I1-I12)
```

## Sources

The methodology synthesizes established transaction-processing literature plus security and compliance references for the payment-systems specialization:

- Pat Helland, [_Idempotence Is Not a Medical Condition_](https://queue.acm.org/detail.cfm?id=2187821) (ACM Queue, 2012). Cite for I2, I8.
- Pat Helland, _Life Beyond Distributed Transactions: An Apostate's Opinion_ (CIDR, 2007). Cite for I4, I6.
- Martin Kleppmann, [_Designing Data-Intensive Applications_](https://dataintensive.net/) (O'Reilly, 2017). Ch. 5 (causality, I19), Ch. 6 (partitioning, I14), Ch. 7 (transactions / 2PL, I5), Ch. 9 (consistency, linearizability vs serializability, I4 / I5).
- Hector Garcia-Molina and Kenneth Salem, _Sagas_ (SIGMOD 1987). Cite for I10.
- Chris Richardson, [_Pattern: Transactional Outbox_](https://microservices.io/patterns/data/transactional-outbox.html). Cite for I6, I7.
- Jim Gray and Andreas Reuter, _Transaction Processing: Concepts and Techniques_ (Morgan Kaufmann, 1992). Cite for I1, I7.
- Philip Bernstein and Eric Newcomer, _Principles of Transaction Processing_, 2nd ed. (Morgan Kaufmann, 2009). Cite for I1, I7.
- Marc Brooker, [_Exponential Backoff and Jitter_](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) (AWS Architecture Blog, 2015). Cite for I17.
- OWASP, [_API Security Top 10_](https://owasp.org/API-Security/editions/2023/en/0x11-t10/) (2023). Cite for I13.
- NIST SP 800-53, AU-3 (Audit Record Content). Cite for I16.
- OFAC SDN List (treasury.gov); FinCEN BSA/AML guidance, [31 CFR § 1010](https://www.ecfr.gov/current/title-31/subtitle-B/chapter-X/part-1010) and § 1020. Cite for I15, I18.
- CWE-639 (Authorization Bypass Through User-Controlled Key). Cite for I14.

## License

MIT. See [LICENSE](LICENSE).
