# Acceptance Criteria

Each entry: AC-<SECTION_NO>-<n>, owner, section, statement, verify command, done-when condition.

AC-20-1, master-system-design, 20, The broker design names ordering (G1), exactly-once (G2), and presence (G3) guarantees and shows the relational-DB mechanism for each before the LSM replacement., `grep -E "G1|G2|G3" design/20-master-system-design.md`, All three guarantees appear with both a Phase 1 (relational) and Phase 2 (LSM) mechanism.

AC-20-2, master-system-design, 20, The design specifies WebSocket as the real-time transport for <100ms delivery and presence, with a comparison against Long Polling and SSE., `grep -E "WebSocket|Long Polling|SSE" design/20-master-system-design.md`, All three transports are compared and WebSocket is selected with a stated reason.

AC-20-3, master-system-design, 20, Distributed locking is defined at per-conversation granularity with mutual exclusion, atomic acquire, TTL, and fencing token., `grep -E "fencing|NX PX|per-conversation" design/20-master-system-design.md`, Lock granularity and all four required properties are present.

AC-20-4, master-system-design, 20, Group fan-out for 1024 members is hybrid (write fan-out for online, read fan-out for offline) with the tradeoff table., `grep -E "write fan-out|read fan-out|1024" design/20-master-system-design.md`, Both fan-out strategies are compared and the hybrid decision is stated.

AC-20-5, master-system-design, 20, Scalability covers sharding by conversation_id into mutually exclusive subsets, read replicas for history, tiered storage, and 99.99% uptime via replication., `grep -E "mutually exclusive|read replicas|tiered|99.99" design/20-master-system-design.md`, All four scalability/availability topics are present.

## AC-20-16   owner: master-system-design   section: design/20-master-system-design.md
Statement: The design must map each deferred extended requirement (group chat, multimedia, presence) to its full-requirement counterpart so the MVP-to-full path is traceable.
Verify:    grep -E "group" design/20-master-system-design.md && grep -E "multimedia|image|video" design/20-master-system-design.md && grep -E "presence|online|last seen" design/20-master-system-design.md
Done when: Each deferred item is listed and mapped to the full requirement it becomes.

## AC-20-32   owner: master-system-design   section: design/20-master-system-design.md
Statement: The design must explicitly mark unresolved items as TRAINING_GAP rather than inventing details for the WebSocket frame format, the LSM compaction algorithm, and the exact auto-scaling thresholds.
Verify:    grep -E "TRAINING_GAP" design/20-master-system-design.md && grep -cE "TRAINING_GAP:" design/20-master-system-design.md
Done when: At least the three named gaps (wire protocol, compaction algorithm, auto-scaling thresholds) are marked as gaps with a TODO.

## AC-20-31   owner: master-system-design   section: design/20-master-system-design.md
Statement: The design must be organized into explicit phases with a phase summary table mapping each phase to a deliverable and a key guarantee.
Verify:    grep -E "## Phase Summary" design/20-master-system-design.md && grep -E "^\| 8 \|" design/20-master-system-design.md
Done when: A phase summary table exists listing every phase with its deliverable and key guarantee.

## AC-20-30   owner: master-system-design   section: design/20-master-system-design.md
Statement: Hot partition handling must be addressed with at least two mitigations — splitting a hot conversation's log by seq range and moving the hot conversation to a dedicated shard via static mapping.
Verify:    grep -E "sub-partitions by .seq. range" design/20-master-system-design.md && grep -E "dedicated shard" design/20-master-system-design.md
Done when: Both mitigations are present and the static-mapping move is characterized as a data migration rather than a full rehash.

## AC-20-29   owner: master-system-design   section: design/20-master-system-design.md
Statement: The design must state the availability math for 99.99% (approximately 52 minutes downtime per year) and require over-provisioning headroom so a single node failure does not push remaining nodes past capacity.
Verify:    grep -E "52 minutes downtime/year" design/20-master-system-design.md && grep -E "headroom" design/20-master-system-design.md
Done when: The downtime budget is stated and over-provisioning is required as the mitigation.

## AC-20-28   owner: master-system-design   section: design/20-master-system-design.md
Statement: 99.99% uptime must be supported by leader+follower replication per shard with follower promotion on leader failure, and the fencing token must be named as the safety property preventing a demoted leader from appending.
Verify:    grep -E "leader \+ followers" design/20-master-system-design.md && grep -E "prevents a demoted leader from continuing to append" design/20-master-system-design.md
Done when: The replication and promotion model is stated and fencing is tied to failover safety.

## AC-20-27   owner: master-system-design   section: design/20-master-system-design.md
Statement: Tiered storage must define at least three tiers (hot, warm, cold) with contents, access latency, and cost, and must place media/files in blob storage referenced by URL while message bodies stay in the log.
Verify:    grep -E "Hot \(LSM|Warm \(infrequent access\)|Cold \(glacier-class\)" design/20-master-system-design.md && grep -E "media is referenced by URL" design/20-master-system-design.md
Done when: All three tiers are defined with latency and cost, and the media-vs-body split is stated.

## AC-20-26   owner: master-system-design   section: design/20-master-system-design.md
Statement: History reads must be routed to read replicas per shard while the master handles only writes, separating the read path from the write path.
Verify:    grep -E "read replicas" design/20-master-system-design.md && grep -E "master handles only writes" design/20-master-system-design.md
Done when: The replica routing for history is stated and the master is scoped to the write path.

## AC-20-25   owner: master-system-design   section: design/20-master-system-design.md
Statement: The design must acknowledge that sharding requires the API server to know which shard to forward to, and must describe this as a routing table lookup rather than a broadcast.
Verify:    grep -E "API server must know which shard to forward to" design/20-master-system-design.md && grep -E "routing table lookup, not a broadcast" design/20-master-system-design.md
Done when: The routing consequence of sharding is stated and characterized as a lookup.

## AC-20-24   owner: master-system-design   section: design/20-master-system-design.md
Statement: Masters must be sharded by conversation_id into mutually exclusive subsets so a conversation lives on exactly one shard, preserving ordering without cross-shard coordination.
Verify:    grep -E "mutually exclusive" design/20-master-system-design.md && grep -E "hash\(conversation_id\) % num_shards" design/20-master-system-design.md
Done when: The shard function is given and the design states a conversation lives on exactly one shard.

## AC-20-23   owner: master-system-design   section: design/20-master-system-design.md
Statement: The design must state that the hybrid fan-out keeps the send path O(online members) rather than O(1024), and must reject pure write fan-out and pure read fan-out with a stated reason for each.
Verify:    grep -E "O\(online members\)" design/20-master-system-design.md && grep -E "Why not pure write fan-out|Why not pure read fan-out" design/20-master-system-design.md
Done when: The send-path complexity claim is present and both rejected alternatives have a stated reason.

## AC-20-22   owner: master-system-design   section: design/20-master-system-design.md
Statement: Group fan-out for 1024 members must be hybrid — write fan-out for online members and read fan-out (cursor pull) for offline members — with a tradeoff table comparing the two strategies.
Verify:    grep -E "write fan-out|read fan-out" design/20-master-system-design.md && grep -E "1024" design/20-master-system-design.md
Done when: Both strategies are compared in a table and the hybrid split by online/offline is stated as the decision.

## AC-20-21   owner: master-system-design   section: design/20-master-system-design.md
Statement: Every dequeued message must carry a receipt handle and deletion must require it, so a consumer that did not read a message cannot delete it.
Verify:    grep -E "receipt handle" design/20-master-system-design.md && grep -E "cannot delete it" design/20-master-system-design.md
Done when: The receipt-handle requirement is stated and tied to preventing cross-worker deletion of in-flight messages.

## AC-20-20   owner: master-system-design   section: design/20-master-system-design.md
Statement: The high-fan-out notification path must combine Kafka's bounded parallelism with SQS fan-out, where each Kafka consumer round-robins into M SQS queues each with an independent worker pool.
Verify:    grep -E "Kafka \(N partitions, N consumers\)" design/20-master-system-design.md && grep -E "round-robin into M SQS queues" design/20-master-system-design.md
Done when: The Kafka-to-SQS fan-out pattern is shown and the reason (Kafka parallelism bounded by partition count) is stated.

## AC-20-19   owner: master-system-design   section: design/20-master-system-design.md
Statement: Message queue usage must be split into three distinct uses — offline push notifications, multi-device sync, and async persistence — with the sync queue carrying notifications rather than message bodies.
Verify:    grep -E "Offline push notifications|Multi-device sync|Async persistence" design/20-master-system-design.md && grep -E "notify the device that new messages exist, not to carry the messages" design/20-master-system-design.md
Done when: All three uses are enumerated and the sync use is explicitly a notification, with the device pulling by cursor.

## AC-20-18   owner: master-system-design   section: design/20-master-system-design.md
Statement: The design must state that on the single-writer-per-shard LSM layer the lock is only needed at the shard leader and during leader failover, not on the steady-state append path.
Verify:    grep -E "only needed at the shard leader" design/20-master-system-design.md && grep -E "leader failover" design/20-master-system-design.md
Done when: The interaction between locking and the single-writer log is stated, with the lock scoped to failover.

## AC-20-17   owner: master-system-design   section: design/20-master-system-design.md
Statement: Every log write must carry a fencing token and the log must reject a write whose token is lower than the highest token seen for that conversation.
Verify:    grep -E "fencing token" design/20-master-system-design.md && grep -E "rejects a write whose token is lower" design/20-master-system-design.md
Done when: The fencing rule is stated as a rejection condition at the log, preventing a stale holder from writing an out-of-order `seq`.

## AC-20-16   owner: master-system-design   section: design/20-master-system-design.md
Statement: Lock acquire must use set-if-not-exists with expiry semantics and release must verify the token before deleting, so a holder cannot release another holder's lock.
Verify:    grep -E "SET lock:cid token NX PX" design/20-master-system-design.md && grep -E "only release your own lock" design/20-master-system-design.md
Done when: The acquire uses NX PX semantics and the release is guarded by a token comparison.

## AC-20-15   owner: master-system-design   section: design/20-master-system-design.md
Statement: The lock manager must be specified by the four required properties — mutual exclusion, atomic acquire, TTL/auto-release, and fencing — and must state that any system providing these four can serve as the lock manager.
Verify:    grep -E "Mutual exclusion|Atomic acquire|TTL / auto-release|Fencing" design/20-master-system-design.md && grep -E "If a candidate system provides these four" design/20-master-system-design.md
Done when: All four properties are enumerated and the design states the properties, not Redis specifically, are the requirement.

## AC-20-14   owner: master-system-design   section: design/20-master-system-design.md
Statement: Distributed locking must be defined at per-conversation granularity, not global and not per-message, with the reason that ordering is only defined within a conversation.
Verify:    grep -E "Lock per conversation, not per message and not global" design/20-master-system-design.md
Done when: The granularity decision is stated and justified by the scope of the ordering guarantee.

## AC-20-13   owner: master-system-design   section: design/20-master-system-design.md
Statement: API servers must be declared stateless with all correctness-relevant state held in broker storage and the presence store, and an auto-scaling policy must define both a scale-out and a scale-in signal.
Verify:    grep -E "stateless" design/20-master-system-design.md && grep -E "Scale-out|Scale-in" design/20-master-system-design.md
Done when: The stateless claim is stated with where state actually lives, and both scale-out and scale-in triggers are defined.

## AC-20-12   owner: master-system-design   section: design/20-master-system-design.md
Statement: The load balancer core must be shown as a small reverse-proxy implementation that picks a backend and forwards, with health checks, connection draining, and TLS termination named as production additions.
Verify:    grep -E "class LoadBalancer" design/20-master-system-design.md && grep -E "health checks|connection draining|TLS termination" design/20-master-system-design.md
Done when: The core pick/forward logic is present and the production additions are explicitly listed as beyond the core.

## AC-20-11   owner: master-system-design   section: design/20-master-system-design.md
Statement: The design must compare WebSocket, Long Polling, and SSE on delivery latency, server-to-client direction, presence fit, and cost, and select WebSocket with a stated reason.
Verify:    grep -E "WebSocket|Long Polling|SSE" design/20-master-system-design.md && grep -E "<100ms" design/20-master-system-design.md
Done when: All three transports appear in the comparison table and the decision section names WebSocket as the choice with the <100ms and presence-signal justification.

## AC-20-10   owner: master-system-design   section: design/20-master-system-design.md
Statement: Phase 2 must state that the custom log-structured/LSM layer is correct if and only if it satisfies G1, G2, and G3, and must derive `seq` from the append-only log offset rather than a separate counter.
Verify:    grep -E "if and only if it satisfies G1, G2, G3" design/20-master-system-design.md && grep -E "seq = log_offset" design/20-master-system-design.md
Done when: The correctness criterion is stated and `seq` is shown to be monotonic by construction from the single-writer append-only log.

## AC-20-09   owner: master-system-design   section: design/20-master-system-design.md
Statement: Presence must be declared eventually consistent, stored outside the message log, and driven by WebSocket connect/disconnect plus a heartbeat.
Verify:    grep -E "eventually consistent" design/20-master-system-design.md && grep -E "on WS connect|on WS disconnect|heartbeat" design/20-master-system-design.md
Done when: The design states presence is eventually consistent, gives the connect/disconnect upsert, and specifies a heartbeat interval that marks a user offline on miss.

## AC-20-08   owner: master-system-design   section: design/20-master-system-design.md
Statement: Exactly-once delivery must be enforced by a unique idempotency key on `(conversation_id, client_msg_id)` in Phase 1 and by a `client_msg_id -> seq` secondary index in Phase 2.
Verify:    grep -E "UNIQUE \(conversation_id, client_msg_id\)" design/20-master-system-design.md && grep -E "client_msg_id -> seq" design/20-master-system-design.md
Done when: Both the relational unique constraint and the LSM idempotency index are present, and the design states a retry returns the existing `seq` without appending.

## AC-20-07   owner: master-system-design   section: design/20-master-system-design.md
Statement: The Phase 1 schema must define a per-conversation monotonic `seq` as part of the messages primary key, so ordering is enforced by the storage layer rather than by the client.
Verify:    grep -E "seq +BIGINT NOT NULL" design/20-master-system-design.md && grep -E "PRIMARY KEY \(conversation_id, seq\)" design/20-master-system-design.md
Done when: The `messages` table declares `seq` as NOT NULL and includes `(conversation_id, seq)` as its primary key.

## AC-20-06   owner: master-system-design   section: design/20-master-system-design.md
Statement: The design must name all three guarantees (G1 ordering, G2 exactly-once, G3 presence) and show both a Phase 1 relational mechanism and a Phase 2 LSM mechanism for each.
Verify:    grep -E "G1|G2|G3" design/20-master-system-design.md && grep -E "PRIMARY KEY \(conversation_id, seq\)|UNIQUE \(conversation_id, client_msg_id\)|append-only log offset" design/20-master-system-design.md
Done when: All three guarantee labels appear, and the Phase 2 guarantee checklist table maps each to a relational mechanism and an LSM mechanism.

## AC-20-15   owner: master-system-design   section: design/20-master-system-design.md
Statement: The design must state whether sharding is required based on the scale estimate, and if so, name the shard key.
Verify:    grep -E "shard" design/20-master-system-design.md && grep -E "conversation_id|user_id" design/20-master-system-design.md
Done when: The sharding decision is explicit and a shard key is named.

## AC-20-14   owner: master-system-design   section: design/20-master-system-design.md
Statement: The design must justify the SQL vs NoSQL choice for the message store with an explicit decision and reasoning.
Verify:    grep -E "SQL|NoSQL|relational" design/20-master-system-design.md && grep -E "justif|because|reason" design/20-master-system-design.md
Done when: A storage choice is made and the justification is stated.

## AC-20-13   owner: master-system-design   section: design/20-master-system-design.md
Statement: The design must include a text description of the HLD architecture diagram covering client, API tier, broker/storage, and real-time transport.
Verify:    grep -E "architecture|diagram" design/20-master-system-design.md && grep -E "client" design/20-master-system-design.md && grep -E "API" design/20-master-system-design.md && grep -E "broker|storage" design/20-master-system-design.md
Done when: A textual architecture description names the client, API tier, storage/broker, and transport layers.

## AC-20-12   owner: master-system-design   section: design/20-master-system-design.md
Statement: The API section must define sendMessage, getMessages, and createGroup with a max group size of 1024.
Verify:    grep -E "sendMessage" design/20-master-system-design.md && grep -E "getMessages" design/20-master-system-design.md && grep -E "createGroup" design/20-master-system-design.md && grep -E "1024" design/20-master-system-design.md
Done when: All three APIs are defined and the 1024 group cap is stated.

## AC-20-11   owner: master-system-design   section: design/20-master-system-design.md
Statement: The NFRs must state <100ms delivery latency, 99.99% availability, and strict message ordering consistency.
Verify:    grep -E "100ms|100 ms" design/20-master-system-design.md && grep -E "99.99" design/20-master-system-design.md && grep -E "ordering|order" design/20-master-system-design.md
Done when: All three NFRs appear with their target values.

## AC-20-10   owner: master-system-design   section: design/20-master-system-design.md
Statement: The design must state that a system that is both read and write heavy cannot be optimized perfectly for both, and must name the storage compromise taken.
Verify:    grep -E "compromise|trade.?off" design/20-master-system-design.md && grep -E "read" design/20-master-system-design.md && grep -E "write" design/20-master-system-design.md
Done when: The compromise is explicitly named and tied to the read/write-heavy analysis.

## AC-20-09   owner: master-system-design   section: design/20-master-system-design.md
Statement: The design must analyze read vs write heaviness, stating messenger is roughly 1:1 to 2:1 read:write and contrasting it with Instagram's write-once-read-many pattern.
Verify:    grep -E "1:1|2:1|read.heavy|write.heavy" design/20-master-system-design.md && grep -E "Instagram" design/20-master-system-design.md
Done when: The read:write ratio is stated and the Instagram contrast is present.
