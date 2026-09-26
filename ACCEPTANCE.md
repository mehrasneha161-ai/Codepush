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
