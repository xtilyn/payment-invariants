# payment-invariants

A Claude Code skill that reviews payment-system code against twelve invariants any production payment platform must preserve.

Most payment-system bugs (duplicate charges, ledger drift, stuck payments, reconciliation discrepancies) map to a violated invariant that can be named precisely. This skill names them.

The companion article is at [arjona.dev/writing/twelve-invariants-payment-systems](https://arjona.dev/writing/twelve-invariants-payment-systems).

## Install

Clone into your Claude Code skills directory:

```bash
git clone https://github.com/xtilyn/payment-invariants.git ~/.claude/skills/payment-invariants
```

The skill auto-triggers when you work on payment, billing, charge, refund, settlement, reconciliation, or webhook code. You can also invoke it explicitly by mentioning "payment invariants" in a message.

## What it does

Three modes, picked automatically based on what you're doing:

- **Design review.** Enumerate which of the twelve invariants apply before discussing components. Each invariant must have a concrete enforcement mechanism, structural or operational.
- **PR / code review.** Walk a diff and flag invariant violations by number. Cite the matching anti-pattern. Recommend the structural fix.
- **Incident debugging.** Identify which invariant a failure mode violates, then recommend the structural fix instead of a workaround.

## The twelve invariants

| #   | Invariant                                            | One-line                                                  |
| --- | ---------------------------------------------------- | --------------------------------------------------------- |
| I1  | Conservation of money                                | Sum of debits equals sum of credits, always               |
| I2  | At-most-once terminal effect                         | Retries produce one effect; enforce with a unique constraint |
| I3  | Monotonic, append-only audit                         | State transitions are new rows, never mutations           |
| I4  | Processor is the source of truth for settlement      | Internal DB is a cache; reconcile on divergence           |
| I5  | Linearizability per aggregate                        | Row locks within; eventual consistency across             |
| I6  | No external call inside a database transaction       | Use a transactional outbox                                |
| I7  | Intent persisted before effect                       | Commit a `pending` row before any external call           |
| I8  | Idempotency at every layer                           | Independent enforcement at client, DB, processor, webhook |
| I9  | Money is never floating point                        | Decimal or integer minor units; currency code travels     |
| I10 | Every transition is reversible or explicitly terminal | No quiet one-way doors                                    |
| I11 | Time is untrusted across systems                     | Use sequence numbers, not wall-clock timestamps           |
| I12 | PCI scope declared and enforced at the boundary      | PANs / CVVs never enter application code                  |

Full statements with literature in [`references/twelve-invariants.md`](references/twelve-invariants.md).

## What it doesn't cover

Fraud detection, regulatory compliance (PSD2, SCA, OFAC), tax, currency conversion at scale, chargeback workflow beyond a two-paragraph summary, and the organizational side of running a payments team. These are essential to a real platform; they are out of scope for an invariant-driven correctness review.

## Repo structure

```
.
├── SKILL.md                        the body Claude loads
├── README.md                       this file
├── LICENSE                         MIT
└── references/
    ├── twelve-invariants.md        I1-I12 expanded
    ├── lifecycle.md                five-stage payment lifecycle
    ├── anti-patterns.md            anti-pattern catalog with fixes
    └── saleor-case-study.md        worked example: Saleor payment module
```

## Sources

The methodology synthesizes established transaction-processing literature for the payment-systems specialization:

- Pat Helland, [_Idempotence Is Not a Medical Condition_](https://queue.acm.org/detail.cfm?id=2187821) (ACM Queue, 2012)
- Pat Helland, _Life Beyond Distributed Transactions: An Apostate's Opinion_ (CIDR, 2007)
- Martin Kleppmann, [_Designing Data-Intensive Applications_](https://dataintensive.net/) (O'Reilly, 2017)
- Hector Garcia-Molina and Kenneth Salem, _Sagas_ (SIGMOD, 1987)
- Chris Richardson, [_Pattern: Transactional Outbox_](https://microservices.io/patterns/data/transactional-outbox.html)
- Jim Gray and Andreas Reuter, _Transaction Processing: Concepts and Techniques_ (Morgan Kaufmann, 1992)
- Philip Bernstein and Eric Newcomer, _Principles of Transaction Processing_, 2nd ed. (Morgan Kaufmann, 2009)

## License

MIT. See [LICENSE](LICENSE).
