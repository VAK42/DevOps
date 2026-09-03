## NGINX

### 1. Architecture + Process Model
- Master Process vs Worker Processes: Privilege Separation - Master Never Handles Connections
- Event-Driven | Async + Non-Blocking I/O Model
- `epoll`: Linux / `kqueue`: BSD / Event Ports: Solaris - OS-Level Readiness Notification
- Worker Process = Single-Threaded Event Loop Handling Thousands Of Connections: C10K Solution
- `worker_processes` ≈ CPU Cores | Max Clients = `worker_processes` * `worker_connections` / 2 In Reverse Proxy Mode | `worker_rlimit_nofile` ≥ `worker_connections` * 2
- NGINX vs Apache: Event Loop vs Thread / Process-Per-Connection
- Hot Reload / Zero-Downtime Binary Upgrade: `kill -USR2` - Master Spawns New Master - Old Drains
- Graceful Shutdown: `kill -WINCH` / `kill -QUIT`

### 2. Config System
- Context Hierarchy: Main → Events → HTTP → Server → Location: Inheritance + Override Rules
- Directive Processing Order + Merge Behavior
- `location` Matching Priority: Exact Match `=` > Preferential Prefix `^~` > Regex `~` / `~*`: In Order > Prefix Match: Longest
- `try_files` | Internal Redirects + Named Locations
- Variables: `$uri` | `$args` | `$request_uri` | `$host` + Evaluation Cost

### 3. Roles + Core Features
- Reverse Proxy: `proxy_pass` | Load Balancer | Static Web Server + API Gateway
- SSL/TLS Termination: Session Cache | Session Tickets | OCSP Stapling + TLS 1.3 0-RTT Tradeoffs
- HTTP/2 + HTTP/3: QUIC Support + Multiplexing Implications
- Caching: `proxy_cache` | `fastcgi_cache` | Cache Keys | Cache Locking: `proxy_cache_lock` | Stale-While-Revalidate + Cache Purge Strategies
- Rate Limiting Internals: `limit_req`: Leaky Bucket - `burst` + `nodelay` Semantics | `limit_conn`: Concurrent Connection Limiting
- Compression: Gzip / Brotli - CPU Tradeoff + Compression Level Tuning
- Header Manipulation | `proxy_set_header` + `add_header` Inheritance Gotchas
- `resolver` Directive - Dynamic Upstream DNS Resolution: Required In Kubernetes / Cloud Where Backend IPs Change - Without It NGINX Resolves Upstream Hostnames Once At Startup + Caches Forever: Stale IP Risk | `resolver valid=30s` Controls Re-Resolution Interval

### 4. Load Balancing
- Algorithms: Round Robin | `least_conn` | `ip_hash` | Hash: Consistent Hashing + Random With Two Choices
- Passive Health Checks: Built-In Via `max_fails` / `fail_timeout` vs Active Health Checks: Plus / Module-Based
- Upstream Keepalive Connections: `keepalive` Directive - Reduces TCP/TLS Handshake Overhead To Backends
- Session Persistence Strategies: Sticky Sessions Via `ip_hash` / Cookie
- Weighted + Backup Servers + Slow-Start

### 5. Buffering + Performance Internals
- Client Body / Header Buffers | Proxy Buffers: `proxy_buffering` | Buffer Size vs Disk Spooling
- Common 502 / 504 / 413 Root Causes: Buffer Exhaustion | Upstream Timeout + Body Size Limits
- `sendfile` | `tcp_nopush` + `tcp_nodelay`: Kernel-Level Zero-Copy File Serving
- `upstream` Zone - Shared Memory: `zone` Directive - Required For Accurate Rate Limiting + Health State Sharing Across All Worker Processes: Without It Each Worker Has Independent Counters
- Connection Reuse + File Descriptor Limits: `ulimit` Tuning

### 6. Security
- WAF Integration: ModSecurity | NAXSI
- DDoS Mitigation Patterns: Rate Limiting | Connection Limiting + Geo-Blocking
- Hide Server Tokens + Security Headers: HSTS | CSP | X-Frame-Options
- mTLS: Client Certificate Verification

### 7. High Availability + Scaling
- NGINX + Keepalived: VRRP-Based Active-Passive Failover
- Multi-Instance Behind DNS Round Robin / External Load Balancer
- NGINX As Ingress Controller In Kubernetes: Annotations-Driven Config Generation

### 8. Extensibility
- OpenResty: Embed Lua Via LuaJIT For Custom Logic Inside Request Lifecycle
- Njs: NGINX JavaScript Scripting Module
- Dynamic Modules vs Compiled-In Modules

### 9. Observability + Troubleshooting
- Access / Error Log Formats | Log Levels + Conditional Logging
- `stub_status` Module: Basic Metrics + Prometheus NGINX-Exporter
- Debugging Strategies: `strace` On Worker | Core Dumps + Request Tracing Via Headers

### 10. Expansion
- `stream` Module - L4: TCP / UDP Proxying / Load Balancing - Distinct From `http` Module
- Request Processing Phases: 11 Internal Phases: Post-Read | Server-Rewrite | Find-Config | Rewrite | Post-Rewrite | Preaccess | Access | Post-Access | Precontent | Content | Log - Where Modules Hook In
- `map` Module: Efficient Key-Value Lookup For Conditional Logic - Avoids `if` Performance Pitfalls
- `geo` / `split_clients` Modules: IP-Based Routing + A/B Testing / Canary At Edge
- gRPC Proxying: `grpc_pass` - HTTP/2 Requirement + WebSocket Proxying: Upgrade Header Handling + Long-Lived Connection Considerations
- SSL Session Resumption Internals: Session Cache vs Session Tickets - Stateful vs Stateless Resumption Tradeoffs + OCSP Must-Staple
- HTTP/3: QUIC Over UDP - 0-RTT Replay Attack Considerations
- Njs vs OpenResty / Lua - Performance + Use-Case Tradeoffs: Njs = Safer / Sandboxed Subset - Lua / LuaJIT = More Powerful But More Complexity
- Log Rotation Without Dropping Requests: `kill -USR1` Reopen Log Handles
- Unix Domain Socket vs TCP For Upstream: Performance Gain For Co-Located Backends
- Connection Draining During Reload / Deploy: Graceful Worker Shutdown Timing

---

## Kafka

### 1. Foundational Model
- Distributed Commit Log / Append-Only Pub-Sub System
- Topics → Partitions → Offsets: Ordering Guaranteed Only Within A Partition
- Producer / Consumer / Broker / Cluster
- ZooKeeper: Legacy Metadata / Coordination vs KRaft: Raft-Based Self-Managed Metadata Quorum - ZooKeeper Removal In Kafka 4.0

### 2. Storage Internals
- Partition = Physical Append-Only Log Split Into Segments: Active + Closed Segment Files
- Log Segment Structure: `.log` | `.index` + `.timeindex` Files
- Sequential Disk I/O + OS Page Cache Reliance: Avoids JVM Heap For Message Data
- Zero-Copy Transfer: `sendfile` Syscall - Kernel Copies Disk → Socket - Bypasses User Space
- Log Retention: Time-Based + Size-Based
- Log Compaction: Keyed Latest-Value Retention - Tombstones - Compacted Topics For Changelog / State Use Cases

### 3. Replication + Consensus
- Replication Factor: Leader / Follower Per Partition
- In-Sync Replica: ISR Set + `min.insync.replicas`
- Controller Node: KRaft Quorum Leader / Raft Log - Manages Partition Leadership + Metadata Propagation
- Unclean Leader Election: `unclean.leader.election.enable` - Availability vs Consistency Tradeoff
- High Watermark: HW - Offset Up To Which All ISR Replicas Have Replicated - Defines What Is Committed / Visible To Consumers
- Leader Epoch: Prevents Log Divergence After Leader Changes - Replaces Old HW-Truncation Issues
- ISR Shrink Trigger: `replica.lag.time.max.ms` - Replica Removed From ISR If It Falls Behind Leader By More Than This Duration: Key Tuning Knob For Durability vs Availability Balance

### 4. Producer Internals
- Partitioning Strategy: Key Hash: Murmur2 | Round Robin: Sticky Partitioner Default + Custom Partitioner
- `acks`: `0` / `1` / `all` / `-1` - Durability vs Latency Tradeoff
- Batching + `linger.ms` | `batch.size` + Compression Codecs: Gzip / Snappy / LZ4 / Zstd Tradeoffs
- Idempotent Producer: Producer ID: PID + Sequence Number Per Partition → Broker-Side Deduplication
- Transactions: `transactional.id` | Two-Phase-Commit Protocol | Transaction Coordinator + Atomic Writes Across Partitions / Topics
- Retry Semantics: `delivery.timeout.ms` | In-Flight Requests + Ordering Risk: `max.in.flight.requests.per.connection` > 1 With Retries

### 5. Consumer Internals
- Consumer Groups - Partition Assignment: One Partition Consumed By Exactly One Consumer Within A Group
- Partition Assignment Strategies: Range | Round Robin | Sticky + Cooperative Sticky
- Rebalancing: Eager: Stop-The-World - Revoke All vs Cooperative / Incremental: Minimal Disruption
- Group Coordinator + `__consumer_offsets` Internal Topic
- Offset Commit Strategies: Auto-Commit vs Manual: Sync / Async - At-Least-Once vs At-Most-Once Implications
- Static Membership: `group.instance.id` - Avoids Rebalance On Transient Restarts
- Poll Loop Mechanics: `max.poll.interval.ms` | `session.timeout.ms` + `heartbeat.interval.ms`

### 6. Delivery Semantics
- At-Most-Once | At-Least-Once + Exactly-Once: Idempotent Producer + Transactions + `read_committed` Isolation Level
- `read_committed` vs `read_uncommitted`: `read_committed` Consumers Only See Messages Up To The Last Stable Offset: LSO - Uncommitted Transactional Messages Are Hidden Until `COMMIT`: Prevents Reading Dirty / Aborted Data
- Exactly-Once Across Kafka Streams: Transactional Processing

### 7. Ecosystem
- Kafka Connect: Source / Sink Connectors | Distributed Mode | Offset / Configuration Storage Topics + CDC Via Debezium
- Kafka Streams: KStream / KTable Duality | Stateful Operations | State Stores: RocksDB-Backed | Changelog Topics + Exactly-Once Processing
- Schema Registry: Avro / Protobuf / JSON Schema | Compatibility Modes: Backward / Forward / Full + Schema Evolution Rules
- ksqlDB: SQL Abstraction Over Streams
- MirrorMaker 2: Cross-Cluster Replication + Active-Active / Active-Passive Disaster Recovery

### 8. Performance + Capacity Planning
- Partition Count Tradeoffs: Parallelism vs Metadata Overhead vs Rebalance Cost vs Open File Handles
- Throughput Tuning: `batch.size` | `linger.ms` | Compression + `fetch.min.bytes` / `fetch.max.wait.ms` On Consumer
- Broker Tuning: `num.network.threads` | `num.io.threads` + Socket Buffer Sizes
- Disk Strategy: JBOD vs RAID - Page Cache Sizing vs Heap Sizing: Small JVM Heap Recommended

### 9. Failure Modes + Operations
- Broker Failure → Leader Election → ISR Shrink / Expand
- Consumer Lag Monitoring + Causes: Slow Processing | Rebalancing Storms + GC Pauses
- Split-Brain Prevention Via Controller Epoch / KRaft Quorum
- Multi-Datacenter Strategies: Stretch Cluster vs MirrorMaker Active-Active
- Quotas: Producer / Consumer / Broker Throttling

### 10. Expansion
- Exactly-Once Internals: Transaction Coordinator | Transaction Log: `__transaction_state` | Control / Marker Messages: `COMMIT` / `ABORT` + Last Stable Offset: LSO - `read_committed` Consumers Only See Up To LSO
- Security: SASL: PLAIN / SCRAM / GSSAPI / OAUTHBEARER For Authentication | mTLS For Transport + ACLs: Resource-Level Authorization For Topic / Group / Cluster
- Multi-Tenancy Patterns: Quotas + ACLs + Naming Conventions Per Tenant
- Rack Awareness: `broker.rack` - Replica Placement Across Availability Zones For Fault Tolerance
- Tiered Storage: Offload Older Log Segments To Object Storage - Decouples Storage Cost From Compute / Retention
- Cruise Control: Automated Partition Rebalancing | Leader Distribution + Resource-Aware Reassignment
- Static Partition Assignment / Manual Partition Reassignment Tooling: `kafka-reassign-partitions`
- Message Format Versions + Compatibility During Broker / Client Upgrades
- Consumer Lag = Log End Offset - Consumer Committed Offset | Lag Monitoring Tools: Burrow | Kafka Lag Exporter
- JVM Tuning For Brokers: G1GC Recommended - Heap Sizing Small Relative To Page Cache Reliance
- Kafka Streams Exactly-Once V2: Simplified Transactional Protocol | RocksDB State Store Tuning + Punctuators: Time / Stream-Time Triggered Callbacks

---

## RabbitMQ

### 1. Foundational Model
- AMQP 0-9-1 Broker: Also Supports STOMP | MQTT Via Plugins
- Producer → Exchange → Binding → Queue → Consumer: Smart Broker - Routing-Based vs Kafka Commit Log Model
- Erlang/OTP Runtime Foundation: Actor-Model Concurrency + "Let It Crash" Supervision Trees

### 2. Exchange Types + Routing
- Direct: Exact Routing Key Match
- Fanout: Broadcast - Ignores Routing Key
- Topic: Wildcard Pattern: `*` = One Word - `#` = Zero+ Words
- Headers: Match On Header Key / Value Instead Of Routing Key
- Alternate Exchange: Fallback For Unroutable Messages

### 3. Queue Mechanics + Internals
- Bindings: Exchange ↔ Queue + Routing Key / Arguments
- Message Flow: Publish → Exchange Routes → Queue Target → Consumer Delivery
- Durable vs Transient Queues - Persistent vs Non-Persistent Messages: Delivery Mode
- Consumer Prefetch / QoS: `basic.qos` - Flow Control - Prevents Overwhelming Slow Consumers
- Manual ACK / NACK / Reject + Requeue Semantics
- Dead Letter Exchange: DLX - Routing On Reject / NACK / TTL Expiry / Queue Length Limit
- TTL: Per-Message + Per-Queue
- Priority Queues
- Lazy Queues: Disk-First - Low Memory Footprint For Large Backlogs

### 4. Reliability Guarantees
- Publisher Confirms: Async ACK From Broker On Persisted Message - Publisher-Side Reliability
- Consumer Acknowledgments: ACK / NACK - Consumer-Side Reliability
- Transactions: AMQP Transactions - Heavier - Generally Superseded By Publisher Confirms
- At-Least-Once Delivery Model: Deduplication Responsibility On Consumer Side

### 5. Clustering + High Availability
- Classic Clustering: Shared Schema / Metadata Across Nodes - Queue Lives On One Node Unless Mirrored
- Classic Mirrored Queues: Deprecated vs Quorum Queues: Raft Consensus - Recommended Modern HA
- Quorum Queue Internals: Raft Log Replication | Leader / Follower Per Queue + Automatic Leader Election On Failure
- Streams: RabbitMQ Streams - Log-Based - High-Throughput Replay Use Cases
- Federation: Loosely-Coupled Cross-Broker / Datacenter Message Forwarding
- Shovel: Point-To-Point Queue / Exchange Bridging Between Brokers

### 6. Flow Control + Resource Management
- Memory Alarms + Disk Alarms: Broker Blocks Publishers When Thresholds Exceeded
- Connection / Channel Limits
- Flow Control Backpressure Mechanism: Credit-Based Between Erlang Processes

### 7. Messaging Patterns
- Work Queues: Competing Consumers - Round-Robin Dispatch
- Pub/Sub: Fanout
- Routing: Direct / Topic Selective Consumption
- RPC Pattern: `reply-to` + `correlation-id`
- Delayed Message Exchange Plugin: Scheduled Delivery

### 8. Kafka vs RabbitMQ: Architecture Comparison
- Retention Model: Kafka Retains / Replays By Offset - RabbitMQ Deletes On ACK Unless Streams Used
- Throughput vs Routing Complexity Tradeoff
- Ordering Guarantees: Kafka Per-Partition - RabbitMQ Per-Queue With Caveats Under Multiple Consumers
- When To Choose Which: Event Streaming / Replay / Analytics: Kafka vs Task Queues / Complex Routing / RPC: RabbitMQ

### 9. Operations + Troubleshooting
- Queue Growth / Backlog Diagnosis: Consumer Down | Slow Consumer + Poison Message Loop
- Management UI + `rabbitmqctl` Diagnostics
- Split-Brain In Classic Clustering: Partition Handling Strategies: `pause_minority` | `autoheal` - `pause_minority` Stops The Minority Partition - `autoheal` Picks A Winner + Forces Others To Rejoin

### 10. Expansion
- Erlang/OTP VM Tuning: Scheduler Count | Distribution Buffer Size + Erlang Cookie Security For Inter-Node Authentication
- Message Store Internals: Per-Virtual-Host - Index vs Message Body Storage Separation
- Single Active Consumer: SAC Pattern - Exclusive Processing With Automatic Failover - Ordering Guarantee Use Case
- Consumer Cancellation Notifications: Detect Queue Deletion / Node Failure From Consumer Side
- Priority Queue Caveats: Limited Priority Levels Recommended - Performance Cost Of Many Priority Levels
- Stream Offset Tracking: RabbitMQ Streams Consumer Offset Storage - Replay Semantics Closer To Kafka
- AMQP 1.0 Support: RabbitMQ 4.X Native vs Legacy 0-9-1 - Interoperability Implications
- Plugin Ecosystem Depth: Management | Federation | Shovel | Delayed-Message-Exchange + Consistent-Hash-Exchange
- Memory / Disk Watermark Calculation Specifics + Tuning: `vm_memory_high_watermark` - Paging Behavior Under Pressure

---

## Redis

### 1. Core Model
- In-Memory Key-Value Store | Single-Threaded Command Execution: I/O Threading In Redis 6+ For Network Only - Not Command Execution
- Data Structures: String | List | Hash | Set | Sorted Set: ZSet - Skip List + Hash Table Internally | Bitmap | HyperLogLog: Probabilistic Cardinality | Stream + Geo

### 2. Persistence Internals
- RDB: Point-In-Time Snapshot: `fork` Syscall + Copy-On-Write Child Process Writes Snapshot - Parent Keeps Serving
- AOF: Append-Only Write Log | Rewrite / Compaction Process: `BGREWRITEAOF` + `fsync` Policy: `always` / `everysec` / `no`
- Hybrid Persistence: RDB Preamble + AOF Tail - Faster Recovery + Durability
- Recovery Time + Data-Loss Window Tradeoffs Between RDB / AOF / Hybrid

### 3. Replication
- Async Master-Replica Replication + Replication Backlog: Partial Resync Buffer
- PSYNC Protocol: Full Resync vs Partial Resync Using Replication ID + Offset
- Replica Read Scaling + `replica-read-only`
- `WAIT numreplicas timeout`: Synchronous Replication Barrier - Blocks Until N Replicas ACK / Timeout: Used For Durability-Critical Writes Without Full Sync Replication Overhead

### 4. High Availability + Scaling
- Redis Sentinel: Monitoring | Quorum-Based Failure Detection | Automatic Failover + Service Discovery For Clients
- Redis Cluster: Hash Slot Sharding: 16384 Slots | Gossip Protocol For Cluster State Propagation | Resharding + Redirection: MOVED / ASK Responses
- Cluster Failover - Slot Ownership Handoff + Split-Brain Considerations: `cluster-node-timeout`

### 5. Concurrency + Atomicity
- Single-Threaded Execution → Command-Level Atomicity Guarantee
- Transactions: `MULTI` / `EXEC` / `DISCARD` / `WATCH`: Optimistic Locking - Check-And-Set Semantics
- Lua Scripting: `EVAL` - Atomic Multi-Command Execution Server-Side + Redis Functions: 7.0+ Persisted Scripts
- Client-Side Caching / Tracking: RESP3 Protocol Invalidation Push

### 6. Eviction + Memory Management
- `maxmemory-policy` Values: `noeviction` | `allkeys-lru` | `volatile-lru` | `allkeys-lfu` | `volatile-lfu` | `allkeys-random` | `volatile-random` + `volatile-ttl`
- Approximate LRU / LFU Implementation: Sampling-Based - Not Exact
- Memory Fragmentation | `MEMORY DOCTOR` + `MEMORY USAGE` Diagnostics

### 7. Advanced Use Cases
- Caching Patterns: Cache-Aside | Write-Through | Write-Behind + TTL-Based Invalidation
- Cache Stampede Prevention: Mutex Lock / Probabilistic Early Expiration: XFetch / Request Coalescing
- Distributed Locks - Redlock Algorithm: Multi-Instance Quorum Lock - Clock Drift / GC Pause Considerations - Kleppmann Critique: Redlock Is Unsafe Under Process Pauses / Clock Jumps: Use Fencing Tokens For True Safety - Redlock Remains Useful As Best-Effort Lock - Not For Correctness-Critical Mutual Exclusion
- Rate Limiting: Token Bucket / Sliding Window Via `INCR` + `EXPIRE` / Lua Scripts
- Redis Streams: Consumer Groups | `XACK` | `XCLAIM` | `XPENDING` - Durable Message Queue Semantics vs Simple Pub/Sub
- Leaderboards: Sorted Set - O(log N) Rank Operations

### 8. Performance
- Pipelining: Batch Commands - Amortize Round-Trip Latency
- Connection Pooling + `CLIENT NO-EVICT`
- Big Key Problem: Blocking Operations On Huge Collections + Hot Key Problem: Single-Key Overload Despite Cluster Sharding
- `SCAN` vs `KEYS`: Cursor-Based Non-Blocking Iteration

### 9. Failure Modes
- Redis As Primary Datastore Without Persistence Strategy = Data Loss Risk
- Blocking Commands In Production: `KEYS *` | `FLUSHALL` + Unbounded `SORT`
- Thundering Herd On Cache Expiry
- Cluster Resharding Downtime / Latency Spikes

### 10. Expansion
- Internal Object Encodings: `OBJECT ENCODING` - Listpack / Ziplist / Intset / Skiplist Transitions Based On Size Thresholds - Memory Efficiency Implications
- RDB `fork` Syscall Copy-On-Write Memory Doubling Risk Under High Write Load During Snapshot
- Active Defragmentation: `activedefrag` - Mitigates Memory Fragmentation From Allocator Behavior Over Time
- ACL System: Redis 6+ - Fine-Grained Command / Key-Pattern Permissions Per User
- TLS Support: Client-Server + Replication Encryption
- Diskless Replication: Replica Sync Via Socket Instead Of RDB File - Reduces Disk I/O On Full Resync
- Keyspace Notifications: Pub/Sub On Key Expiry / Eviction Events - Event-Driven Architectures
- Hash Tags + `CROSSSLOT` Errors: Cluster Mode Multi-Key Operation Constraints - `{tag}` Forcing Same-Slot Placement
- Redis Stack Modules: RedisJSON | RediSearch: Secondary Indexing / Full-Text | RedisTimeSeries + RedisGraph
- `CLIENT PAUSE`: Coordination Primitive For Failover / Maintenance Windows
- Redis Functions: 7.0+ Persisted / Versioned Server-Side Functions Replacing Ad-Hoc `EVAL` Scripts

---

## Docker

### 1. Foundational Concepts
- OS-Level Virtualization: Shares Host Kernel vs VM: Hypervisor + Full Guest OS
- Image: Immutable | Layered | Read-Only vs Container: Image + Writable Layer - Runtime Instance
- OCI: Open Container Initiative Spec - Image Spec + Runtime Spec: Portability Standard
- Docker Engine Architecture: `dockerd`: Daemon → `containerd`: High-Level Runtime → `runc`: Low-Level OCI Runtime - Creates Actual Container Via Namespaces / Cgroups → `shim`: Per-Container Process Supervision

### 2. Filesystem + Storage
- Union Filesystem - OverlayFS: Lowerdir / Upperdir / Merged - Copy-On-Write Layering
- Image Layer Caching + Content-Addressable Storage: SHA256 Layer Digests
- Volumes: Docker-Managed - Persistent - Best Practice For Stateful Data
- Bind Mounts: Host Path Direct Mapping
- Tmpfs Mounts: In-Memory - Ephemeral - No Disk I/O

### 3. Dockerfile + Image Building
- Instruction Set: FROM | RUN | COPY | ADD | WORKDIR | ENV | ARG | EXPOSE | CMD vs ENTRYPOINT: Semantic Difference + Override Behavior
- Layer Caching Strategy - Order Instructions Least → Most Frequently Changed
- Multi-Stage Builds - Separate Build / Runtime Environments - Minimal Final Image
- BuildKit: Parallel Layer Builds | Cache Mounts + Secret Mounts Without Baking Into Layers
- `.dockerignore` + Image Size Optimization: Distroless / Alpine / Scratch Base Images

### 4. Isolation Primitives
- Linux Namespaces: PID | NET | MNT | UTS | IPC | USER | CGROUP - What Each Isolates
- Cgroups: V1 vs V2 - Resource Limiting / Accounting: CPU | Memory | I/O | PIDs
- Capabilities: Fine-Grained Root Privilege Subset - Drop Unnecessary Capabilities
- Seccomp Profiles: Syscall Filtering + AppArmor / SELinux: MAC - Mandatory Access Control

### 5. Networking
- Bridge: Default Isolated Network + NAT | Host: Share Host `netns` | None | Overlay: Multi-Host - Swarm / Kubernetes + `macvlan`
- Container-To-Container DNS Resolution On User-Defined Bridge Networks
- Port Publishing: `-p` - `iptables` DNAT Rules Under The Hood

### 6. Registry + Distribution
- Registry API | Image Manifests + Manifest Lists: Multi-Arch Images
- Tagging Strategy: SemVer vs `latest` Anti-Pattern + Immutable Tags
- Image Signing + Provenance: Notary / Cosign - Supply Chain Security: SBOM

### 7. Security
- Rootless Containers + Non-Root USER In Dockerfile
- Image Scanning: Trivy | Grype | Snyk - CVE Detection In Layers
- Read-Only Root Filesystem | Dropped Capabilities + No-New-Privileges
- Secrets Management: Avoid `ARG` / `ENV` Baking - Use BuildKit Secret Mounts / External Vault At Runtime
- Supply Chain: Base Image Provenance + Minimal Attack Surface

### 8. Orchestration Basics
- Docker Compose - Multi-Container Local Orchestration | Service Dependency: `depends_on` + Networks / Volumes Declaration
- Docker Swarm - Native Clustering - Simpler Alternative To Kubernetes - Raft-Based Manager Consensus

### 9. Resource Management + Limits
- CPU Shares / Quota / Period + Memory Limits: Hard OOM Kill vs Soft Limits
- PID Limits: Fork Bomb Protection
- `--restart` Policies: `no` / `on-failure` / `always` / `unless-stopped`

### 10. Operational Pitfalls
- Not Using Multi-Stage Builds → Bloated Attack Surface
- Stateful Data Inside Container Filesystem: Should Be Externalized
- Layer Cache Invalidation From Poor Instruction Ordering
- Orphaned Volumes / Images: `docker system prune` Hygiene

### 11. Expansion
- Storage Drivers Deep Dive: Overlay2: Default - Recommended vs `devicemapper` vs `btrfs` vs ZFS - Performance / Stability Tradeoffs
- Sandboxed / Alternative Runtimes: gVisor: Userspace Kernel - Stronger Isolation + Kata Containers: Lightweight VM Per Container vs `runc`: Default - Namespace / Cgroup Only
- BuildKit Frontend Architecture: LLB - Low-Level Build Definition - Enables Cache Mounts `--mount=type=cache` + Secret Mounts `--mount=type=secret` Without Layer Leakage
- Docker Content Trust: Image Signing + Cosign / Sigstore: Modern Supply-Chain Signing
- `daemon.json` Tuning: Log Driver | Storage Driver | Default `ulimit` Settings + Registry Mirrors
- Live Restore: Containers Keep Running During Daemon Restart / Upgrade
- Checkpoint / Restore Via CRIU: Experimental - Pause / Resume Container State
- Docker-In-Docker: DinD Risks vs Mounting Host Docker Socket: `-v /var/run/docker.sock` - Privilege Escalation Implications Of Both Approaches
- Rootless Docker Mode Deep Dive: User Namespace Remapping + Network / Storage Limitations

---

## Kubernetes

### 1. Control Plane Architecture
- API Server: Stateless REST Front-End - Authentication / Authorization / Admission Entry Point
- Etcd: Distributed Key-Value Store - Raft Consensus - Single Source Of Truth For Cluster State
- Scheduler: Binds Unscheduled Pods To Nodes Based On Constraints / Scoring
- Controller Manager: Runs Core Reconciliation Control Loops - Node | Replication | Endpoint ...
- Cloud Controller Manager: Cloud-Provider-Specific Integrations
- Reconciliation Loop Pattern: Observe → Diff Desired vs Actual → Act: Level-Triggered

### 2. Node Components
- Kubelet: Node Agent - Pod Lifecycle - Talks To Container Runtime Via CRI
- Kube-Proxy: Implements Service Abstraction - `iptables` Mode vs IPVS Mode vs eBPF / Cilium Replacement
- Container Runtime Interface: CRI - `containerd` / CRI-O Pluggable Runtimes
- CNI: Container Network Interface - Pluggable Pod Networking: Calico | Flannel | Cilium
- eBPF-Native Networking: Cilium - Replaces `kube-proxy` Entirely Via eBPF: No `iptables` Chain Overhead | Hubble: eBPF-Based Network Observability: Service Map + Flow Visibility Without Sidecars - Represents The Direction Kubernetes Networking Is Moving
- CSI: Container Storage Interface - Pluggable Storage Provisioning

### 3. Core Workload Objects
- Pod: Shared Network / IPC Namespace | Co-Located Containers + Pause / Infrastructure Container Mechanics
- ReplicaSet: Desired Replica Count Enforcement
- Deployment: Manages ReplicaSets - Rolling Update / Rollback History + Revision Tracking
- StatefulSet: Stable Network Identity + Ordered Deploy / Scale + Persistent Per-Replica Storage - For Stateful Applications / Databases
- DaemonSet: Exactly One Pod Per Node - Node-Level Agents
- Job / CronJob: Batch + Scheduled Workloads - Completion / Backoff Semantics

### 4. Networking Deep Dive
- Service Types: ClusterIP | NodePort | LoadBalancer | ExternalName + Headless Services: Direct Pod DNS - Used By StatefulSets
- Endpoints / EndpointSlices: Service → Pod IP Mapping - Updated By Controller
- Ingress + Ingress Controller: L7 Routing - TLS Termination - Being Superseded By Gateway API
- Gateway API: GatewayClass / Gateway / HTTPRoute / TLSRoute / TCPRoute - GA Since Kubernetes 1.28 | Role-Oriented: Infrastructure Admin Owns GatewayClass - Cluster Operator Owns Gateway - Application Developer Owns Route: Better Multi-Tenancy Than Ingress | Supports Cross-Namespace Routing + Traffic Weighting Natively
- Network Policies: Pod-Level Micro-Segmentation | Default-Deny Patterns + Ingress / Egress Rules
- Service Mesh Concepts: Sidecar Proxy Pattern | mTLS + Traffic Shaping - Istio / Linkerd As Evolution Beyond Kube-Proxy
- DNS: CoreDNS - Cluster Service Discovery

### 5. Storage Deep Dive
- PersistentVolume: PV / PersistentVolumeClaim: PVC Decoupling - Binding Lifecycle
- StorageClass + Dynamic Provisioning: CSI Driver-Backed
- Access Modes: RWO | ROX | RWX | RWOP
- Reclaim Policies: Retain / Delete / Recycle
- Volume Snapshots: CSI Snapshot API
- StatefulSet `volumeClaimTemplates`: Per-Replica Persistent Identity

### 6. Configuration + Secrets
- ConfigMap: Non-Sensitive Config Injection - Environment Variable vs Volume Mount - Live-Update Caveats
- Secret: Base64-Encoded - NOT Encrypted By Default - Needs Encryption-At-Rest Config + External Secret Managers Like Vault / External Secrets Operator
- Immutable ConfigMaps / Secrets: Performance + Safety Optimization

### 7. Scheduling Deep Dive
- Resource Requests: Guaranteed Minimum vs Limits: Hard Ceiling - Scheduler Uses Requests For Bin-Packing
- QoS Classes: Guaranteed: Req=Limit For All Resources | Burstable + BestEffort - Drives OOM / Eviction Priority
- Node Affinity / Anti-Affinity + Pod Affinity / Anti-Affinity: Topology-Aware Scheduling
- Taints + Tolerations: Node Repels Pods Unless Tolerated
- Topology Spread Constraints: Even Distribution Across Zones / Nodes
- Scheduler Extensibility - Scheduling Framework Plugins + Custom Schedulers
- Priority + Preemption: PriorityClass - Higher Priority Pods Evict Lower Priority

### 8. Autoscaling
- Horizontal Pod Autoscaler: HPA - Metrics-Server / Custom Metrics / External Metrics Driven
- Vertical Pod Autoscaler: VPA - Auto-Adjusts Requests / Limits - Conflicts With HPA On Same Resource If Misconfigured
- Cluster Autoscaler - Node Pool Scaling Based On Pending Unschedulable Pods
- KEDA: Event-Driven Autoscaling - Scale On Queue Depth + Custom Event Sources

### 9. Deployment Strategies
- Rolling Update: `maxSurge` / `maxUnavailable` Tuning
- Recreate: Downtime - Full Replace
- Blue-Green: Full Environment Switch - Instant Rollback
- Canary: Progressive Traffic Shift - Via Service Mesh / Argo Rollouts
- Feature Flags vs Deployment Strategy: Decoupling Release From Deploy

### 10. Health | Reliability + Disruption
- Liveness / Readiness / Startup Probes - Distinct Purposes + Failure Consequences
- PodDisruptionBudget: Protects Availability During Voluntary Disruptions - Node Drain + Upgrades
- Graceful Termination: SIGTERM → `terminationGracePeriodSeconds` → SIGKILL + `preStop` Hooks
- Node Pressure Eviction: Memory / Disk Pressure - Kubelet Eviction Manager - QoS-Based Ordering

### 11. Extensibility + Custom Resources
- Custom Resource Definitions: CRDs - Extend The API
- Operators: CRD + Custom Controller - Encode Operational Knowledge
- Admission Controllers: Mutating vs Validating Webhooks - Request Interception Pipeline
- Dynamic Admission Control Ordering: Mutating Before Validating

### 12. Security + Multi-Tenancy
- RBAC: Role / ClusterRole + RoleBinding / ClusterRoleBinding - Least-Privilege Design
- ServiceAccounts: Pod Identity | Token Projection + Workload Identity Federation With Cloud IAM
- Pod Security Standards / Admission: Restricted / Baseline / Privileged Profiles - Replaced PodSecurityPolicy
- Namespaces As Tenancy Boundary - ResourceQuota + LimitRange Per Namespace
- Network Policies For Tenant Isolation
- Secrets Encryption At Rest: EncryptionConfiguration + Audit Logging

### 13. Helm + Packaging
- Charts: Templated Manifests | `values.yaml` Overrides + Releases / Rollback
- Helm Hooks: Pre-Install / Post-Upgrade Lifecycle Events
- Kustomize: Overlay-Based Config Management - Alternative / Complement To Helm - Patch-Based

### 14. GitOps
- GitOps Model: Git As Single Source Of Truth For Cluster State - Desired State In Repository | Controller Continuously Reconciles Actual State Toward It
- ArgoCD: Application CRD + App-Of-Apps Pattern | Sync Waves + Sync Hooks | Health Assessment + Drift Detection + Auto-Sync
- Flux V2: Kustomization / HelmRelease CRDs + Source Controller: Git / Helm / OCI Source Tracking | Image Reflector + Automation: Automated Image Tag Update Pull Requests
- Push vs Pull Model: Traditional CI Pushes To Cluster: Credentials In CI - GitOps Pull: Controller Has Cluster-Internal Credentials - Smaller Attack Surface
- Multi-Tenant GitOps: Separate Repositories Per Team + ArgoCD Projects / AppProjects RBAC
- Drift Detection + Alerting: Cluster State Diverges From Git: Alert Before Auto-Sync / Manual Review

### 15. Observability In Kubernetes Context
- Metrics-Server: Resource Metrics For HPA + Prometheus Operator: ServiceMonitor CRDs
- Liveness / Readiness Feeding Into Monitoring + Kube-State-Metrics
- Distributed Tracing Integration: OpenTelemetry: OTEL Collector As Sidecar / DaemonSet - Collect / Process / Export Traces + Metrics + Logs

### 16. Operational Failure Modes
- Etcd Quorum Loss: Cluster-Wide Outage Risk - Odd-Numbered Member Count + Backup / Restore Strategy
- No Resource Requests / Limits → Noisy Neighbor + Unpredictable OOMKill
- Missing Readiness Probes → Traffic Sent To Unready Pods
- StatefulSet Misuse For Stateless Workloads
- Split-Brain / Control Plane HA Design: Multiple API Servers Behind Load Balancer + Etcd Cluster Sizing
- Cascading Failures From Missing PodDisruptionBudgets During Node Maintenance

### 17. Expansion
- Etcd Operational Depth: Compaction + Defragmentation | WAL / Snapshot Internals | Quorum Sizing Math: 2f+1 + Backup / Restore Procedures
- API Priority + Fairness: APF - Protects API Server From Overload By Prioritizing / Queuing Requests
- API Aggregation Layer: Extension API Servers Alongside Core API Server
- Full Admission Chain Ordering: Authentication → Authorization → Mutating Admission: In Registration Order → Object Schema Validation → Validating Admission → Etcd Persist
- Client-Go Controller Pattern: Informers / Listers: Local Cache Via Watch | Workqueue: Rate-Limited Reconciliation Triggers + Reconcile Loop - Foundational To Writing Custom Operators
- Leader Election Pattern: Controllers Use Lease Objects To Ensure Single Active Instance In HA Controller Deployments
- Finalizers + Owner References - Garbage Collection Graph - Blocking Deletion Until Cleanup Logic Completes
- CRD Versioning + Conversion Webhooks: Multi-Version API Evolution For Custom Resources
- Kubelet Cgroup Driver Mismatch: `cgroupfs` vs `systemd` - Source Of Node Instability If Kubelet / Container Runtime Disagree
- Ephemeral Containers: `kubectl debug` - Attach Debug Tooling To Running Pod Without Restart
- Multi-Container Pod Design Patterns: Sidecar | Ambassador | Adapter + Init Containers: Sequential Pre-Application Setup
- Service Mesh Control Plane Depth: Istio: Istiod / Pilot Config Distribution | Envoy Sidecar xDS Protocol | mTLS Via SPIFFE / SPIRE Identity + Automatic Certificate Rotation
- Multi-Cluster Management: Cluster API For Cluster Lifecycle + Karmada / KubeFed For Multi-Cluster Workload Federation
- NetworkPolicy Implementation Performance: `iptables` O(N) Rule Evaluation vs eBPF-Based Cilium - Scalability At High Policy / Pod Counts
- Vertical Pod Autoscaler Modes: Off / Initial / Auto / Recreate + Conflict Avoidance With HPA On The Same Metric

---

## Terraform

### 1. Core Model
- Infrastructure As Code - Declarative - Desired-State Driven: HCL - HashiCorp Configuration Language
- Providers: Plugins Translating HCL Resources → Cloud / API Calls + Provider Plugin Protocol: gRPC-Based
- Resources: Managed Infrastructure Objects vs Data Sources: Read-Only Lookups Of Existing Infrastructure

### 2. Core Workflow
- `init`: Backend Init | Provider / Module Download + Dependency Lock File `.terraform.lock.hcl`
- `validate` + `fmt`
- `plan`: Refresh State → Diff Desired vs Real → Execution Plan: Dry-Run
- `apply`: Execute Plan + Update State
- `destroy`: Teardown In Dependency-Reverse Order

### 3. State Management Deep Dive
- State File: `.tfstate` - JSON Mapping Of Resource Addresses → Real Infrastructure IDs / Attributes
- Why State Matters: Performance: Avoid Re-Querying Everything - Mapping Config ↔ Real-World + Metadata For Dependency Resolution
- Remote Backends: S3 + DynamoDB Lock | Terraform Cloud / Enterprise | Azure Blob | GCS - Team Collaboration
- State Locking Mechanism: Prevents Concurrent Apply Corruption
- State Drift: Manual Out-Of-Band Changes + Detection Via `plan`
- `terraform import`: Adopt Existing Unmanaged Infrastructure
- `terraform state mv` / `rm` / `list`: Surgical State Manipulation
- State File Sensitive Data Exposure Risk: Secrets Stored In Plaintext State - Mitigations: Encryption At Rest On Backend - `sensitive = true` Marking Limits CLI Output Only - Not Storage

### 4. Dependency Graph + Execution
- Implicit Dependency Graph: DAG Built From Resource Attribute References
- Explicit `depends_on`: For Non-Attribute-Inferable Dependencies
- Parallelism: `-parallelism=n` - Concurrent Resource Operations Within DAG Constraints
- Resource Lifecycle Meta-Arguments: `create_before_destroy` | `prevent_destroy` + `ignore_changes`

### 5. Modules
- Root Module vs Child Modules - Encapsulation + Reuse
- Input Variables: With Type Constraints | Validation Blocks + Defaults + Output Values
- Module Versioning + Registry: Public / Private Registries
- Module Composition Patterns: Avoid Excessive Nesting + Single-Responsibility Modules

### 6. Language Features
- `for_each` vs `count` - Resource Addressing Implications: Map / Set-Based Stable Keys vs Index-Based - Count Reordering Risk
- Dynamic Blocks: Generate Nested Blocks Programmatically
- Conditional Expressions | `try()` / `can()` Functions
- Locals: Computed / Derived Reusable Values

### 7. Workspaces + Environment Strategy
- Terraform Workspaces: Multiple State Instances From Same Config - Pros / Cons vs Directory-Per-Environment Pattern
- Environment Isolation Strategies: Separate State Files / Backends Per Environment To Reduce Blast Radius

### 8. Policy + Governance
- Sentinel / Open Policy Agent: OPA - Policy-As-Code Gating On Plan / Apply
- `terraform plan` Output Review In CI/CD: Pull-Request-Based Plan Preview Workflows
- Cost Estimation Integration: Infracost

### 9. Testing + CI/CD Integration
- `terraform validate` | `tflint` + `checkov` / `tfsec`: Static Security Scanning
- Terratest: Go-Based Integration Testing Of Provisioned Infrastructure
- GitOps-Style Pipelines: Plan On Pull Request - Apply On Merge With Approval Gates

### 10. Advanced / Operational Concerns
- Provider Version Pinning + Upgrade Strategy
- Large Monolithic State vs Decomposed State Per Component: Blast Radius - Plan / Apply Speed Tradeoffs
- Secrets Handling: Avoid Hardcoding - Integrate With Vault / SSM / Secrets Manager Via Data Sources
- `-target` Flag Risks: Partial Apply Can Cause State / Dependency Inconsistency
- Refactoring Safely: Moved Blocks In Newer Terraform Versions To Preserve State Across Renames

### 11. Expansion
- Provider Development: SDKv2 vs Modern Provider Framework - Plugin Protocol Internals For Building Custom Providers
- CDK For Terraform: CDKTF - Define Infrastructure In General-Purpose Languages: TypeScript / Python - Synthesizes To HCL / JSON
- Terraform Cloud / Enterprise Run Workflow: VCS-Driven Runs | Remote Execution + Sentinel Policy Gating In Pipeline
- `count` / `for_each` Resource-Addressing Gotchas During Refactors: Index Shifting Causing Destroy / Recreate Storms
- `terraform graph`: Visualize DAG For Debugging Complex Dependency Chains
- `null_resource` + `local-exec` / `remote-exec` Provisioners - Recognized Anti-Pattern: Imperative Escape Hatch Breaks Declarative Model - Use Sparingly
- Backend Migration Process: Safely Moving State Between Backend Types Without Loss
- State Encryption Specifics: Backend-Level Encryption At Rest - S3 SSE | GCS CMEK
- Automated Drift Remediation Strategies: Scheduled Plan Diffing + Alerting / Auto-Apply In Controlled Pipelines

---

## Ansible

### 1. Core Model
- Agentless | SSH / WinRM-Based Push Model Configuration Management vs Chef / Puppet Pull-Agent Model
- Idempotency Principle - Playbook Re-Run Converges To Same State Without Unintended Side Effects
- YAML-Based Declarative Playbooks + Python Under The Hood: Modules Executed Remotely

### 2. Architecture
- Control Node: Orchestrator → Managed Nodes: Targets
- Inventory: Static: INI / YAML vs Dynamic Inventory: Cloud Plugins - AWS / Azure / GCP Auto-Discovery
- Modules - Idempotent Units Of Work: Package | File | Service | Cloud Resource Modules - Module Execution Mechanism: Python Script Pushed + Executed Remotely - Then Cleaned Up
- Connection Plugins: SSH | WinRM | Local | Docker

### 3. Building Blocks
- Playbook → Play: Hosts + Tasks Mapping → Task: Module Invocation
- Roles - Structured Reusable Bundles: Tasks / Handlers / Variables / Templates / Files / Defaults / Meta
- Handlers - Triggered Via `notify` - Run Once At End Of Play: Deduplication Of Multiple Notifies
- Collections - Namespaced Distribution Of Roles / Modules / Plugins: Modern Packaging Unit

### 4. Variables + Templating
- Variable Precedence Hierarchy: Extra Variables Highest → Role Defaults Lowest - Full Order Matters In Production
- `group_vars` / `host_vars` Directory Structure
- Facts: Auto-Gathered Via `setup` Module - `gather_facts: false` Optimization
- Registered Variables: Task Output Capture For Conditional Downstream Logic
- Jinja2 Templating - Filters | Tests | Loops + Macros In Templates: `template` Module Renders `.j2` Templates

### 5. Control Flow
- Conditionals: `when` + Loops: `loop` | `with_*` Legacy Lookups
- Tags: Selective Execution - `--tags` / `--skip-tags`
- Blocks: Grouping + `rescue` / `always` - Structured Error Handling Analog To Try / Catch / Finally
- `delegate_to`: Run Task On Different Host Than Play Target - Load Balancer Draining
- `run_once` + `serial`: Rolling Batch Deployment Across Host Groups

### 6. Execution + Strategy
- Execution Strategies: `linear`: Default - Lockstep Across Hosts vs `free`: Hosts Run Independently vs `host_pinned`: Free But No Cross-Host Interleaving
- Forks: Parallelism Level - `-f`
- Check Mode: `--check` - Dry-Run + Diff Mode: `--diff`
- Ansible Vault - Encryption Of Sensitive Variables / Files + Vault IDs For Multi-Key Management

### 7. Idempotency + Safety Patterns
- Preferring Declarative Modules Over `shell` / `command`: Idempotency By Design
- `changed_when` / `failed_when` Custom Logic For Non-Idempotent Commands
- `ignore_errors` vs Proper Error Handling: Block / Rescue - Pitfalls Of Over-Suppression

### 8. Ecosystem + Scale
- Ansible Galaxy: Community Role / Collection Sharing
- Ansible Tower / AWX: Enterprise RBAC | Scheduling | Workflow Visualization + API-Driven Execution
- Molecule: Role Testing Framework
- Integration With Dynamic Inventory + Cloud Provisioning Workflows: Terraform Provisions - Ansible Configures

### 9. Common Production Pitfalls
- Fact-Gathering Overhead At Scale: Thousands Of Hosts
- Secrets In Plaintext vs Vault Discipline
- Non-Idempotent Raw Shell Tasks Breaking Convergence Guarantees
- Inventory Sprawl / Lack Of Dynamic Inventory In Cloud-Elastic Environments

### 10. Expansion
- Mitogen For Ansible: Performance Optimization - Persistent Interpreter - Avoids Repeated SSH / Python Startup Overhead Per Task
- `ansible-pull`: Inverted Pull-Model Variant - Nodes Pull + Apply Their Own Config - Useful For Large Fleets / Edge Devices
- Custom Module + Plugin Development: Python Module Contract: Stdin JSON In - JSON Out + Callback Plugins For Custom Output / Integration
- Custom Jinja2 Filters / Tests For Advanced Templating Logic
- Execution Environments: Containerized - Reproducible Ansible Runtime - Replaces Control-Node Drift
- `requirements.yml`: Collections / Roles Dependency Pinning For Reproducible Automation
- Async Tasks + Polling: `async` / `poll` For Long-Running Operations Without Blocking The Play
- Fault Tolerance Controls: `any_errors_fatal` | `max_fail_percentage` + `force_handlers`
- `ansible-lint` / Molecule Testing Framework For Role Correctness + Style Enforcement In CI

---

## Jenkins

### 1. Core Model
- Automation Server For CI/CD Orchestration
- Controller: Master - Agent: Node Distributed Architecture
- Java-Based | Plugin-Driven Extensibility: Large Ecosystem + Maintenance Governance

### 2. Pipeline As Code
- Declarative Pipeline: `pipeline {}` Structured Syntax - Stages / Steps / Post vs Scripted Pipeline: Groovy DSL - Full Programmatic Control
- Jenkinsfile - Versioned Alongside Source: Pipeline-As-Code Principle
- Multibranch Pipeline - Auto-Discovers Branches / Pull Requests - Per-Branch Jenkinsfile Execution
- Shared Libraries - Reusable Groovy Pipeline Code Across Multiple Jenkinsfiles / Repositories

### 3. Pipeline Structure Deep Dive
- Stages → Steps | Parallel Stage Execution + Matrix Builds: Multi-Axis Parameter Combinations
- Post Section Semantics: Always / Success / Failure / Unstable / Changed / Aborted
- Environment Blocks | Parameters: Build-Time Input + Triggers: Cron / `pollSCM` / Upstream
- `input` Step: Manual Approval Gates In Pipeline
- Pipeline Restart From Stage / Resume Capability

### 4. Agents + Execution Model
- Static Long-Lived Agents vs Dynamic / Ephemeral Agents: Docker / Kubernetes Plugin - Spin Up Pod Per Build - Tear Down After
- Labels - Route Jobs To Agents With Specific Capabilities
- Executors - Concurrent Build Slots Per Agent
- Agent Connection Methods: SSH | JNLP / Inbound

### 5. Triggers + Integration
- Webhook-Based Triggers: GitHub / GitLab Push / Pull Request Events vs SCM Polling: Less Efficient
- Upstream / Downstream Job Chaining: `build` Step
- Scheduled Builds: Cron Syntax

### 6. Credentials + Security
- Credential Types: Username / Password | SSH Key | Secret Text | Certificate + Secret File
- Credential Scoping: Global vs Folder-Level vs Job-Level
- Integration With External Secret Stores: Vault | AWS Secrets Manager Plugins
- Script Security Sandbox: Groovy Sandbox For Untrusted Pipeline Scripts + Approval Workflow For Admin-Approved Methods
- Role-Based Access Control: Matrix / Role Strategy Plugin

### 7. Scaling + HA
- Distributed Builds Across Agent Pool: Horizontal Scaling
- Jenkins Configuration As Code: JCasC - Declarative Controller Config - Avoids Manual UI Drift
- Controller HA Strategies: Active / Passive Failover
- Ephemeral Kubernetes Agents As Elastic Scaling Pattern

### 8. Artifacts + Build Management
- Artifact Archiving + Fingerprinting: Track Artifact Usage Across Jobs
- Build Retention Policies: Discard Old Builds
- Workspace Cleanup Strategies: Disk Bloat Prevention

### 9. Common Production Pitfalls
- Plugin Version Conflicts / Unmaintained Plugin Risk
- Secrets Embedded Directly In Jenkinsfile Instead Of Credentials Store
- Single Point Of Failure At Controller: `numExecutors = 0` Mitigation
- Not Leveraging Shared Libraries → Duplicated Pipeline Logic Across Repositories
- Groovy Sandbox Escapes / Trusting Unreviewed Pipeline Scripts

### 10. Expansion
- Configuration As Code: JCasC Plugin Depth - Full Controller Config: Security Realm | Credentials References | Clouds As Versioned YAML
- Shared Library Versioning: `@Library('name@version')` - Pinning Strategies For Stable vs Bleeding-Edge Pipeline Logic
- Kubernetes Plugin Pod Templates: Per-Stage Container Definitions + Dynamic Agent Provisioning Specifics
- Build Discarder Strategies: Log / Artifact Retention Tuning To Control Disk Growth
- Reverse Proxy Configuration Nuances: Jenkins Behind NGINX / ALB - Required Headers + Context Path Handling
- CSRF Protection: Crumb Issuer - Common Automation / API Integration Pitfall
- Distributed Build Load Balancing Algorithm: Label-Matching + Least-Loaded Agent Selection
- Jenkins REST API + CLI For Programmatic Job / Pipeline Management: Automation Of Jenkins Itself
- Pipeline Replay + Declarative Validation: Fast Iteration Without Full Commit / Push Cycle
- Migration / Modernization Context: Jenkins X / Evaluating Jenkins vs Newer Cloud-Native CI Like Argo Workflows / Tekton / GitHub Actions Runners

---

## Prometheus

### 1. Core Model
- Pull-Based Metrics Monitoring System + Time-Series Database: TSDB
- Scrape Model - Prometheus Actively Pulls `/metrics` HTTP Endpoints On Interval vs Push-Based Systems
- Multi-Dimensional Data Model: Metric Name + Label Key-Value Pairs Uniquely Identify A Time Series

### 2. Metric Types
- Counter: Monotonically Increasing - Use `rate()` / `increase()` - Never Read Raw Value Directly For Dashboards
- Gauge: Arbitrary Up / Down Value
- Histogram: Bucketed Observations + `_sum` / `_count` - Server-Side Aggregable - `histogram_quantile()` For Percentiles
- Summary: Client-Side Calculated Quantiles - NOT Aggregable Across Instances - Key Tradeoff vs Histogram

### 3. Architecture
- Prometheus Server: Scrape + Local TSDB Storage + PromQL Query Engine + Rule Evaluation
- Exporters: Translate Third-Party Metrics To Prometheus Format - `node_exporter` | `blackbox_exporter` | `mysqld_exporter`
- Pushgateway: Bridge For Short-Lived / Batch Jobs That Cannot Be Scraped Directly - Narrow Use Case
- Service Discovery: Kubernetes SD | Consul SD | EC2 SD + File-Based SD - Dynamic Target Discovery + Relabeling
- OpenTelemetry: OTEL - Vendor-Neutral Observability Standard: Unified API / SDK For Traces + Metrics + Logs | OTEL Collector: Receive / Process / Export Pipeline - Prometheus Remote Write Is A Common OTEL Export Target: OTEL + Prometheus Are Converging Not Competing

### 4. PromQL Deep Dive
- Instant Vector vs Range Vector Selectors
- `rate()` vs `irate()` - Smoothed Average vs Instantaneous: When To Use Which
- Aggregation Operators: `sum` | `avg` | `min` | `max` | `count` With `by` / `without` Clauses
- `histogram_quantile()` For Latency Percentile Estimation From Histogram Buckets
- Recording Rules: Precompute Expensive Queries vs Alerting Rules

### 5. Alerting
- Alerting Rules: PromQL Boolean Condition → Fires ALERT
- Alertmanager - Deduplication | Grouping | Routing Trees | Silencing + Inhibition Rules
- Notification Integrations: Slack | PagerDuty | Opsgenie + Webhook
- Alert Fatigue Prevention: Proper Grouping / Thresholds + Symptom-Based vs Cause-Based Alerting

### 6. Storage Internals
- Local TSDB: WAL: Write-Ahead Log For Crash Recovery + Immutable Time-Series Blocks + Compaction
- Chunk Encoding: Gorilla Compression - Delta-Of-Delta Timestamps + XOR Float Compression
- Retention Configuration: Time + Size-Based
- Remote Write / Read Protocol - Long-Term Storage Integration

### 7. Scaling Patterns
- Federation: Hierarchical Scraping Between Prometheus Servers - Limited Scale Ceiling
- Thanos / Cortex / Mimir - Horizontally Scalable | Globally Queryable | Long-Term Storage + HA Via Deduplication
- Sharding Strategy: By Job / Target Across Multiple Prometheus Instances

### 8. Production Architecture Concerns
- High Cardinality Labels - The #1 Operational Risk: Unique Label Combos Explode Memory / Disk
- Scrape Interval vs Staleness / Resolution Tradeoff
- No Remote-Write = Data Loss Risk On Server Restart / Disk Failure
- Symptom vs Cause Based Alerting Philosophy: Alert On User-Facing SLO Burn - Not Every Internal Metric
- Multi-Tenancy Considerations At Scale: Cortex / Mimir / Thanos Tenant Isolation

### 9. Expansion
- Native Histograms: Sparse | High-Resolution Histograms - Reduces Cardinality Cost vs Classic Bucketed Histograms
- Exemplars - Link Metric Spikes Directly To Trace IDs: Metrics ↔ Tracing Correlation
- OpenMetrics Format: Interoperability Standard Prometheus Format Is Based On / Converging With
- Remote Write Protocol V2: Efficiency + Metadata Improvements Over V1
- PromQL Subqueries: Range Query Over A Range Query Result - Advanced Aggregation Patterns
- Staleness Handling: How Prometheus Marks / Query-Handles Series That Stop Being Scraped
- Out-Of-Order: OOO Sample Ingestion Support: Handling Late-Arriving Data
- TSDB Block Compaction Levels: 2h Raw Blocks → Progressively Larger Compacted Blocks + Retention Interaction
- Prometheus Operator CRDs: ServiceMonitor | PodMonitor | PrometheusRule - Kubernetes-Native Declarative Scrape / Alert Config
- Cost Control Via Relabeling: Drop High-Cardinality / Unneeded Labels / Metrics At Scrape Time Before Ingestion
- Federation Performance Ceiling + When To Graduate To Thanos / Cortex / Mimir Instead

---

## Grafana

### 1. Core Model
- Visualization / Dashboarding Layer - Data Source Agnostic - Not A Storage System Itself
- Query → Transform → Visualize Pipeline Per Panel

### 2. Data Sources
- Native Support: Prometheus | Loki | Elasticsearch | InfluxDB | MySQL / PostgreSQL | CloudWatch + Tempo: Traces
- Data Source Proxying: Grafana Backend Proxies Queries - Credentials Never Exposed To Browser
- Mixed Data Source Panels: Combine Multiple Sources In One Visualization

### 3. Dashboards
- Panel Types: Time Series / Graph | Table | Gauge | Stat | Heatmap | Bar Gauge | Logs + Node Graph
- Templating / Variables - Dropdown Filters: `$Environment` | `$Namespace` | `$Service` | `$Instance` - Query-Based Variable Population + Chained Variables
- Annotations - Mark Events On Timeline: Deploys | Incidents - Manual / Query-Driven
- Dashboard JSON Model - Export / Import + Version Control: Dashboards-As-Code
- Repeat Panels / Rows: Dynamic Panel Generation Per Variable Value

### 4. Alerting: Unified Alerting
- Data-Source-Agnostic Alert Rule Evaluation
- Contact Points + Notification Policies: Routing Tree Similar To Alertmanager Concept
- Multi-Dimensional Alerting: Alerts Per Label Combination
- Alert State Management: Pending / Firing / Resolved + Silence

### 5. Access Control + Organization
- Organizations → Teams → RBAC: Viewer / Editor / Admin - Fine-Grained Permissions In Enterprise
- Folders: Dashboard Grouping + Permission Inheritance
- Public Dashboards / Snapshot Sharing: Data Exposure Risk Considerations
- Playlists: Auto-Rotating Dashboard Display - NOC / TV Use Case

### 6. Provisioning + GitOps
- Dashboard Provisioning Via Config Files / API: Avoid Manual UI-Only Dashboards - Drift Risk
- Data Source Provisioning As Code
- Grafana Operator: Kubernetes-Native Provisioning

### 7. Extensibility
- Panel Plugins | Data Source Plugins + App Plugins
- Grafana Scenes / Grafana Explore: Ad-Hoc Query Exploration

### 8. Production Architecture Concerns
- Dashboard Sprawl Without Provisioning / Version Control → Configuration Drift
- Overloaded Dashboards: Excessive Concurrent Queries → Slow Load - Backend Data Source Strain
- Not Leveraging Variables → Duplicated Per-Environment Dashboards
- Correlating Metrics / Logs / Traces: Exemplars + Trace-To-Logs Linking For Full Observability

### 9. Expansion
- Unified Alerting V2 Architecture Internals: Scheduler + State Manager Evaluating Rules Independent Of Any Single Data Source
- Grafana Loki + LogQL: Log Aggregation Designed To Pair With Prometheus Philosophy - Index Only Labels - Not Full Text
- Grafana Tempo: Distributed Tracing Backend + Service Graph: Auto-Derived Service Dependency Maps From Trace Data
- Grafana ML: Outlier / Anomaly Detection On Top Of Existing Metrics
- Dashboards / Data Sources As Code Via Terraform Grafana Provider / Grafonnet / Jsonnet: Full GitOps For Observability Config
- SSO / OAuth / LDAP Integration For Enterprise Identity Federation
- Templated Variable Performance At Scale: Expensive Variable Queries Can Bottleneck Dashboard Load - Caching / Scoping Strategies
- Grafana Enterprise Reporting: Scheduled PDF / Image Export + White-Labeling For External Stakeholders

---

## ELK: Elasticsearch | Logstash | Kibana + Beats

### 1. Elasticsearch Core Model
- Distributed Document Store + Search Engine Built On Apache Lucene
- JSON Documents + Inverted Index: Term → Document Mapping Enabling Fast Full-Text Search
- Index → Shard: Primary + Replica Shards → Segment: Immutable Lucene-Level Structure - Periodically Merged
- Licensing Context: Elastic Changed To SSPL License In 2021 - AWS Forked Elasticsearch + Kibana Into OpenSearch / OpenSearch Dashboards: Apache 2.0 Licensed - When Choosing ELK In Enterprise Contexts: Evaluate Elastic vs OpenSearch Based On License / Support / Feature Tradeoffs

### 2. Cluster Architecture
- Node Roles: Master-Eligible | Data | Ingest | Coordinating-Only + Machine Learning
- Master Election: Quorum-Based - Avoids Split-Brain - `cluster.initial_master_nodes`
- Shard Allocation Algorithm: Balances Shards Across Nodes Based On Disk / Shard Count Heuristics
- Cluster State Propagation

### 3. Indexing + Mapping
- Mapping: Explicit Schema: Field Types | Analyzers vs Dynamic Mapping: Auto-Inferred - Risk Of Mapping Explosion
- Analyzers / Tokenizers / Filters: Text Processing Pipeline: Standard Analyzer | Custom Analyzers | Stemming + Stop Words
- Doc Values: Columnar Storage For Aggregations / Sorting vs Fielddata: In-Memory - Deprecated For Text Fields - Memory Risk
- Refresh Interval: Near-Real-Time Search - Tradeoff Of Refresh Frequency vs Indexing Throughput

### 4. Querying
- Query DSL: JSON-Based - Match | Term | Bool: Must / Should / MustNot / Filter + Range
- Filter Context: Cacheable - No Scoring vs Query Context: Relevance Scoring Via TF-IDF / BM25
- Aggregations: Metric: Avg / Sum / Percentiles | Bucket: Terms / Histogram / DateHistogram + Pipeline: Derivative / MovingAvg
- Relevance Scoring Internals: BM25 Default In Modern Elasticsearch

### 5. Reliability + Lifecycle
- Replication: Fault Tolerance + Read Scaling - Replica Shards
- ILM: Index Lifecycle Management - Hot / Warm / Cold / Frozen / Delete Phase Automation
- Snapshot + Restore: Incremental Backups To S3 / GCS / Shared Filesystem Repository
- Circuit Breakers: Prevent OOM From Oversized Queries / Aggregations
- Cross-Cluster Replication + Cross-Cluster Search: Multi-Datacenter / Disaster Recovery Patterns

### 6. Logstash
- Pipeline Model: Input → Filter → Output - Multiple Pipelines Support
- Input Plugins: Beats | Kafka | JDBC | HTTP | Syslog + File
- Filter Plugins: Grok: Regex-Based Parsing | Mutate | Date | JSON | GeoIP + Dissect: Faster Alternative To Grok For Fixed-Format Lines
- Output Plugins: Elasticsearch | Kafka | S3 + Stdout
- Persistent Queues: Disk-Backed Buffering For Backpressure Resilience
- Performance Tuning: Pipeline Workers + Batch Size / Delay

### 7. Beats: Lightweight Shippers
- Filebeat: Log Shipping - Harvester / Registry Mechanism For Tracking File Offsets | Metricbeat | Packetbeat | Winlogbeat + Auditbeat
- Beats → Logstash: Heavy Transform Needed vs Beats → Elasticsearch Directly: Lightweight Ingest Pipelines Using Elasticsearch Ingest Nodes
- Modules: Pre-Built Parsing Configs For Common Log Sources

### 8. Kibana
- Discover: Ad-Hoc Search / Filter On Raw Documents
- Visualizations + Dashboards | Lens: Drag-And-Drop Visualization Builder + Canvas: Pixel-Perfect Reporting
- Index Patterns / Data Views: Kibana Mapping To Underlying Elasticsearch Indices
- Dev Tools Console: Direct Query DSL / API Access - Critical For Debugging
- Kibana-Native Alerting + Machine Learning Features: Anomaly Detection

### 9. Architecture Patterns
- Beats → Logstash → Elasticsearch → Kibana: Full Pipeline - Complex Transforms
- Beats → Elasticsearch: Ingest Pipeline → Kibana: Lightweight - Less Operational Overhead
- Hot-Warm-Cold-Frozen Tiered Architecture: Cost / Performance Optimization By Data Age

### 10. Operational Concerns + Failure Modes
- Over-Sharding: Excess Small Shards → Cluster Metadata / Heap Overhead - Shard Count Is Not Free
- Missing ILM → Unbounded Index Growth → Disk Pressure → Cluster Red Status
- Poor Mapping Design / Mapping Explosion From Unrestricted Dynamic Mapping
- JVM Heap Sizing: ≤50% Of RAM - ≤ ~32 GB To Preserve Compressed OOPs
- Grok Performance: Catastrophic Backtracking In Poorly Written Regex Patterns - Logstash Bottleneck
- Split-Brain Avoidance In Master Election
- Query-Time vs Index-Time Tradeoffs: Denormalization | Nested vs Parent-Child vs Flattened Field Design

### 11. Expansion
- Data Streams: Modern Time-Series Index Abstraction Replacing Manual Daily-Index + Alias Rotation Pattern
- Composable Index Templates: Component Templates + Index Templates - Modular Mapping / Settings Reuse
- Rollover API: Size / Age / Doc-Count Triggered Index Rotation - Pairs With ILM
- Force Merge: Manually Collapse Segments In Read-Only / Cold Indices - Reduces Overhead - Immutable Data Only
- Index Sorting: Pre-Sort Documents At Index Time For Faster Range / Sort Queries - Tradeoff vs Indexing Throughput
- Native Security Features: RBAC | Field-Level + Document-Level Security: Row / Column-Level Access Control Within An Index
- Elastic Agent + Fleet: Successor To Standalone Beats - Centralized Management Of Data Collection Agents
- APM: Application Performance Monitoring Integration - Distributed Tracing Ingested Into The Same Elastic Stack
- EQL: Event Query Language - Sequence-Based Querying - Primarily For Security / Threat-Hunting Use Cases
- Cross-Cluster Search: Federated Query Across Clusters - Read-Only vs Cross-Cluster Replication: Actual Data Copy For Disaster Recovery / Locality
- Dedicated Node Role Sizing Strategy: Separate Master / Data / Ingest / Coordinating Nodes At Scale - Avoids Resource Contention Between Cluster-Management + Query / Indexing Workloads

---

## Cross-Cutting Systems Themes

### System Design Integration
- CI/CD Flow: Jenkins: Build / Test / Deploy → Docker: Package → Kubernetes: Orchestrate → Terraform / Ansible: Provision Infrastructure / Configuration → GitOps: ArgoCD / Flux: Continuous Reconciliation
- Observability Triad: Prometheus: Metrics + ELK / Loki: Logs + Grafana: Unified Visualization + OpenTelemetry: OTEL Collector: Unified Instrumentation + Distributed Tracing: Tempo / Jaeger = Full Observability
- Messaging Layer Decision Matrix: Kafka: Event Streaming / Replay / High-Throughput vs RabbitMQ: Task Queues / Complex Routing / RPC vs Redis: Cache / Lightweight Pub-Sub / Ephemeral Queues
- Edge / Traffic Layer: NGINX: Reverse Proxy / Load Balancer / Ingress Fronting Kubernetes Services / Standalone Applications
- IaC Layering: Terraform: Provision Cloud Infrastructure + Ansible: Configure OS / Application Layer - Complementary - Chained In Pipelines

### Reliability Engineering Concepts
- CAP Theorem Tradeoffs In Each System: Kafka ISR / ACKs | Redis Cluster | Elasticsearch Quorum | RabbitMQ Quorum Queues | Etcd
- Consensus Algorithms Encountered: Raft: Kafka KRaft | Etcd | RabbitMQ Quorum Queues + Gossip Protocols: Redis Cluster | Elasticsearch Discovery Historically
- Idempotency As A Cross-System Design Principle: Kafka Producers | Ansible Modules + API Design Generally
- Backpressure + Flow Control Patterns: RabbitMQ Flow Control | Kafka Consumer Lag + Kubernetes Resource Limits
- Split-Brain Prevention Patterns: Quorum-Based Systems Across The Stack

### Capacity Planning + Cost
- Horizontal vs Vertical Scaling Tradeoffs Per System
- Resource Request / Limit Discipline: Kubernetes Mirrored Conceptually In JVM Heap Sizing: Kafka / Elasticsearch | Redis `maxmemory` + NGINX Worker Tuning

### Security Posture Across The Stack
- Secrets Management Consistency: Vault / Cloud Secret Managers - Jenkins Credentials | Kubernetes Secrets | Terraform State | Ansible Vault + Docker Build Secrets
- Least Privilege: Kubernetes RBAC | Ansible Vault Access | Jenkins Credential Scoping + Elasticsearch Security Features
- Network Segmentation: Kubernetes NetworkPolicy | NGINX WAF + RabbitMQ / Kafka TLS + ACLs

### Disaster Recovery Patterns
- Backup / Restore Strategy Per System: Etcd Snapshots | Elasticsearch Snapshots | Redis RDB / AOF + Terraform State Backups
- Multi-Region / Multi-Datacenter Strategies: Kafka MirrorMaker | RabbitMQ Federation / Shovel | Elasticsearch Cross-Cluster Replication + Kubernetes Multi-Cluster

### Core Systems Synthesis
- Recognizing The Same Distributed-Systems Primitive Wearing Different Clothes Across Tools: Raft: Etcd | Kafka KRaft | RabbitMQ Quorum Queues vs Gossip: Redis Cluster | Historical Elasticsearch Discovery vs External Coordination: Legacy Kafka + ZooKeeper - Knowing Which Consistency Model Underlies A Tool Predicts Its Failure Behavior
- Every Scale-Out System In This List Re-Derives The Same Tradeoffs: Shard / Partition Count vs Metadata Overhead: Kafka Partitions | Elasticsearch Shards | Redis Cluster Slots - Over-Sharding Is A Recurring Anti-Pattern Across All Of Them
- Buffer / Backpressure Design Recurs Everywhere: NGINX Proxy Buffers | RabbitMQ Flow Control | Kafka Producer Batching | Kubernetes Admission Queueing / APF - The General Pattern: Bound The Queue - Reject / Shed Load Explicitly Rather Than Let It Grow Unbounded
- Declarative Reconciliation Is The Dominant Control Pattern Of The Modern Stack: Kubernetes Controllers | Terraform Plan / Apply | GitOps / ArgoCD | Ansible Idempotent Convergence - Understanding One Reconciliation Loop Transfers Directly To Reading Any Other
- Secrets / Identity Is The Connective Tissue Across The Whole List: A Mature Systems Architecture Treats Vault / Cloud KMS / Secrets Manager As The Single Source Of Truth Feeding Jenkins Credentials | Kubernetes Secrets Via External Secrets Operator | Ansible Vault | Terraform Sensitive Variables + Application Configurations Uniformly
- Observability Is Not Three Separate Tools Bolted Together: Metrics: Prometheus / Grafana | Logs: ELK / Loki + Traces: Tempo / Jaeger Are Correlated Via Shared Identifiers: Trace ID As Exemplar - Log Line As Span Event - OpenTelemetry: OTEL Is The Industry-Standard Instrumentation Layer That Ties Them Together: Vendor-Neutral SDK + Collector Pipeline - The Architecture Focus Is Designing Systems That Emit Correlated Telemetry From Day One - Not Retrofitting It
- Cost / Performance Tiering Is A Repeated Pattern: Elasticsearch Hot-Warm-Cold | Kafka Tiered Storage | Kubernetes Node Pools: Spot / On-Demand Mix - Same Underlying Idea Of Matching Resource Cost To Data / Workload Temperature
- Migration / Versioning Discipline Matters As Much As Steady-State Design: Terraform State Migrations | Kubernetes CRD Conversion Webhooks | Elasticsearch Reindex-On-Upgrade | Kafka Message Format Upgrades - The Ability To Evolve A Running System Without Downtime Is What Defines High Operational Maturity