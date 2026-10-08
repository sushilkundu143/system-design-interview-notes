# Event-Driven Architecture: Ingestion Jobs, Queues, and Streams

## 1. Mental model

In event-driven architecture, producers publish facts about something that
happened. Consumers react without requiring the producer to call each consumer
synchronously.

```text
Payment service -> PaymentCompleted event -> Broker
                                              |
                   +--------------------------+-------------------+
                   |                          |                   |
              Notification service      Analytics service    Audit service
```

This reduces direct coupling and supports asynchronous processing. It also
introduces eventual consistency, duplicates, ordering challenges, and more
complex debugging.

### Event versus command

- **Event:** A past-tense fact, such as `PaymentCompleted`.
- **Command:** A request to do something, such as `SendPaymentReceipt`.

Events may interest multiple consumers. Commands normally have an intended
handler. Distinguishing them clarifies ownership and semantics.

## 2. Push and pull ingestion

Ingestion brings external data into your system.

### Push ingestion

The source sends data when it is available, using webhooks, event publishers,
or persistent connections.

```text
External provider -> Webhook receiver -> Durable queue -> Workers
```

Typical receiver lifecycle:

1. Verify authentication/signature and validate the envelope.
2. Check an idempotency identifier where applicable.
3. Durably accept the work.
4. Return a timely acknowledgment.
5. Process asynchronously.

Acknowledging before work is durably recorded risks losing the event if the
receiver crashes. Doing all processing inside the webhook request can cause
timeouts and retries.

Benefits:

- Low latency when the provider sends promptly.
- No repeated polling for unchanged data.

Challenges:

- Provider retries can create duplicates.
- Bursts require buffering and backpressure.
- Endpoints need security, replay protection, and availability.
- Events may arrive late or out of order.

### Pull ingestion

Your system requests data from the source periodically or continuously.

```text
Scheduler -> Ingestion job -> External API -> Durable storage/broker
```

Benefits:

- Works with sources that do not support push.
- You control scheduling, concurrency, and request budgets.
- Useful for reconciliation and backfills.

Challenges:

- Freshness is bounded by polling interval plus processing delay.
- Pagination, rate limits, and timeouts need handling.
- Incorrect cursor handling can skip or duplicate records.

### Incremental pull lifecycle

1. Read the last committed cursor or watermark.
2. Request a page of changes.
3. Validate and durably store/enqueue those records.
4. Advance the cursor only after durable acceptance.
5. Repeat until caught up or until the job's bounded work budget is reached.

If a crash occurs before the cursor advances, some records may be fetched again.
Idempotent downstream handling makes that safe.

Timestamp-only cursors can miss records sharing a timestamp or arriving late.
Prefer provider-issued cursors when available, or a stable compound cursor such
as `(updated_at, id)` with overlap and deduplication where appropriate.

### Push/pull ingestion is not the same as broker delivery

An external source can push webhooks into your system while workers pull messages
from a queue. These are decisions at different stages.

## 3. Scheduled ingestion jobs and batch processing

Use jobs for periodic synchronization, backfills, reconciliation, and expensive
work that does not need to finish within a request.

Design jobs with:

- Durable checkpoints.
- Bounded batch sizes and execution time.
- Retryable, idempotent processing.
- Protection against unintended overlapping executions.
- Cancellation and progress reporting.
- Source API rate limits and bounded concurrency.

A scheduled job is not automatically reliable because it runs every minute.
You must know whether runs succeeded and how missed work is recovered.

For backfills, isolate or throttle the workload so historical processing does
not starve live ingestion.

## 4. Queues

A queue typically distributes work among competing consumers.

```text
Producer -> Queue -> Worker A
                  -> Worker B
                  -> Worker C
```

For one work subscription, each message is normally intended to be processed by
one worker, although failures and delivery semantics can cause redelivery.

Examples: background email jobs, image processing, and webhook processing.

### Lifecycle and acknowledgment

1. Producer sends a message.
2. Broker stores it according to its durability configuration.
3. Worker receives it.
4. Worker performs the side effect.
5. Worker acknowledges/deletes it.

If the worker crashes after the side effect but before acknowledgment, the
message may be delivered again. Therefore the side effect must tolerate retries.

In visibility-timeout systems such as Amazon SQS, a received message is hidden
temporarily. If it is not deleted before the timeout, it can become available
again. Long work may require visibility extensions.

Other brokers use explicit acknowledgment and redelivery mechanisms with
different details.

### Fan-out

If both notifications and analytics need every event, do not simply put both
workers on one competing-consumer queue. Use separate subscriptions/queues or
another fan-out mechanism.

## 5. Streams and retained logs

A stream is commonly a retained, ordered log of records organized into
partitions. Consumers track their positions rather than globally deleting
records when one consumer processes them.

```text
Producer -> Topic
             +-- Partition 0: [offset 0][offset 1][offset 2]...
             +-- Partition 1: [offset 0][offset 1][offset 2]...

Consumer group: notifications -> Own committed positions
Consumer group: analytics     -> Own committed positions
```

Kafka is a common example. Other stream systems have different retention,
acknowledgment, and consumer models.

### Partitions and ordering

Kafka preserves record order within a partition, not globally across a topic.
Using an entity ID as the message key can route that entity's records to the same
partition under a stable partitioning arrangement.

Changing partition counts or partitioning rules can affect key placement.
Also, parallel processing within one consumer must not accidentally reorder
side effects when per-entity ordering matters.

### Consumer groups

In Kafka, partitions are assigned among consumers within a group. Different
groups independently read the same topic.

One partition is assigned to one group member at a time in the usual group
model. Adding consumers beyond the number of partitions does not increase
active partition assignments.

### Replay and retention

Consumers can re-read retained records to rebuild projections, recover bugs, or
perform backfills. Replay is limited by retention and compaction policies.

Replay can repeat external side effects. A notification consumer should not
blindly resend every email when rebuilding historical state.

### Queue versus stream

| Aspect | Typical work queue | Typical retained stream |
| --- | --- | --- |
| Main purpose | Distribute tasks | Retain and distribute event history |
| Completion | Acknowledge/delete work for a subscription | Advance consumer position |
| Multiple independent readers | Separate subscriptions/queues | Independent consumer groups |
| Replay | Depends on broker/features | Common while records remain retained |
| Ordering | Product/configuration dependent | Usually partition-scoped |

The distinction is conceptual: some products offer both models.

## 6. Delivery guarantees

### At-most-once

A message may be lost but is not retried for delivery in the relevant scope.
This may be acceptable for noncritical telemetry, not for a financial event that
must be processed.

### At-least-once

The system retries under its supported failure model, so duplicates can occur.
Durability, retention, retry policy, and eventual consumer recovery still matter:
the label is not an unconditional guarantee against every possible loss.

### Exactly-once

Always ask: exactly once **where**?

Kafka transactions can provide exactly-once processing for supported
consume-transform-produce flows inside Kafka. They do not automatically make an
external email, database write, or payment API call exactly once.

For external side effects, use idempotency and transactional coordination
appropriate to that destination.

## 7. Idempotent consumers

Processing the same event more than once should not create duplicate business
effects.

An event envelope might contain:

```json
{
  "eventId": "evt_123",
  "type": "PaymentCompleted",
  "schemaVersion": 1,
  "aggregateId": "payment_456",
  "aggregateVersion": 3,
  "occurredAt": "2026-10-08T09:00:00Z",
  "correlationId": "request_789",
  "data": {
    "amountMinor": 120000,
    "currency": "INR"
  }
}
```

Use integer minor units or an appropriate decimal representation for money,
not binary floating-point arithmetic.

### Database-side deduplication

For a consumer whose side effect is a database update:

1. Begin a transaction.
2. Insert `(consumer_name, event_id)` into a deduplication table with a unique
   constraint.
3. If that pair was already committed, skip the duplicate safely.
4. Apply the business change.
5. Commit both the marker and change together.
6. Acknowledge the broker message.

Marking an event processed before the business update commits can lose work.
A non-atomic "check, then insert" can race.

For an external API, use its idempotency key if supported and persist workflow
state. A local database transaction cannot atomically include an arbitrary
remote network call.

## 8. The dual-write problem and transactional outbox

Suppose a service must update its database and publish an event:

```text
Commit payment update
         |
Crash before broker publish
         |
Payment updated, but no event
```

Publishing first creates the opposite risk: an event can escape even if the
database transaction fails.

### Outbox pattern

Within one database transaction:

```text
BEGIN
  Update payment
  Insert PaymentCompleted into outbox
COMMIT
```

A relay later publishes committed outbox records:

```text
Database/outbox -> Relay or CDC connector -> Broker -> Consumers
```

If the relay crashes after publishing but before recording completion, it may
publish again. Consumers still need idempotency.

The outbox closes the database/broker dual-write gap; it does not remove
duplicates, guarantee global ordering, or solve every downstream failure.

**CDC**, or change data capture, reads database changes from a log or another
supported mechanism. It can relay outbox records or support ingestion, but
requires operational management of offsets, retention, schemas, and lag.

## 9. Retries, dead-letter handling, and backpressure

### Retry policy

- Retry transient failures with exponential backoff and jitter.
- Bound retry attempts or elapsed time according to the workflow.
- Do not repeatedly retry permanent validation failures.
- Preserve event identity across retries.
- Avoid retrying ambiguous non-idempotent side effects without reconciliation.

### Dead-letter queue

A DLQ isolates messages that cannot be processed under the current policy.
Store failure details, alert owners, investigate, and replay only after a fix.
A DLQ is not a place to silently abandon required work.

### Backpressure

If production exceeds consumption for a sustained period, backlog grows.
Queues absorb bursts; they do not provide infinite capacity.

Respond with bounded concurrency, admission controls, producer throttling,
capacity changes, or explicit load shedding where permissible.

If arrival rate is 1,000 messages/second and effective worker throughput is 50
messages/second, 20 workers only match arrivals theoretically. Add headroom and
account for downstream limits, partition constraints, retries, and backlog
recovery. Autoscaling workers can overload the database.

## 10. Ordering, event time, and schema evolution

**Event time** is when the source event occurred. **Processing time** is when a
consumer handles it. Late events mean those times differ.

For time-window analytics, watermarks express how far event time is believed to
have progressed. Define what happens to events arriving after the allowed
lateness window.

For entity projections, sequence/version numbers can detect stale updates,
duplicates, or missing events. Define recovery rather than blindly overwriting
newer state with an older event.

Schema practices:

- Include an event type and version.
- Prefer compatible additive changes when possible.
- Do not silently change field meaning.
- Validate producer and consumer contracts.
- Plan for old events during replay.
- Avoid unnecessary personal data in long-retained events.

## 11. Frontend implications

React normally consumes a secured API, not the internal broker directly.

To reflect asynchronous processing, use polling, Server-Sent Events, or
WebSockets through an authorized backend.

```text
React -> Submit operation -> Backend returns accepted operation ID
                               |
                          Queue/stream workers
                               |
                          Status projection
                               |
React <- Polling/SSE/WebSocket <- Backend
```

An HTTP `202 Accepted` means work was accepted, not that the business operation
succeeded. Show pending, completed, and failed states explicitly.

On reconnect, the frontend may need to fetch authoritative current state rather
than assume every live event was received.

## 12. Observability and security

Monitor:

- Publish/accept failures.
- End-to-end event age.
- Queue depth and oldest-message age.
- Stream consumer lag and processing latency.
- Retries, DLQ volume, and duplicate handling.
- Outbox backlog and relay failures.
- Broker capacity and downstream resource saturation.

Propagate correlation IDs through asynchronous steps. Secure producer/consumer
access, verify webhook signatures, encrypt transport, and apply retention and
access controls to sensitive payloads.

## 13. Practice questions

**When would you choose a queue over a stream?**

For distributing tasks where independent replay is not a central requirement.
Choose a retained stream when multiple consumers and historical replay matter.

**What if a consumer crashes after updating the database?**

The message can be redelivered. An atomic deduplication marker and business
update prevent a repeated effect.

**Why combine push and pull?**

Push provides timely notifications; pull reconciliation can recover missed
events and verify source-of-truth state.

**Does event-driven architecture mean event sourcing?**

No. Event sourcing uses events as the authoritative history from which state is
derived. A system can publish events while retaining ordinary mutable database
state as its source of truth.

## 14. Interview summary

> I choose push or pull ingestion based on source capabilities and freshness,
> queues for work distribution, and streams for retained history and independent
> consumers. I design durable acceptance, idempotency, outbox publishing,
> ordering, retries, backpressure, and observability before claiming reliability.

## References

- [Apache Kafka documentation](https://kafka.apache.org/documentation/)
- [Amazon SQS visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)
- [RabbitMQ consumer acknowledgments](https://www.rabbitmq.com/docs/confirms)
- [Debezium outbox event router](https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html)
