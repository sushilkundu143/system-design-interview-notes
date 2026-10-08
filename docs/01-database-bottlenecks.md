# Database Bottlenecks: Partitioning, Sharding, and Rate Limiting

## 1. Mental model

A database bottleneck occurs when database work becomes the limiting factor in
the application's throughput or response time. More application servers can
make it worse by sending even more concurrent work to the same database.

Three frequently discussed techniques solve different problems:

| Technique | Primary purpose | Does it add database capacity? |
| --- | --- | --- |
| Partitioning | Organize a large table and reduce irrelevant data access | Not automatically |
| Sharding | Spread data and workload across independent database units | Yes, when load distributes well |
| Rate limiting | Restrict incoming work before it overwhelms the system | No; it protects existing capacity |

They are complementary, not interchangeable.

## 2. Diagnose before redesigning

Start with an actual slow request and trace where time is spent: waiting for a
connection, executing SQL, waiting for a lock, transferring results, or processing
them in the application.

| Observation | Likely investigation | Possible response |
| --- | --- | --- |
| Slow query with large row scans | Execution plan, selectivity, indexes | Add an appropriate index or rewrite the query |
| Hundreds of queries for one screen | N+1 patterns and ORM behavior | Batch reads or preload related records |
| High connection wait time | Pool usage and transaction duration | Bound connections and shorten transactions |
| High lock wait time | Blocking transactions and hot rows | Reduce lock duration and redesign contested updates |
| High disk I/O | Working-set size and access patterns | Reduce scans, improve indexes, evaluate memory/storage |
| High CPU | Sorting, joins, aggregates, excessive query rate | Optimize queries and offload suitable work |
| Growing replica lag | Write volume and replica capacity | Tune replication; do not assume replicas are current |

Measure p50, p95, and p99 latency, query throughput, errors, connection utilization,
lock waits, CPU, storage latency, and replica lag. Averages can hide severe tail
latency.

### Example: account transaction history

```sql
SELECT id, amount, created_at
FROM transactions
WHERE account_id = 42
  AND created_at < '2026-10-01T00:00:00Z'
ORDER BY created_at DESC, id DESC
LIMIT 50;
```

An index starting with `account_id`, followed by fields supporting the date filter
and ordering, may help this access pattern. Verify the actual execution plan:
index design depends on database engine, query shape, and data distribution.

Prefer keyset pagination for deep histories where appropriate. Large `OFFSET`
values can require the database to process many rows that are discarded.

In PostgreSQL, `EXPLAIN (ANALYZE, BUFFERS)` provides actual execution information,
but it executes the statement. Do not casually run it against costly queries or
mutating statements in production.

## 3. Partitioning

Partitioning divides one logical table into smaller physical partitions.
Applications usually continue querying the parent table.

```text
transactions
  +-- transactions_2026_08
  +-- transactions_2026_09
  +-- transactions_2026_10
```

### Range partitioning

Useful when access and retention follow a range such as time.

Illustrative PostgreSQL DDL:

```sql
CREATE TABLE transactions (
    id BIGINT NOT NULL,
    account_id BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    amount NUMERIC(18, 2) NOT NULL,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

CREATE TABLE transactions_2026_10
PARTITION OF transactions
FOR VALUES FROM ('2026-10-01 00:00:00+00')
             TO ('2026-11-01 00:00:00+00');
```

The composite primary key includes the partition key to satisfy PostgreSQL's
partitioned-table uniqueness requirements in this example. It does not enforce
global uniqueness of `id` alone.

### Partition pruning

A query for October can skip partitions whose bounds cannot contain matching
rows:

```sql
SELECT account_id, amount
FROM transactions
WHERE created_at >= '2026-10-01 00:00:00+00'
  AND created_at <  '2026-11-01 00:00:00+00';
```

This is partition pruning. It reduces the search space; an appropriate index
inside the relevant partitions can still be necessary.

### Other strategies

- **List:** Group explicit values, such as business regions.
- **Hash:** Distribute rows into buckets based on a hash of a key.
- **Subpartitioning:** Combine strategies when the engine supports it and the
  complexity is justified.

### Benefits and limitations

Benefits:

- Less irrelevant data scanned for suitable queries.
- Easier retention: detach or drop an old partition instead of deleting rows
  individually, subject to locking and operational constraints.
- More manageable maintenance of very large tables.

Limitations:

- Queries without useful partition predicates may touch every partition.
- Too many partitions can increase planning and maintenance overhead.
- Missing future partitions can cause inserts to fail without a fallback.
- Constraints, foreign keys, and indexing have engine-specific behavior.
- Partitioning on one machine does not remove that machine's capacity limit.

**Interview point:** Choose a partition key from actual query and lifecycle
patterns, not simply because a column exists.

## 4. Sharding

Sharding horizontally distributes subsets of data across separate database
instances or independently managed database units.

```text
Request with customer_id
          |
     Shard router
     /     |     \
 Shard A Shard B Shard C
```

### Choosing the shard key

A good shard key:

- Distributes data and traffic reasonably evenly.
- Is available in common requests.
- Keeps frequently joined or transactionally related data together.
- Does not frequently change.

Sharding banking data by customer can keep that customer's accounts and
transactions together. However, customers with exceptionally high traffic can
create hot shards, and transfers between customers may cross shard boundaries.

### Routing strategies

| Strategy | Advantage | Challenge |
| --- | --- | --- |
| Range-based | Related ranges stay together | Uneven or sequential traffic can create hot shards |
| Hash-based | Often spreads keys more evenly | Range queries become scatter-gather operations |
| Directory-based | Explicit control over placement | The directory becomes critical infrastructure |

`hash(customer_id) % shard_count` is a useful teaching example, but changing
`shard_count` remaps many keys. Production designs may use logical buckets,
consistent hashing, or a routing directory to control migration.

### Cross-shard operations

Customer-specific reads can target one shard. Global reporting may require
scatter-gather queries or a separate analytics system.

Cross-shard writes are more difficult. A financial transfer requires a precise
correctness model: distributed transactions, a durable ledger workflow, or
another carefully designed protocol. "Use eventual consistency" is not enough
to explain how money remains correct.

### Rebalancing

Adding a shard requires more than starting another database:

1. Identify data ranges or buckets to move.
2. Copy data while tracking changes.
3. Validate correctness.
4. Coordinate routing cutover and in-flight writes.
5. Retire the old copy only when safe.

Plan recovery if migration fails halfway through.

### Sharding versus replication

- **Sharding:** Different databases contain different subsets of data.
- **Replication:** Another database contains a copy of data from its primary.

Read replicas can offload suitable reads, but replication lag may violate
read-after-write expectations. They do not generally scale primary write
capacity.

## 5. Rate limiting and overload protection

Rate limiting controls request arrival rates before expensive work occurs.

```text
Client -> Gateway/rate limiter -> Application -> Database
                    |
              Limit exceeded
                    |
                 HTTP 429
```

Example policy: 100 requests per authenticated user per minute. This is an
illustrative value; choose limits from business needs and load tests.

### Algorithms

| Algorithm | Mechanism | Trade-off |
| --- | --- | --- |
| Fixed window | Count within fixed intervals | Boundary bursts can nearly double the intended short-term rate |
| Sliding window log | Track request timestamps | Accurate rolling windows, but higher storage/work |
| Sliding window counter | Approximate a rolling count | Efficient, with approximation trade-offs |
| Token bucket | Refill tokens; each request consumes tokens | Supports bounded bursts |
| Leaky bucket | Drain work at a controlled rate | Smooths traffic; any queue must be bounded |

For a token bucket with capacity 20 and refill rate 5/second, a client can make a
burst of up to 20 requests after the bucket fills, then sustain roughly 5/second.

Multiple application instances often use Redis or a gateway's shared mechanism.
The check and update must be atomic; a naive separate read and write permits
concurrent requests to bypass the limit.

### Rate versus concurrency

A rate limit controls requests per unit time. A concurrency limit controls
simultaneously active work.

By Little's Law, in a stable system:

```text
Average in-flight work = Throughput x Average processing time
```

At 100 requests/second with a 2-second processing time, about 200 requests are
in flight on average. When processing slows, concurrency rises even if the
arrival rate is unchanged.

Use bounded concurrency, bounded queues, timeouts, and backpressure alongside
rate limits. Per-user limits alone are insufficient if thousands of users
collectively overload the database.

### Failure and client behavior

- Return `429 Too Many Requests` for client rate-limit violations.
- Supply `Retry-After` when appropriate.
- Clients should back off and add jitter, not immediately retry.
- Decide explicitly whether limiter failure is fail-open or fail-closed.
- Infrastructure overload may warrant `503 Service Unavailable`, rather than
  pretending the caller exceeded a personal quota.

## 6. Practical decision sequence

1. Establish latency, throughput, consistency, and availability requirements.
2. Measure and fix inefficient queries, indexes, N+1 requests, and transactions.
3. Bound connections and concurrency.
4. Cache suitable reads and evaluate read replicas.
5. Partition when table size and access/retention patterns justify it.
6. Shard when a single database cannot meet capacity needs after simpler fixes.
7. Apply overload protection throughout; it is not only a final-stage fix.

## 7. Practice questions

**Does partitioning always improve query speed?**

No. It helps when pruning or maintenance benefits match the workload. A query
that touches all partitions may see little improvement or additional overhead.

**Why not shard every application from the beginning?**

Routing, migrations, joins, transactions, backups, and operations become more
complex. That cost is rarely justified before a real capacity need exists.

**How do you handle a hot customer?**

Measure its traffic, isolate placement if needed, cache suitable reads, and
consider a finer distribution model without breaking transaction requirements.

**Can rate limiting solve a slow SQL query?**

It can protect the system from repeated expensive work, but the query still
needs optimization.

## 8. Interview summary

> I diagnose bottlenecks using query plans, slow-query logs, connection waits,
> locks, and resource metrics. I optimize queries first, partition large tables
> according to access and retention patterns, shard when capacity requires
> distribution, and use rate and concurrency limits to prevent overload.

## References

- [PostgreSQL table partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)
- [PostgreSQL EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [Redis INCR and rate-limiter patterns](https://redis.io/docs/latest/commands/incr/)
- [HTTP 429](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429)
