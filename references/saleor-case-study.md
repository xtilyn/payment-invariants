# Case Study: Saleor Payment Module

[Saleor](https://github.com/saleor/saleor) is an open-source e-commerce platform written in Python and Django, with a payments subsystem that integrates multiple external processors including Stripe, Braintree, and Adyen. Saleor's payment module is a useful case study because it contains two generations of design in the same codebase, which lets us observe how invariant enforcement evolved.

## Legacy design

The legacy design is organized around a `Payment` model and a `Transaction` model.

The `Payment` model stores a mutable `captured_amount` column and a stringly-typed `charge_status` column with no database-level transition constraints. Legal transitions are enforced by read-time helper methods such as `can_authorize`, `can_capture`, and `can_refund`. The `Transaction` model has a Boolean `already_processed` flag used for application-level deduplication.

Mapped to the invariants:

- **I2** is approximated by the `already_processed` flag, which is a race-prone alternative to a unique constraint. Two concurrent webhooks observing `already_processed=False` will both proceed.
- **I1** is partially violated by the mutable `captured_amount` column. A capture event mutates the column rather than appending a posting, which means there is no audit of how the column reached its current value other than the `Transaction` log.
- **I10** is partially supported by the `can_*` helper methods but not enforced against concurrent writers.
- **I3** is conditionally satisfied: the `Transaction` log is append-only by convention, but `Payment` is mutable.

This is a "discipline plus reconciliation" approximation. It works as long as the team enforces the discipline; it accumulates inconsistencies under load.

## Modern design

The modern design is organized around a `TransactionItem` aggregate and a `TransactionEvent` event log.

`TransactionItem` enforces idempotency at the database level via a unique constraint on `(app_identifier, idempotency_key)`. Pending amounts for authorize, charge, refund, and cancel are tracked as explicit columns, separate from committed amounts, which gives reconciliation sufficient state to detect in-flight divergence. `TransactionEvent` carries its own independent idempotency constraint on `(transaction_id, idempotency_key)`, providing a second layer of duplicate protection for webhook deliveries.

The PSP reference (the processor's transaction identifier) is stored explicitly and indexed, enabling reconciliation joins without chasing foreign keys through the aggregate graph.

Mapped to the invariants:

- **I2** is now structurally enforced at two layers: `TransactionItem` and `TransactionEvent`.
- **I8** layers 2 and 4 are explicit. (Layers 1 (client) and 3 (processor call) live outside Saleor proper.)
- **I5** is enforced via row-level locks on `TransactionItem` during state transitions.

## What's still partial

Several invariants remain only partially enforced in the modern design:

- `TransactionItem` has a `modified_at` column, which means the row is mutable. Audit (**I3**) is delegated entirely to `TransactionEvent` rather than enforced at the `TransactionItem` level. If `TransactionItem` rows are updated and `TransactionEvent` rows are not also written, the audit gap is silent.
- `TransactionEvent` itself is append-only by convention rather than by database trigger. A buggy migration could update rows and silently break the audit.
- The state machine for legal transitions is enforced in application code in `payment/gateway.py` and `payment/lock_objects.py` rather than in database check constraints. **I10** is operational, not structural; a race between two writers can in principle produce an illegal transition, caught only by reconciliation.

## What it teaches

Saleor's modern design encodes most of the invariants here, with the remainder addressed operationally rather than structurally. This is consistent with the general pattern in production payment systems: combine structural enforcement (unique constraints, append-only tables) with operational enforcement (reconciliation, state-machine discipline) in a ratio that reflects the cost of structural enforcement in the chosen framework.

PostgreSQL check constraints on state transitions are possible but verbose. Django migrations make them hard to evolve. The Saleor team, like most teams, accepted the structural gap and closed it with reconciliation. This is a defensible trade-off, but it is a trade-off; the gap is real and worth naming.

The instructive lesson for using this skill on a real codebase: the invariants are useful even when not fully enforced. Naming them lets you observe where each is and is not enforced, and reason about the specific failure modes each gap admits. That is the diagnostic use of the methodology.

## Reading list for Saleor specifically

- [`saleor/payment/models.py`](https://github.com/saleor/saleor/blob/main/saleor/payment/models.py): both legacy and modern models live here.
- [`saleor/payment/gateway.py`](https://github.com/saleor/saleor/blob/main/saleor/payment/gateway.py): application-level transition logic.
- [`saleor/payment/lock_objects.py`](https://github.com/saleor/saleor/blob/main/saleor/payment/lock_objects.py): row-locking helpers for state transitions.
