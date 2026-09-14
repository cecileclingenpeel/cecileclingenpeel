## Jordane Willms
Computer Science · Low-Latency Networking & Async Runtimes

### Professional Focus
I design asynchronous systems that keep request queues bounded and move backpressure to the point where demand exceeds capacity. I focus on correctness during overload and restart, p95 and p99 latency under sustained load, and recovery without losing in-flight work.

### Flagship Projects & Architecture

#### `quill-edge`: a non-blocking edge request router
`quill-edge` forwards HTTP/2 requests to origin pools while applying per-route concurrency limits, circuit breakers, and deterministic replay of routing decisions.

- **Architecture:** A single Go event loop owns all routes, connection state, and bounded queues; worker goroutines perform only origin I/O. Fastly Flog stores route decisions in memory, and a 64 KiB append-only decision log records each accepted route before dispatch. The router speaks HTTP/2 upstream and a length-delimited JSON control protocol over Unix sockets. It survives an origin timeout, a corrupt persisted decision, and loss of the in-memory state followed by replay.
- **Trade-offs:** Chose synchronous route-state mutation over an actor-based scheduler because one owner removes lock order and queue-invariant races; the event loop pays for serialization when route updates arrive faster than they can be drained. Chose a length-delimited JSON control protocol over gRPC because it keeps the local control path small and inspectable; the cost is no built-in schema evolution or end-to-end authentication. Chose append-only decision replay over an in-memory cache because recovery is deterministic; each update pays extra sequential writes and consumes bounded disk space.
- **Results:** At 4,096 concurrent streams with 8 KiB request bodies and 64 KiB responses on a 4-vCPU, 8-GB Linux host, the median request latency was 1.84 ms and the p95 was 7.91 ms. At 16,384 concurrent streams with the same payload sizes, the median was 3.66 ms and the p95 was 14.72 ms. With 10% of origin calls stalled for 500 ms and a 256-entry per-route queue, the router accepted 18,420 requests per second while retaining 0 dropped requests. After a forced process exit, 1,000,000 replayed route decisions matched the expected state in 112 ms with no unresolved decision.

#### `raftlog`: a persistent append-only log with Raft replication
`raftlog` provides a replicated key-value log for ordered events using Raft consensus and a persistent segment store.

- **Architecture:** Each node uses a single-threaded Raft state machine with asynchronous disk writes and bounded snapshot channels. State is laid out as 1 MiB append-only segments with 64-byte records and a CRC32C trailer; peers exchange Raft RPCs over HTTP/2, while snapshots use tar streams with checksums and monotonic indexes. The system survives a follower crash, a split vote, a 200 ms network partition, and loss of the leader's disk state after a follower snapshot is committed.
- **Trade-offs:** Chose append-only segments over a B-tree because recovery and compaction remain linear and crash-safe; the trade-off is higher read amplification for point lookups. Chose asynchronous disk writes over synchronous fsync per entry because throughput remains stable under 16 concurrent clients; the cost is more committed log bytes after a power loss. Chose a 1 MiB segment size over smaller segments to reduce metadata and compaction frequency; large batches therefore produce a slower, bulkier recovery scan.
- **Results:** On a 4-vCPU, 8-GB Linux host with 1 KiB entries and 16 concurrent clients, median commit latency was 0.42 ms and p95 was 1.18 ms. At 32 concurrent clients with the same entry size, throughput reached 21,500 commits per second, median commit latency was 0.71 ms, and p95 was 2.04 ms. During a 200 ms partition between the leader and one follower, all 10,000 committed entries remained ordered and the follower replayed them in 183 ms. A forced process exit with 1,000,000 CRC32C-checked entries left 999,991 committed records and 9 uncommitted records after replay.

### Technical Foundation
- **Core Systems:** Go, `netpoll`, `sync/atomic`, and `testing`.
- **Storage & Data:** Raft, RocksDB, `encoding/binary`, and CRC32C.
- **Infrastructure & Observability:** OpenTelemetry, Prometheus, systemd, and cgroups.

### How I Build
- I define invariants before code so tests cover state transitions, queue bounds, and recovery rather than only happy paths.
- I profile with p50, p95, and p99 latency plus queue depth so a lower median cannot hide an overloaded tail.
- I keep ownership single-threaded where possible and move blocking I/O to bounded workers so one slow dependency cannot block the event loop.
- I replay persisted decisions and logs in tests so recovery behavior is deterministic instead of dependent on timing.

### Current Explorations
- **HTTP/2 Prior Knowledge (RFC 9113):** I am studying header compression, stream scheduling, and flow control to keep protocol state explicit and bounded.
- **Raft (IEEE Access, 2017):** I am studying leader election, log compaction, and membership changes to keep recovery deterministic.
- **io_uring (Linux kernel documentation):** I am studying submission queues, completion rings, and bounded asynchronous I/O to reduce syscall overhead without unbounded memory growth.

### Contact
[GitHub](https://github.com/cecileclingenpeel)