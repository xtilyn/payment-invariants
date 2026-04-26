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

## 9. Webhook applied without HMAC verification

**Violates:** I13.

```python
# anti-pattern
@csrf_exempt
def webhook(request):
    payload = json.loads(request.body)
    apply_event(payload)  # trusts whoever called us
    return HttpResponse(200)
```

Anyone who knows the URL becomes "the processor." All of I4 collapses, because the local mirror is updated based on unauthenticated input.

**Fix:** verify the HMAC over the raw request body before persisting or applying anything. Use a constant-time comparison.

```python
import hmac
import hashlib

@csrf_exempt
def webhook(request):
    body = request.body  # raw bytes; do not parse first, parsing breaks HMAC
    sig = request.headers.get("X-Provider-Signature", "")
    expected = hmac.new(
        WEBHOOK_SECRET.encode(),
        body,
        hashlib.sha256,
    ).hexdigest()
    if not hmac.compare_digest(sig, expected):
        return HttpResponse(401)

    # also check the timestamp window to prevent replay
    ts = int(request.headers.get("X-Provider-Timestamp", "0"))
    if abs(time.time() - ts) > 300:  # 5-minute window
        return HttpResponse(401)

    payload = json.loads(body)
    try:
        WebhookEvent.objects.create(event_id=payload["event_id"], payload=payload)
    except IntegrityError:
        return HttpResponse(200)  # replay; handled per anti-pattern 7
    apply_event_async.delay(payload["event_id"])
    return HttpResponse(200)
```

## 10. Cross-tenant query path

**Violates:** I14.

```python
# anti-pattern
def get_bill(request, bill_id):
    bill = Bill.objects.get(id=bill_id)  # no tenant filter
    return JsonResponse(bill.to_dict())
```

A user in tenant A passes a `bill_id` belonging to tenant B and gets it back. This is CWE-639, the canonical multi-tenant breach class.

**Fix:** every queryset starts with a tenant filter. Apply tenant scoping before role scoping. Return 404 (not 403) on cross-tenant access so existence is not leaked.

```python
def get_bill(request, bill_id):
    try:
        bill = Bill.objects.get(id=bill_id, tenant=request.user.tenant)
    except Bill.DoesNotExist:
        return HttpResponse(status=404)
    return JsonResponse(bill.to_dict())
```

For DRF or similar frameworks, enforce this in the base `get_queryset`:

```python
class TenantScopedViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        return super().get_queryset().filter(tenant=self.request.user.tenant)
```

Cross-tenant administrative operations bypass the filter explicitly and write to an audit log.

## 11. Payment originated without OFAC clearance check

**Violates:** I15.

```python
# anti-pattern
def schedule_payment(request):
    payment = Payment.objects.create(
        bill=bill,
        amount_cents=bill.amount_cents,
    )
    enqueue_send.delay(payment.id)
    return JsonResponse(payment.to_dict())
```

For US-touched rails, every counterparty must be screened against the OFAC SDN list as a precondition on the state transition. Post-hoc screening (or relying solely on the processor's screening) is non-compliant and operationally noisy.

**Fix:** screening status is a precondition; reject with 422 if not cleared.

```python
def schedule_payment(request):
    counterparty = bill.vendor.counterparty
    if counterparty.screening_status != "cleared":
        return JsonResponse(
            {"error": "counterparty under review"},  # do not tip off
            status=422,
        )
    payment = Payment.objects.create(...)
    enqueue_send.delay(payment.id)
    return JsonResponse(payment.to_dict())
```

A blocked counterparty routes to a compliance queue. The user sees "under review," never "denied," to avoid tipping off in case of a true positive.

## 12. State transition with no recorded actor

**Violates:** I16.

```python
# anti-pattern
def mark_sent(payment):
    payment.status = "sent"
    payment.save()
    PaymentEvent.objects.create(payment=payment, status="sent")
```

Six months later, an auditor asks "who marked this as sent?" There is no answer. "The system" is not an acceptable answer.

**Fix:** every mutating record carries `actor_type`, `actor_id`, and (for user actions) the role at the time of the action.

```python
def mark_sent(payment, actor_type, actor_id, role=None):
    with transaction.atomic():
        payment.status = "sent"
        payment.save()
        PaymentEvent.objects.create(
            payment=payment,
            from_status="dispatching",
            to_status="sent",
            actor_type=actor_type,    # "user" | "system" | "webhook" | "scheduled_job"
            actor_id=actor_id,         # e.g. "user:42", "webhook:provider:evt_xyz", "worker:dispatch:host:pid"
            metadata={"role": role} if role else {},
        )
```

System actors get explicit identities: `worker:<job_name>:<host>:<pid>`, `webhook:<provider>:<event_id>`, `cron:<job_name>`. Roles are recorded as they were at the time of action because roles change over time.

## 13. Retry loop without `max_retries` or dead-letter

**Violates:** I17.

```python
# anti-pattern
@shared_task(bind=True)
def call_processor(self, payment_id):
    try:
        processor.charge(payment_id)
    except ProcessorError:
        raise self.retry(countdown=60)  # forever
```

A flapping dependency keeps the queue saturated. A 5-minute downstream incident becomes a 5-hour outage as the retry pile compounds.

**Fix:** declare the budget in code, with backoff and a dead-letter destination.

```python
@shared_task(
    bind=True,
    max_retries=5,
    autoretry_for=(ProcessorError,),
    retry_backoff=True,
    retry_backoff_max=600,
    retry_jitter=True,
)
def call_processor(self, payment_id):
    try:
        processor.charge(payment_id)
    except ProcessorError:
        if self.request.retries >= self.max_retries:
            payment = Payment.objects.get(id=payment_id)
            payment.status = "failed"
            payment.failure_reason = "processor_unavailable_after_retries"
            payment.save()
            alert_ops.delay(payment_id, reason="exhausted_retries")
            return
        raise
```

Continuous reconciliation jobs typically need no in-loop retry: the next tick covers a missed one.

## 14. Payment without velocity check

**Violates:** I18.

```python
# anti-pattern
def schedule_payment(request):
    # nothing checks the per-tenant or per-counterparty rolling sum
    payment = Payment.objects.create(...)
    return JsonResponse(payment.to_dict())
```

A compromised credential can drain an account before anyone notices. The blast radius is whatever fits in the daily ceiling, and without a velocity check there is no daily ceiling.

**Fix:** check the relevant `velocity_rule` rows in a single transaction; reject with 429 if any rule is breached.

```python
def schedule_payment(request):
    rules = VelocityRule.objects.filter(
        Q(scope="tenant", scope_id=request.user.tenant_id)
        | Q(scope="counterparty", scope_id=bill.vendor.counterparty_id)
    )
    for rule in rules:
        window_start = timezone.now() - timedelta(seconds=rule.window_seconds)
        committed = Payment.objects.filter(
            ...,
            created_at__gte=window_start,
            status__in=("scheduled", "dispatching", "sent"),
        ).aggregate(total=Sum("amount_cents"), count=Count("id"))
        if committed["total"] + amount_cents > rule.max_amount_cents:
            return JsonResponse({"error": f"velocity_limit:{rule.id}"}, status=429)
        if committed["count"] + 1 > rule.max_count:
            return JsonResponse({"error": f"velocity_count:{rule.id}"}, status=429)
    payment = Payment.objects.create(...)
    return JsonResponse(payment.to_dict())
```

A separate per-counterparty rule catches the "scammed bookkeeper" case where a single new vendor receives an outsized share of the day's spend.

## 15. Refund / return without parent FK + state precondition

**Violates:** I19.

```python
# anti-pattern
class Refund(models.Model):
    payment_id = models.CharField()  # not a FK
    amount_cents = models.IntegerField()
```

An orphan refund is a correctness bug, not an edge case. So is a refund larger than its parent payment.

**Fix:** explicit foreign key with `PROTECT`, a check constraint on parent state, and an amount precondition.

```python
class Refund(models.Model):
    payment = models.ForeignKey(
        Payment,
        on_delete=models.PROTECT,
        related_name="refunds",
    )
    amount_cents = models.BigIntegerField()
    idempotency_key = models.CharField(max_length=64)

    class Meta:
        constraints = [
            models.CheckConstraint(
                check=Q(amount_cents__gt=0),
                name="refund_amount_positive",
            ),
            models.UniqueConstraint(
                fields=["payment", "idempotency_key"],
                name="refund_idempotent_per_payment",
            ),
        ]


def issue_refund(payment_id, amount_cents, idempotency_key):
    with transaction.atomic():
        payment = Payment.objects.select_for_update().get(id=payment_id)
        if payment.status not in ("sent", "returned"):
            raise ValueError(f"cannot refund payment in status {payment.status}")
        refunded_so_far = payment.refunds.aggregate(s=Sum("amount_cents"))["s"] or 0
        if refunded_so_far + amount_cents > payment.amount_cents:
            raise ValueError("refund exceeds parent payment amount")
        return Refund.objects.create(
            payment=payment,
            amount_cents=amount_cents,
            idempotency_key=idempotency_key,
        )
```

For webhook handlers that arrive out of order, the parent lookup must refuse to apply if the parent is missing; requeue with backoff (per I8 layer 4) rather than create an orphan.
