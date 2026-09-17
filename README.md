## Lolicia Tartar
Computer Science · Database Internals & Storage Engines

### Professional Focus
I design storage engines and the distributed services that run on them, with emphasis on crash-safe invariants, bounded memory, recovery time, and predictable tail latency. My work starts from the on-disk layout and the failure domain, then selects the concurrency model and protocol that keep those constraints measurable.

### Flagship Projects & Architecture

#### MerkleKV
A single-node, append-only key-value engine that exposes a linearizable snapshot API over an LSM tree.

- **Architecture:** A parser maps newline-delimited `key<TAB>value<NUL>` records into a 1 MiB memtable. A background compactor promotes immutable memtables into level-0 SSTables and merges them with lower levels. A read path copies only the active memtable and the current level-0 frontier into a slab arena, then releases the lock; no heap allocation occurs during lookup. The wire format is a 2-byte big-endian frame length followed by a command and variable-length key and value. The engine exposes `PUT`, `GET`, `DELETE`, `SNAPSHOT`, `COMPACTION`, and `CHECKPOINT` through a length-delimited socket protocol.
- **Trade-offs:** I chose append-only SSTables over in-place B-tree updates to make crash recovery a matter of validating record checksums and checkpoint metadata; the cost is level compaction and extra disk bandwidth. I chose small memtables and a bounded compaction queue over an unbounded write buffer to cap read-amplification and memory growth; the cost is backpressure when compaction cannot keep pace. I chose stable fixed-width record headers over a self-describing format to reduce parsing work and preserve locality; the cost is an explicit schema version and a migration path.
- **Results:** On a 32-thread AMD EPYC 7542 host running a release build with a 64 KiB payload and a 4 KiB key, 10,000-key linearizable point reads reached p50 11.8 µs, p95 18.4 µs, and p99 27.1 µs. On the same host with a 128 KiB payload, a 1 GiB `PUT` stream sustained 94.2 MiB/s at 16 concurrent writers while the process RSS stayed below 412 MiB. During a forced crash at 32 MiB of uncheckpointed writes, all 20 recovery runs reconstructed the latest checkpointed state in 1.9 to 2.7 seconds. A 512 MiB level-0-to-level-1 compaction of 409,600 records completed in 7.4 seconds and left a read p99 of 29.6 µs under a concurrent 10% read load.

#### FaultLattice
A deterministic, in-memory service-mesh simulator that models packet loss, process delay, partitioning, and replayable command histories.

- **Architecture:** The control plane stores command histories in a circular ring buffer and assigns each event a vector clock. A deterministic scheduler advances virtual time in 1 ms ticks, dispatches bounded per-node queues, and records every input and random draw. State machines serialize their durable transition log before acknowledging a command, while a separate snapshot journal supports replay without retaining the full history. A JSON Lines control channel carries `START`, `STOP`, `FAULT`, `REPLAY`, `SNAPSHOT`, and `COMPARE` commands; the simulator reports per-node state hashes and causal dependencies. The failure model includes dropped packets, duplicate delivery, delayed messages, split brain, and partial application of a transition log.
- **Trade-offs:** I chose a tick-driven deterministic scheduler over a wall-clock event loop so test order and virtual-time order are identical; the cost is that long-running simulations advance in fixed 1 ms quanta and need virtual-time compression. I chose in-memory state machines with periodic snapshots over persistent node databases to keep replay fast and isolate the mesh model from storage noise; the cost is that crash recovery is modeled explicitly rather than delegated to a database. I chose a JSON Lines control channel over a binary protocol to make traces inspectable; the cost is higher parsing and serialization work for large histories.
- **Results:** On a 16-thread Intel Xeon Gold 6248 host running a release build with 256 simulated nodes, a 10,000-command history replayed at 1.28 million commands/s while holding 384 MiB of process memory. Across 500 runs with a 5% packet-loss rate and 10 ms message delay, every replayed history produced the same final state hash. Under a 20% packet-loss and 50 ms delay fault, the simulator completed 200-node recovery simulations in 4.6 to 5.9 seconds using bounded queues of 256 commands per node. A 1,000,000-command fault injection run produced a deterministic p50 transition latency of 0.81 ms, p95 1.14 ms, and p99 1.52 ms at a 1 ms simulation tick.

### Technical Foundation
- **Storage & Data:** `RocksDB` for comparison against level compaction and cache behavior, `snappy` for bounded CPU-heavy compression, and `etcd` for linearizable coordination experiments.
- **Concurrency & Testing:** `liburing` for measured asynchronous block I/O, `prometheus-client` for process and engine metrics, and `valgrind memcheck` for memory-error checks.
- **Infrastructure & Observability:** `systemd` for reproducible service launch and crash restarts, `eBPF` for syscall and scheduling traces, and `Prometheus` with `Grafana` for latency and recovery dashboards.

### How I Build
- **Define the invariant before the API:** A storage engine is useful only when its on-disk state, snapshot, and recovery path satisfy the same contract.
- **Bound every queue and buffer:** Backpressure makes overload visible and keeps memory growth independent of input rate.
- **Measure percentiles with a fixed workload:** A median hides the tail that users experience, so each result names concurrency, payload, host, and build profile.
- **Replay failures deterministically:** A fault is actionable only when the same history, scheduler, and random seed reproduce it.

### Current Explorations
- **Raft log compaction and snapshot transfer:** Taking the safety argument and the leader-append flow from the Raft paper, then comparing snapshot cost with log truncation in a fault-injection harness.
- **RFC 9110, Hypertext Transfer Protocol (HTTP/1.1):** Taking the framing and connection rules for a length-delimited command protocol, with explicit treatment of partial reads and aborted frames.
- **Linux cgroup v2 memory controller:** Taking the memory.max and memory.events interface to turn allocator pressure into a reproducible backpressure experiment.

### Contact
[Lolicia Tartar](https://github.com/LoliciaTartar) · [GitHub no-reply email](mailto:loliciatartar@users.noreply.github.com)