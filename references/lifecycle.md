# The Payment Lifecycle as Invariant Preservation

Payment systems operate on a five-stage lifecycle: authorization, capture, clearing, settlement, and reconciliation. Each stage is a sequence of state transitions that must preserve a characteristic set of invariants. A dispute or chargeback is modeled as a parallel state machine that can affect already-settled transactions within a regulatory window, typically 120 days.

## 1. Authorization

The issuer places a hold on cardholder funds and returns an authorization code and network-level reference. Externally this is an ISO 8583 message through the acquirer to the network to the issuer; increasingly, ISO 20022 is replacing it for real-time rails.

Internally, the flow begins with a client request carrying an idempotency key. A `payment_intent` row is inserted with a unique constraint on `(merchant_id, idempotency_key)`. In the same database transaction, an outbox row is inserted. The transaction commits. A worker reads the outbox, calls the processor with the same idempotency key, and transitions the intent state from `pending` to `authorized` or `failed` under a row-level lock.

This stage enforces **I2** (via the unique constraint), **I7** (intent before effect), **I6** (outbox), and **I8** (key passthrough). The authorization response is the first durable commitment, and all downstream recovery behavior assumes that any intent reaching the worker has a committed row that predates any external call.

## 2. Capture

Capture instructs the acquirer to collect the previously-authorized amount. For card-present retail the capture is typically immediate; for card-not-present e-commerce, capture occurs at fulfillment, which may be hours or days after authorization.

Internally, capture is a state transition from `authorized` to `captured` on the intent, accompanied by an event row in the append-only log. If the capture call fails mid-flight, a recovery worker observes intents stuck in an intermediate state past a threshold and either retries or issues an authorization reversal. The capture call carries an idempotency reference derived from the intent; retries at the worker layer are safe at the processor boundary.

This stage is governed by **I10**: every pre-clearing transition must be reversible. After clearing, reversibility shifts to a different mechanism (the refund, which is a forward operation rather than a compensation).

## 3. Clearing

Clearing submits a batch of captured transactions to the networks for funding, typically on a per-acquirer daily cutoff. Internally, captures are aggregated into a `settlement_batch` row with an explicit membership and an append-only record of the submission. A mid-batch failure is a recovery event, not a silent drop.

This stage is governed by **I3**: batch submissions are append-only, with an explicit record of what was submitted and what was acknowledged.

## 4. Settlement

Funds move from issuer to acquirer to the merchant's funding account, and on a downstream schedule, to the seller's bank account. Standard settlement windows are T+1 or T+2; instant deposit products compress this at a fee.

Internally, settlement events arrive from the processor or acquirer in end-of-day batch reports. Each settlement posts as a double-entry pair across the funding account, merchant account, fees account, and any reserve accounts. The ledger is append-only; balances are materialized from the journal.

This stage is governed by **I1** and **I4**. The double-entry ledger enforces conservation; the processor's batch report is the source of truth for what actually settled. Internal state is advanced to match the processor's report, not the other way around.

## 5. Reconciliation

Reconciliation runs continuously and on batch windows. The canonical pattern is a three-way match: the internal ledger, the processor's end-of-day report, and the bank settlement record. Each side is indexed by a shared reference, typically the processor-assigned PSP reference. Divergences are bucketed into missing-locally, missing-remotely, amount-mismatch, and status-mismatch. Known-safe divergences are auto-healed; the rest are written to an exceptions table with severity and SLA for manual investigation.

This stage is governed by **I4** and **I1**. It is the mechanism by which I4 is operationally enforced; without reconciliation, the claim that the processor is the source of truth is only a hope. It is also the backstop for every other failure mode in the system: any failure that slips through the forward path eventually surfaces as a reconciliation exception.

## Disputes and chargebacks

A chargeback is a parallel state machine that can affect any already-settled transaction within the regulatory dispute window. The chargeback flow has its own state machine (initiated, evidence-submitted, resolved-merchant, resolved-cardholder), its own ledger postings (typically a debit reversal and a fee), and its own reconciliation pass against the processor's dispute reports.

Disputes interact with **I10** because they are forward operations on a transaction that the merchant considered terminal. The merchant's UI must distinguish between "settled" (terminal under normal flow) and "settled but disputable" (terminal-with-window). Conflating the two is a common source of operator confusion.
