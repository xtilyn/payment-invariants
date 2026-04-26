# Anti-Patterns

Each anti-pattern is named, described, mapped to the invariant it violates, and given a structural fix. These are the most common implementation-level violations I see in payment-system review.

## 1. Check-then-act idempotency

**Violates:** I2.

```python
# anti-pattern
if not Payment.objects.filter(idempotency_key=key).exists():
    Payment.objects.create(idempotency_key=key, ...)
```

Two concurrent requests both observe no existing row, both proceed to create, and two rows are inserted. The transactional boundary does not save you: at standard isolation levels, two separate transactions will both see the absence of the row and both insert.

**Fix:** unique constraint plus caught `IntegrityError`.

```python
from django.db import transaction, IntegrityError

def create_or_get_intent(merchant_id, key, amount_cents, currency, token):
    try:
        with transaction.atomic():
            return PaymentIntent.objects.create(
                merchant_id=merchant_id,
                idempotency_key=key,
                amount_cents=amount_cents,
                currency=currency,
                payment_method_token=token,
            )
    except IntegrityError:
        return PaymentIntent.objects.get(
            merchant_id=merchant_id,
            idempotency_key=key,
        )
```

This pattern is also the canonical answer to "how do you guarantee a payment is processed exactly once." The honest answer: you guarantee _effectively-once_ via layered idempotency, not exactly-once delivery.

## 2. Dual-write without outbox

**Violates:** I6 and I7.

```python
# anti-pattern
payment = Payment.objects.create(...)
stripe.Charge.create(...)  # external call after DB write, no atomicity
```

If the database commit succeeds and the external call fails, the system has recorded an effect it did not produce. If the external call succeeds and the commit fails, the system has produced an effect it did not record. Either case requires manual reconciliation.

**Fix:** transactional outbox. Write the business state and an outbox row in the same transaction. A worker reads the outbox and performs the external effect with at-least-once delivery and idempotent handling.

```python
with transaction.atomic():
    payment = Payment.objects.create(...)
    OutboxEvent.objects.create(
        event_type="charge_request",
        payload={"payment_id": payment.id, "amount_cents": payment.amount_cents},
        idempotency_key=payment.idempotency_key,
    )

# worker, separately:
@shared_task
def process_outbox():
    with transaction.atomic():
        events = OutboxEvent.objects.select_for_update(
            skip_locked=True
        ).filter(processed_at__isnull=True)[:100]
        for event in events:
            stripe.Charge.create(
                idempotency_key=event.idempotency_key,
                **event.payload,
            )
            event.processed_at = timezone.now()
            event.save()
```

## 3. Direct balance mutation

**Violates:** I1 and I3.

```python
# anti-pattern
account.balance += amount
account.save()
```

Replaces an append-only journal with a mutable column. Eliminates both conservation (no audit of where the change came from) and audit (no record of the transition).

**Fix:** insert ledger entries; derive balance from the journal.

```python
with transaction.atomic():
    LedgerEntry.objects.create(
        account=account,
        amount_cents=amount_cents,
        direction="credit",
        reference=payment_id,
    )
    LedgerEntry.objects.create(
        account=clearing_account,
        amount_cents=amount_cents,
        direction="debit",
        reference=payment_id,
    )

# balance lookup:
def balance(account):
    return LedgerEntry.objects.filter(account=account).aggregate(
        total=Sum(Case(
            When(direction="credit", then=F("amount_cents")),
            When(direction="debit", then=-F("amount_cents")),
        ))
    )["total"] or 0
```

For performance, materialize a snapshot table that is rebuilt from the journal on a schedule.

## 4. Status as a mutable string

**Violates:** I3 and I10.

```python
# anti-pattern
class Payment(models.Model):
    status = models.CharField(max_length=20)  # "authorized", "captured", etc.
```

No transition graph enforced at the database layer. Two concurrent writers can drive the aggregate through an invalid sequence. There is no record of how state changed, only the current value.

**Fix:** explicit state machine plus event log. Either enforce transitions via a check constraint on `status`, or derive `status` from the event log (event-sourced reconstruction). The event log is the source of truth either way.

## 5. Retrying without an idempotency key

**Violates:** I2 and I8.

```python
# anti-pattern
for attempt in range(3):
    try:
        stripe.Charge.create(amount=amount, source=token)
        break
    except StripeError:
        continue
```

Each retry is a distinct request from the processor's perspective. The processor will charge the card multiple times.

**Fix:** generate the idempotency key once, persist it with the intent, pass it on every retry.

```python
key = payment_intent.idempotency_key  # generated once at intent creation
for attempt in range(3):
    try:
        stripe.Charge.create(
            amount=amount,
            source=token,
            idempotency_key=key,
        )
        break
    except StripeError:
        continue
```

## 6. Timestamp-based ordering across services

**Violates:** I11.

```python
# anti-pattern
events = ProcessorEvent.objects.filter(payment=p).order_by("created_at")
```

`created_at` is a wall-clock timestamp set by the writing service. Clock skew of a few milliseconds is enough to invert "obvious" orderings, and during incidents these reorderings compound.

**Fix:** use processor-assigned references or monotonic sequence numbers. If neither is available, use a hybrid logical clock or a per-aggregate version number incremented under the same row lock as the state transition.

## 7. Returning 500 on duplicate webhook

```python
# anti-pattern
def handle_webhook(event):
    if WebhookEvent.objects.filter(event_id=event.id).exists():
        return HttpResponse(status=500)  # tells processor to retry
    ...
```

Not a named-invariant violation, but operationally toxic. Processors interpret 500 as a transient failure and retry indefinitely, consuming both processor rate limits and your handler capacity.

**Fix:** return 200 on duplicate. Treat duplicate detection as the normal expected case.

```python
def handle_webhook(event):
    try:
        WebhookEvent.objects.create(event_id=event.id, payload=event.data)
    except IntegrityError:
        return HttpResponse(status=200)  # already handled, acknowledge
    process_event(event)
    return HttpResponse(status=200)
```

## 8. Storing floats for money

**Violates:** I9.

```python
# anti-pattern
class Payment(models.Model):
    amount = models.FloatField()
```

IEEE 754 binary floating point cannot exactly represent most decimal values. Accumulated rounding error is detectable by users. Detection is an embarrassment; correction is expensive.

**Fix:** integer minor units (`amount_cents = models.IntegerField()`) or a fixed-precision `Decimal` (`amount = models.DecimalField(max_digits=12, decimal_places=2)`). Carry the currency code on every amount so cross-currency arithmetic fails loudly rather than silently.
