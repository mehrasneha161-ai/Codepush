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
