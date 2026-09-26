# Design Spine

One paragraph: what is being built.

## Module Map

| Module | Owner | Section | Status |
|--------|-------|---------|--------|
| Distributed Messaging Core | master-system-design | 20 | Draft |

## Statements

| ID | Statement | Owner | Status | Decided in |
|----|-----------|-------|--------|------------|
| S-20-1 | The broker is built on a relational DB first to name the required guarantees (ordering, exactly-once, presence) before replacing storage with a log-structured/LSM layer. | master-system-design | Draft | design/20-master-system-design.md |
| S-20-2 | Ordering (G1) is per-conversation, enforced by a monotonic `seq`; exactly-once (G2) is enforced by a `client_msg_id` idempotency key; presence (G3) is eventually consistent. | master-system-design | Draft | design/20-master-system-design.md |
| S-20-3 | WebSocket is chosen over Long Polling and SSE for <100ms delivery and presence, because connection lifecycle maps directly to online/last_seen. | master-system-design | Draft | design/20-master-system-design.md |
| S-20-4 | Distributed locking is per-conversation (not global, not per-message) and requires mutual exclusion, atomic acquire, TTL, and fencing. | master-system-design | Draft | design/20-master-system-design.md |
| S-20-5 | Group fan-out for 1024 members is hybrid: write fan-out for online members, read fan-out (cursor pull) for offline members. | master-system-design | Draft | design/20-master-system-design.md |
| S-20-6 | Masters are sharded by conversation_id into mutually exclusive subsets; history reads go to read replicas; media/history use tiered storage; 99.99% uptime via leader+follower replication and fencing. | master-system-design | Draft | design/20-master-system-design.md |

See ACCEPTANCE.md for the definition of done and DECISIONS.md for the decision log.
