# Distributed Messaging Core — Broker, Queues, Fan-out & Scalability

Owner: master-system-design
Section: 20
Status: Draft

<!-- SOURCE: Training 1 - "I build a message broker on top of a relational database first BECAUSE relational databases give a lot of guarantees/features that make it easy to understand what properties I need from the underlying storage system without just drawing random boxes" -->
<!-- SOURCE: Training 5 - "log structured storage shines ... Kafka itself is log structured storage because it can support very high write throughput ... the magic sauce behind any system that claims to be a high write throughput system" -->

This document designs a distributed messaging core (broker, queues, fan-out, scalability) in **phases**. Phase 1 builds the broker on a relational database so we can *name the exact guarantees* we need (ordering, exactly-once delivery, presence). Phase 2 replaces the relational storage with a log-structured / LSM custom storage layer that must satisfy the *same* guarantees. Phase 3 covers the stateless API tier + load balancer + auto-scaling policy. Phase 4 covers the real-time transport choice (WebSocket vs Long Polling vs SSE) for <100ms delivery and presence. Phase 5 covers distributed locking for ordering consistency under concurrent writes to the same chat/group. Phase 6 covers message-queue usage for offline push, multi-device sync, and async persistence. Phase 7 covers group fan-out for 1024 members. Phase 8 covers sharding, read replicas, tiered storage, and 99.99% uptime.

---

## Phase 0 — The One Thing To Remember

<!-- SOURCE: Training 1 - "there is no secret sauce to approaching distributed systems, you just need to remember one thing" -->
<!-- SOURCE: Training 6 - "the standard way to almost a standard way to think about system is read path and write path optimization. Think of what the read paths are. Think of what the write paths are. And then tackle one at a time" -->

There is no secret sauce. The one thing: **separate the read path from the write path, then scale each independently.** Every phase below is either a write-path decision or a read-path decision. We never scale "the system" — we scale a path.

---

## Phase 1 — Broker on a Relational Database (name the guarantees)

<!-- SOURCE: Training 1 - build the broker on a relational DB first to understand required storage properties -->

We do **not** start by drawing storage boxes. We start with a relational DB (Postgres/MySQL) because it gives us transactions, unique constraints, and a monotonic sequence — which lets us *state* the guarantees we need before we try to re-implement them.

### 1.1 Schema (Phase 1)

```sql
-- A conversation is a 1:1 chat or a group.
CREATE TABLE conversations (
    conversation_id   BIGINT PRIMARY KEY,
    type              TEXT NOT NULL,          -- 'direct' | 'group'
    created_at        TIMESTAMP NOT NULL
);

-- Membership. For a 1024-member group this table has 1024 rows per group.
CREATE TABLE members (
    conversation_id   BIGINT NOT NULL,
    user_id           BIGINT NOT NULL,
    joined_at         TIMESTAMP NOT NULL,
    PRIMARY KEY (conversation_id, user_id)
);

-- The message log. seq is the per-conversation monotonic ordering key.
CREATE TABLE messages (
    conversation_id   BIGINT NOT NULL,
    seq               BIGINT NOT NULL,        -- per-conversation, monotonic
    message_id        UUID NOT NULL,
    sender_id         BIGINT NOT NULL,
    body              TEXT NOT NULL,
    client_msg_id     UUID NOT NULL,          -- idempotency key from sender
    created_at        TIMESTAMP NOT NULL,
    PRIMARY KEY (conversation_id, seq),
    UNIQUE (conversation_id, client_msg_id)   -- exactly-once on retry
);

-- Per-recipient delivery state (drives presence + multi-device sync).
CREATE TABLE deliveries (
    conversation_id   BIGINT NOT NULL,
    seq               BIGINT NOT NULL,
    recipient_id      BIGINT NOT NULL,
    device_id         TEXT NOT NULL,
    state             TEXT NOT NULL,          -- 'pending'|'delivered'|'read'
    updated_at        TIMESTAMP NOT NULL,
    PRIMARY KEY (conversation_id, seq, recipient_id, device_id)
);

-- Presence: last seen + online. One row per user (not per device).
CREATE TABLE presence (
    user_id           BIGINT PRIMARY KEY,
    online            BOOLEAN NOT NULL,
    last_seen_at      TIMESTAMP NOT NULL,
    connection_id     TEXT                    -- null when offline
);
```

### 1.2 The three guarantees, stated precisely

**G1 — Ordering.** Within a conversation, messages are totally ordered by `seq`. `seq` is assigned by the broker, never by the client. Two messages from the same sender must never be reordered; two messages from different senders must be ordered consistently for *all* readers of that conversation.

**G2 — Exactly-once delivery (effectively-once).** A sender retrying a send must not create a duplicate message. Enforced by `UNIQUE (conversation_id, client_msg_id)`: the retry hits the constraint and returns the *existing* `seq`. On the recipient side, `deliveries` is keyed by `(conversation_id, seq, recipient_id, device_id)`, so a redelivered message is an idempotent upsert, not a second row.

**G3 — Presence.** `online` is true only while at least one live connection exists for the user. `last_seen_at` is updated on disconnect. Presence is *eventually consistent* — this is acceptable and is the correct choice here.

<!-- SOURCE: Training 8 - "you would never find any production use case where you would not really strong consistency ... if it is eventually consistent good enough for 99% cases only some use cases where you need very strong" -->

### 1.3 Write path (Phase 1)

```
client --send--> API server
  1. BEGIN
  2. INSERT INTO messages (conversation_id, seq, ...) VALUES (cid, next_seq(cid), ...)
       -- next_seq(cid) is a per-conversation counter, taken under a lock (Phase 5)
  3. INSERT INTO deliveries (...) for each recipient device  -- fan-out (Phase 7)
  4. COMMIT
  5. publish to real-time transport (Phase 4)
```

The relational DB gives us G1/G2 for free via the primary key + unique constraint. **This is the point of Phase 1**: we now know exactly what the custom storage layer must reproduce.

### 1.4 Read path (Phase 1)

```
client --history(cid, since_seq)--> API server
  SELECT * FROM messages WHERE conversation_id = cid AND seq > since_seq
  ORDER BY seq ASC LIMIT 200;
```

Reads are served from replicas (Phase 8). The read path is a range scan on `(conversation_id, seq)` — this is the access pattern the LSM layer must optimize for.

---

## Phase 2 — Replace the Relational DB with Log-Structured / LSM Storage

<!-- SOURCE: Training 5 - "log structured storage shines ... metrics, analytics, clickstream ... Kafka itself is log structured storage ... write on cheap commodity hardware at very high performance" -->

Once Phase 1 has named G1/G2/G3, we replace the relational DB with a custom log-structured store. **The custom layer is correct if and only if it satisfies G1, G2, G3.**

### 2.1 Storage layout

- **Commit log (append-only).** One logical log per conversation shard. Every message is appended as `(conversation_id, seq, message_id, sender_id, body, client_msg_id, created_at)`. Appends are sequential writes — this is where the high write throughput comes from.
- **`seq` assignment.** The per-conversation counter is *not* stored in the log; it is derived from the log offset. Because the log is append-only and single-writer per shard, `seq = log_offset` is monotonic by construction. This is how G1 is preserved without a separate counter.
- **Memtable + SSTables (LSM).** Recent appends live in an in-memory memtable keyed by `(conversation_id, seq)`. On flush, the memtable becomes an immutable SSTable. Reads merge the memtable with SSTables (newest wins).
- **Idempotency index (G2).** A secondary index `client_msg_id -> seq` is maintained in the LSM. On append, we look up `client_msg_id`; if present, we return the existing `seq` and do **not** append. This reproduces the `UNIQUE` constraint from Phase 1.
- **Delivery state (G3 + sync).** `deliveries` is a separate LSM keyed by `(recipient_id, conversation_id, seq)` so that "give me everything pending for this device" is a single range scan — the multi-device sync read path.

### 2.2 Compaction

SSTables are compacted in the background. Compaction must preserve `seq` ordering and must not resurrect a deleted/acked delivery row. Tombstones are used for delivery acks.

### 2.3 Guarantee checklist (the acceptance test for Phase 2)

| Guarantee | Phase 1 mechanism | Phase 2 mechanism |
|-----------|-------------------|-------------------|
| G1 ordering | `PRIMARY KEY (conversation_id, seq)` | append-only log offset = `seq` |
| G2 exactly-once | `UNIQUE (conversation_id, client_msg_id)` | `client_msg_id -> seq` LSM index |
| G3 presence | `presence` table | presence store (Phase 4), eventually consistent |

If all three rows hold, the broker works on the custom layer.

---

## Phase 3 — Stateless API Servers + Load Balancer + Auto-Scaling

<!-- SOURCE: Training 1 - "the load balancer is the core component that makes distributed systems possible ... it abstracts the details of having multiple servers in the backend" -->
<!-- SOURCE: Training 1 - "writing a load balancer is simple with hardly 10 lines of core code" -->
<!-- SOURCE: Training 3 - "When scaling API servers, put a load balancer, add more servers, and define an auto scaling policy BECAUSE API servers are stateless and can very easily scale" -->

### 3.1 Why stateless

API servers hold **no** conversation state, no connection state that matters for correctness, and no session. All state is in the broker storage (Phase 2) and the presence store (Phase 4). This is what makes them trivially scalable.

### 3.2 The load balancer (core logic)

The core of a load balancer is ~10 lines. It is a reverse proxy that picks a backend and forwards.

```python
# SOURCE: Training 1 - "hardly 10 lines of core code. That's it."
# SOURCE: Training 1 - "The one who built Nginx. People use Nginx as a reverse proxy."
import itertools, http.client

class LoadBalancer:
    def __init__(self, backends):
        self.backends = backends
        self._rr = itertools.cycle(backends)   # round-robin

    def pick(self):
        return next(self._rr)

    def forward(self, method, path, body, headers):
        host, port = self.pick()
        conn = http.client.HTTPConnection(host, port)
        conn.request(method, path, body=body, headers=headers)
        return conn.getresponse()
```

That is the whole idea. Production adds health checks, connection draining, and TLS termination — but the core is this small.

### 3.3 Auto-scaling policy

- **Signal:** p95 request latency and CPU per API server, plus active WebSocket connection count (Phase 4).
- **Scale-out:** if p95 latency > 100ms for 60s, add servers.
- **Scale-in:** if CPU < 30% for 10 minutes, remove servers.
- **Why this works:** API servers are stateless, so a new server needs no warm-up beyond process start. <!-- SOURCE: Training 3 -->

---

## Phase 4 — Real-Time Transport: WebSocket vs Long Polling vs SSE

Requirement: **<100ms delivery** and **presence (Last Seen / Online)**.

### 4.1 Comparison

| Transport | Delivery latency | Server→client | Presence fit | Cost |
|-----------|------------------|---------------|--------------|------|
| **WebSocket** | <100ms (persistent, full-duplex) | push | Excellent — connection open/close *is* the presence signal | Holds a connection per device |
| **Long Polling** | ~100ms–1s (reconnect gap) | push-ish | Poor — "online" is ambiguous between polls | Cheap, but chatty |
| **SSE** | <100ms (persistent, one-way) | push only | Good for online, but client→server needs a second channel | Holds a connection per device |

### 4.2 Decision

**Use WebSocket for the real-time channel.** It is the only option that gives <100ms *and* a natural presence signal: the connection lifecycle maps directly to `online`/`last_seen_at`.

<!-- SOURCE: Training 1 - "remote and distributed locking forms the heart and crux of data correctness" (transport must feed the presence store that locking/sync depends on) -->

### 4.3 Presence implementation

```
on WS connect:    presence.upsert(user_id, online=true, connection_id=conn)
on WS disconnect: presence.upsert(user_id, online=false, last_seen_at=now())
```

- Presence is **eventually consistent** — acceptable per Training 8.
- A heartbeat every 30s refreshes `last_seen_at`; a missed heartbeat marks the user offline.
- Presence is stored in a fast key-value store (Redis-class), not the message log, because it is high-write and low-value-per-byte.

### 4.4 Delivery path

```
broker commit (Phase 1/2)
   -> publish to real-time transport
   -> if recipient online: push over WebSocket  (<100ms)
   -> if recipient offline: enqueue to push queue (Phase 6)
```

---

## Phase 5 — Distributed Locking for Ordering Consistency

<!-- SOURCE: Training 1 - "imagine you have multiple processes and they have to coordinate among themselves so that the data consistency thing stays in check ... two transactions happening in a distributed fashion, both of them trying to update the same row. One transaction needs to take a lock so that the other does not update" -->
<!-- SOURCE: Training 1 - "why you need it. That's a more important question to answer. And then how you implement it using Redis" -->
<!-- SOURCE: Training 1 - "what's the core property that we seek from a system that could be our lock manager? So that you don't want to use Redis, build your own" -->

### 5.1 Why (answer this before how)

Two API servers receive two messages for the **same conversation** at the same instant. Both try to assign `seq`. Without coordination, both could read `next_seq = 42` and both write `seq = 42` — a duplicate ordering key, breaking G1. The lock exists to serialize `seq` assignment **per conversation**.

### 5.2 The core property we need from a lock manager

We do not need "Redis". We need a system that provides:

1. **Mutual exclusion** — at most one holder at a time.
2. **Atomic acquire** — `SET key value NX PX ttl` semantics (set-if-not-exists with expiry).
3. **TTL / auto-release** — so a crashed holder does not deadlock the conversation forever.
4. **Fencing** — a monotonically increasing token so a stale holder's write is rejected.

If a candidate system provides these four, it can be our lock manager. Redis is one such system; a custom one is possible.

### 5.3 Lock granularity

**Lock per conversation, not per message and not global.** A global lock would serialize the entire system; a per-message lock would not prevent two messages from racing for the same `seq`. Per-conversation is the correct granularity because ordering is only defined *within* a conversation.

### 5.4 Acquire / release

```
acquire(cid):
  token = monotonic_counter(cid)
  ok = SET lock:cid token NX PX 5000
  if not ok: retry with backoff
  return token

release(cid, token):
  if GET lock:cid == token: DEL lock:cid   -- only release your own lock
```

### 5.5 Fencing token

Every write to the log carries the fencing token. The log rejects a write whose token is lower than the highest token it has seen for that conversation. This prevents a paused-then-resumed holder from writing an out-of-order `seq`.

### 5.6 Interaction with Phase 2

On the custom LSM layer, the lock is only needed at the **shard leader** (Phase 8). Because the log is single-writer per shard, the leader can assign `seq` from the log offset without a distributed lock — the lock is only needed during leader failover to prevent two leaders from appending.

---

## Phase 6 — Message Queues: Offline Push, Multi-Device Sync, Async Persistence

<!-- SOURCE: Training 2 - "consumers can write to each write to three SQS ... they might have their own independent worker ... this is typically how notification systems ... very high fan out ... you are limited by parallelism of Kafka, which is what you do, you extend it with brokers" -->
<!-- SOURCE: Training 4 - "you club the parallelization limitation of Kafka with high fan out of SQS" -->

### 6.1 Three distinct queue uses

**Use 1 — Offline push notifications.**
When a recipient is offline (Phase 4), the message is enqueued to a push queue. A worker pool consumes and calls APNs / FCM / Twilio.

**Use 2 — Multi-device sync.**
Each device has its own cursor (`last_seq`). When a device reconnects, it reads `deliveries` (Phase 2) for `(recipient_id, conversation_id, seq > last_seq)`. The queue is used to *notify* the device that new messages exist, not to carry the messages themselves.

**Use 3 — Async persistence.**
The API server commits to the log (Phase 2) and returns. Secondary work — search indexing, analytics, media metadata — is enqueued and processed asynchronously so it never blocks the send path.

### 6.2 The Kafka + SQS fan-out pattern

<!-- SOURCE: Training 2 - "consumers can write to each write to three SQS ... you club the parallelization limitation of Kafka with high fan out of SQS" -->

Kafka parallelism is bounded by partition count. For a bulk notification (e.g., a 1024-member group announcement), that bound is too low. The pattern:

```
Kafka (N partitions, N consumers)
   -> each consumer fans out round-robin into M SQS queues
   -> each SQS queue has its own independent worker pool
   -> workers call APNs / FCM / Twilio
```

This is the standard way to break the Kafka parallelism ceiling for high-fan-out notification workloads. <!-- SOURCE: Training 2 -->

### 6.3 Receipt handles

<!-- SOURCE: Training 9 - "if you don't have a receipt handle ... how would you delete a particular message? ... how will you verify that the consumer who has read the message is the one who is deleting the message?" -->

Every dequeued message carries a **receipt handle**. Deletion requires the receipt handle, so a consumer that did not read the message cannot delete it. This prevents one worker from deleting another worker's in-flight message.

---

## Phase 7 — Group Fan-Out for 1024 Members

### 7.1 Write fan-out vs read fan-out

| | Write fan-out | Read fan-out |
|---|---|---|
| On send | Write 1 row per member (1024 rows) | Write 1 row for the group |
| On read | Read 1 row | Read group + expand to members |
| Best when | Groups are small / members are online | Groups are large / members are mostly offline |
| Cost | O(members) per send | O(members) per read |

### 7.2 Decision for 1024 members

**Hybrid, split by online/offline:**

- **Online members (say ~50):** write fan-out. Push directly over WebSocket (Phase 4). This is cheap because the set is small and the delivery is immediate.
- **Offline members (~974):** read fan-out. Do **not** write 974 delivery rows synchronously. Instead, write one row to the group log and let each device pull on reconnect using its cursor (Phase 6, Use 2).

This keeps the send path O(online members) instead of O(1024), while still guaranteeing every member eventually sees the message.

### 7.3 Why not pure write fan-out

Pure write fan-out for 1024 members means 1024 delivery rows per message. At even modest group activity this dominates the write path and is the scaling bottleneck. <!-- SOURCE: Training 3 - "Avoid relying on a single transactional database ... making the database the scaling bottleneck" -->

### 7.4 Why not pure read fan-out

Pure read fan-out means every online member's read hits the group log and expands membership — expensive at read time and adds latency, violating the <100ms goal for online delivery.

---

## Phase 8 — Scalability & Availability

### 8.1 Sharding masters into mutually exclusive subsets

<!-- SOURCE: Training 3 - "When a single vertically scaled node cannot handle increasing write requests ... shard the master into mutually exclusive subsets" -->
<!-- SOURCE: Training 3 - "create mutually exclusive groups like users 1-100 on one shard, 100-200 on another ... sharding requires partitioning data into end databases with mutually exclusive subsets" -->

When one master can no longer absorb writes, shard by `conversation_id`:

```
shard = hash(conversation_id) % num_shards
```

Each shard owns a **mutually exclusive** subset of conversations. A conversation lives on exactly one shard, so ordering (G1) is preserved without cross-shard coordination.

**Consequence:** the API server must know which shard to forward to. <!-- SOURCE: Training 3 - "requires your API server to know where to forward the request to" --> This is a routing table lookup, not a broadcast.

### 8.2 Read replicas for history

<!-- SOURCE: Training 3 - "When you have large amount of reads on your order database from internal services ... add read replicas on the master and transfer reads to replicas BECAUSE those services only read and do not write" -->

History reads (Phase 1.4) are read-only and high-volume. Add read replicas per shard and route all history reads to replicas. The master handles only writes (send path). This separates the read path from the write path — the one thing from Phase 0.

### 8.3 Tiered storage for media/files and 5-year history

<!-- SOURCE: Training 7 - "In S3 there is multiple tiers ... standard tier, infrequent access, and glacier ... glacier is the cheapest one. It gives you very slow reads ... infrequent access is in middle" -->
<!-- SOURCE: Training 3 - "I teach tiered storages deeply BECAUSE it is one of the most underrated ways to get performance out of a system while keeping the cost low" -->

| Tier | Contents | Access latency | Cost |
|------|----------|----------------|------|
| Hot (LSM, Phase 2) | Last 30 days of messages | ms | high |
| Warm (infrequent access) | 30 days – 1 year | seconds | medium |
| Cold (glacier-class) | 1 – 5 years | minutes–hours | lowest |

Media/files (images, video, documents) go to blob storage with the same tiering. Message bodies stay in the log; media is referenced by URL.

### 8.4 99.99% uptime via replication and failure handling

- **Replication:** each shard has a leader + followers. Writes go to the leader; followers replicate the log. On leader failure, a follower is promoted.
- **Failure handling:** the fencing token (Phase 5.5) prevents a demoted leader from continuing to append. This is the critical safety property during failover.
- **Over-provisioning:** keep headroom so a single node failure does not push remaining nodes past capacity. <!-- SOURCE: Training 10 - "You little over provisioned infrastructure to ensure a very high availability problem solved" -->
- **Availability math:** 99.99% = ~52 minutes downtime/year. With leader+follower per shard and fast promotion, a single-node failure is absorbed without breaching this.

### 8.5 Hot partition handling

<!-- SOURCE: Training 10 - "Hot partition problems solved ... static allotment or static mapping is very easy data migration" -->

A single very active group can make one shard hot. Mitigations:
- Split the hot conversation's log into sub-partitions by `seq` range.
- Move the hot conversation to a dedicated shard (static mapping makes this a data migration, not a rehash of everything).

---

## Phase Summary

| Phase | Deliverable | Key guarantee |
|-------|-------------|---------------|
| 1 | Broker on relational DB | G1/G2/G3 named |
| 2 | Log-structured / LSM storage | G1/G2/G3 preserved |
| 3 | Stateless API + LB + auto-scale | horizontal scale |
| 4 | WebSocket transport + presence | <100ms, presence |
| 5 | Distributed locking | ordering under concurrency |
| 6 | Message queues | offline push, sync, async |
| 7 | Group fan-out (1024) | O(online) send path |
| 8 | Sharding, replicas, tiering, HA | 99.99% uptime |

---

## TRAINING_GAP

<!-- TRAINING_GAP: The training transcripts do not specify the exact wire protocol / frame format for the WebSocket transport (e.g., protobuf vs JSON, heartbeat frame schema). TODO: define once an approved source covers it. -->
<!-- TRAINING_GAP: The training transcripts do not give a concrete LSM compaction algorithm (leveled vs size-tiered) or its tuning parameters. TODO: define once an approved source covers it. -->
<!-- TRAINING_GAP: The training transcripts do not specify the exact auto-scaling thresholds beyond the general "define an auto scaling policy" guidance. TODO: confirm thresholds against an approved source. -->
