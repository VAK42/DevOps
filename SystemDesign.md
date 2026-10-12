## Fundamentals

### 1. Interview
- **Framework**: Outline Use Cases & Constraints → Create High-Level Design → Design Core Components → Scale The Design
- **1 - Clarify Requirements**: Ask Who Uses It, How They Use It, How Many Users, What Data Volume Is Expected, Expected Read/Write Ratio & Requests Per Second
- **2 - High-Level Design**: Sketch Main Components & Connections, Justify Each Architectural Decision With Trade-Off Reasoning
- **3 - Core Component Design**: Dive Into Details Per Component (Data Models, API Design, Algorithm Choices, Storage Schema)
- **4 - Scale The Design**: Identify & Address Bottlenecks Using Load Balancing, Horizontal Scaling, Caching & Database Sharding
- **Back-Of-The-Envelope Estimation**: Practice Order-Of-Magnitude Calculations For Storage, Bandwidth & QPS Using Powers-Of-2 Table & Latency Numbers

### 2. Key Trade-Off Axes
- **Performance vs Scalability**: A Performance Problem Means Your System Is Slow For A Single User. A Scalability Problem Means It Is Fast For 1 User But Slow Under Heavy Load
- **Latency vs Throughput**: Latency Is Time To Complete A Single Operation. Throughput Is Operations Completed Per Unit Time. Generally Aim For Maximal Throughput With Acceptable Latency
- **Availability vs Consistency**: The CAP Theorem States That In A Distributed System You Can Only Guarantee 2 Of: Consistency, Availability & Partition Tolerance. Since Networks Are Unreliable, Partition Tolerance Is Non-Negotiable, Forcing A Choice Between Consistency & Availability
- **CP Systems**: Choose Consistency & Partition Tolerance When Business Needs Require Atomic Reads & Writes. Waiting For A Partitioned Node May Return A Timeout Error (etcd, ZooKeeper, HBase, ...)
- **AP Systems**: Choose Availability & Partition Tolerance When Business Needs Allow Eventual Consistency Or The System Must Remain Operational During Network Errors (Cassandra, DynamoDB, DNS, ...)

### 3. Consistency Patterns
- **Weak Consistency**: After A Write, Reads May Or May Not See It. Best-Effort Approach Seen In Systems Like Memcached. Ideal For Real-Time VoIP, Video Chat & Multiplayer Games Where Stale Reads Are Acceptable
- **Eventual Consistency**: After A Write, Reads Will Eventually See It (Typically Within Milliseconds). Data Replicated Asynchronously. Seen In DNS & Email. Works Well In Highly Available Systems
- **Strong Consistency**: After A Write, Reads Always See It. Data Replicated Synchronously. Seen In File Systems & RDBMS. Required For Systems Needing ACID Transactions

### 4. Availability Patterns & SLA Numbers
- **Active-Passive Failover**: Heartbeats Sent Between Active & Passive Servers. If Heartbeat Is Interrupted, Passive Takes Over Active IP Address. Only Active Server Handles Traffic. Also Known As Master-Slave Failover
- **Active-Active Failover**: Both Servers Manage Traffic Spreading Load Between Them. DNS Must Know About Both Public IPs. Also Known As Master-Master Failover
- **Availability In Numbers**: 99.9% = 8h 45min Downtime/Year. 99.99% = 52min Downtime/Year. 99.999% = ~5min Downtime/Year
- **Availability In Sequence**: `Availability(Total) = Availability(A) * Availability(B)`. 2 99.9% Components In Sequence = 99.8% Total
- **Availability In Parallel**: `Availability(Total) = 1 - (1 - Availability(A)) * (1 - Availability(B))`. 2 99.9% Components In Parallel = 99.9999% Total

### 5. Latency Reference Numbers
- **L1 Cache Reference**: ~0.5 ns
- **L2 Cache Reference**: ~7 ns
- **Main Memory Reference**: ~100 ns
- **SSD Random Read (4 KB)**: ~150 us
- **Sequential Memory Read (1 MB)**: ~250 us
- **Same-Datacenter Round Trip**: ~500 us
- **Sequential SSD Read (1 MB)**: ~1 ms
- **HDD Seek**: ~10 ms
- **Cross-Continental Network Round Trip**: ~150 ms
- **Implications**: Memory Is ~200x Faster Than Disk. Prefer In-Memory Operations Over Disk I/O Wherever Feasible. Compress Data Before Network Transfer When Payload Exceeds 1 KB

---

## Domain Name System

### 1. Core Model
- **Hierarchical Resolution**: DNS Translates Human-Readable Domain Names (`www.example.com`, ...) To IP Addresses Via A Hierarchical Tree Of Authoritative Servers
- **Record Types**: `A` (Domain To IPv4), `AAAA` (Domain To IPv6), `CNAME` (Alias To Another Name Or A Record), `MX` (Mail Exchange), `NS` (Authoritative Name Server), `TXT` (Arbitrary Text Used For SPF & DKIM)
- **Time To Live**: Controls How Long DNS Resolvers & Clients Cache A Record. Short TTLs Enable Fast Failover But Increase DNS Query Volume. Long TTLs Reduce Latency But Slow Down Changes

### 2. Traffic Routing Strategies
- **Weighted Round Robin**: Distributes Traffic Proportionally Across Backends. Enables A/B Testing & Gradual Rollouts Without Load Balancer Changes
- **Latency-Based Routing**: Routes Clients To The Nearest Region With Lowest Measured Round-Trip Time
- **Geolocation Routing**: Routes Clients To Specific Endpoints Based On Their Geographic Origin For Data Residency Compliance
- **Failover Routing**: Primary Record Served Normally. Secondary Activated Only When Health Checks On Primary Fail

### 3. Managed DNS Services
- **AWS Route 53**: Supports All Routing Policies, Health Checks, Private Hosted Zones For VPC Internal Resolution & Traffic Flow Visual Editor
- **Cloudflare**: Combines Managed DNS With CDN, DDoS Protection & Proxied Records That Hide Origin IPs
- **Service Mesh DNS**: Kubernetes CoreDNS Provides In-Cluster Service Discovery Via `<service>.<namespace>.svc.cluster.local` Records

### 4. Disadvantages & Failure Modes
- **Propagation Delay**: TTL Expiry Causes Stale Records To Persist In Resolver Caches. Reduce TTL Well Before Planned Migrations
- **DDoS Amplification**: DNS UDP Responses Can Be 70x Larger Than Queries Enabling Amplification Attacks. Mitigated Via Rate Limiting, DNSSEC & Response Rate Limiting
- **Single Point Of Failure**: Managed DNS Providers Have Historically Suffered Outages Bringing Down Entire Sectors Of The Internet

---

## Content Delivery Network

### 1. Core Model
- **Global Proxy Network**: Globally Distributed Network Of Edge Servers Caching & Serving Content From Locations Closest To Users, Reducing Origin Load & Improving Latency
- **Static vs Dynamic Content**: Static Files Served Directly From Cache. Some CDNs (AWS CloudFront, Cloudflare Workers, ...) Support Dynamic Edge Logic & Personalization

### 2. Push vs Pull CDNs
- **Push CDNs**: Content Uploaded Directly To CDN Whenever Origin Changes. Full Control Over Cache Lifetime. Best For Sites With Small Traffic Volume Or Infrequently Updated Content
- **Pull CDNs**: CDN Fetches Content From Origin On First User Request Then Caches It For TTL Duration. Minimizes Storage Overhead. Best For High-Traffic Sites. Risk Of Cache Stampede On Cold Start

### 3. Advanced CDN Capabilities
- **Edge Compute**: Running Serverless Functions At CDN Edge (Cloudflare Workers, Lambda@Edge) For Request Transformation, Auth, Bot Filtering & A/B Testing Without Round-Tripping To Origin
- **Cache Invalidation Strategies**: URL-Based Purging, Tag-Based Purge, Surrogate Keys & Cache-Control `stale-while-revalidate` For Background Refresh Without User-Visible Latency
- **Origin Shield**: Intermediate Caching Layer Between Edge Nodes & Origin Reducing Origin Request Fan-Out During Cache Misses

### 4. Disadvantages
- **Cost**: Traffic-Based Pricing. Must Weigh CDN Cost Against Compute & Bandwidth Savings From Offloading Origin
- **Content Staleness**: Updates May Not Reach Edge Nodes Until TTL Expires Without Explicit Purging
- **URL Rewriting**: All Static Asset URLs Must Reference CDN Hostnames, Requiring Build Pipeline Coordination

---

## Load Balancer

### 1. Core Model
- **Traffic Distribution**: Distributes Incoming Client Requests Across A Pool Of Backend Servers. Returns Backend Responses To Clients. Eliminates Single Points Of Failure & Prevents Resource Overloading
- **Hardware vs Software**: Hardware Load Balancers (F5, ...) Are Expensive But High-Performance. Software Solutions (NGINX, HAProxy, AWS ELB) Are Flexible & Cost-Efficient

### 2. Layer 4 vs Layer 7 Load Balancing
- **Layer 4: Transport Layer**: Routing Decisions Based On Source/Destination IP Addresses & TCP/UDP Ports Without Inspecting Payload. Lower Latency & CPU Overhead. Performs Network Address Translation. Suitable For Raw TCP Streaming & UDP Traffic
- **Layer 7: Application Layer**: Routing Based On HTTP Headers, URLs, Cookies & Message Content. Terminates TCP Connections & Opens New 1s To Backends. Enables Content-Based Routing (Route `/api` To API Servers, `/static` To CDN Origins, Video Traffic To Dedicated Video Servers, ...)

### 3. Balancing Algorithms
- **Round Robin**: Requests Distributed Sequentially Across All Healthy Backends. Simple & Even For Homogeneous Workloads
- **Weighted Round Robin**: Assigns More Requests To Higher-Capacity Servers. Useful During Canary Deployments Or Heterogeneous Fleet
- **Least Connections**: Routes To Backend With Fewest Active Connections. Better For Long-Lived Or Variable-Duration Requests
- **IP Hash**: Consistent Hashing On Client IP Ensures Same Client Always Hits Same Backend. Provides Sticky Sessions Without Shared Session State But Breaks On IP Changes (NAT, Mobile)
- **Random With 2 Choices**: Samples 2 Backends Randomly & Routes To The Less Loaded. Near-Optimal Load Distribution With Low Overhead

### 4. Features & Capabilities
- **Health Checks**: Passive (Mark Unhealthy After N Failures) & Active (Synthetic Probes Via HTTP/TCP) Detection. Automatic Traffic Rerouting On Failure
- **SSL Termination**: Decrypts TLS At Load Balancer So Backend Servers Use Plain HTTP Internally, Reducing Certificate Management Complexity & Compute Overhead
- **Session Persistence (Sticky Sessions)**: Issues Cookies Or Uses IP Hash To Route A Client To The Same Backend For Session State Affinity. Introduces Uneven Load Distribution
- **Connection Draining**: Before Removing A Backend, Allow In-Flight Requests To Complete Before Sending No New Traffic

### 5. Horizontal Scaling
- **Stateless Servers**: Horizontal Scaling Requires Servers To Be Stateless. Session Data Stored In Centralized Stores Like Redis Or Memcached Rather Than Local Memory
- **Auto-Scaling Integration**: Cloud Load Balancers (AWS ALB/NLB, GCP HTTPS LB) Integrate With Auto-Scaling Groups, Dynamically Registering & Deregistering Instances

### 6. Disadvantages
- **Single Point Of Failure**: A Single Load Balancer Becomes A New SPOF. Mitigated By Active-Passive Or Active-Active Load Balancer Pairs
- **Performance Bottleneck**: Under-Provisioned Load Balancers Become Network Throughput Bottlenecks
- **Increased Complexity**: Introducing Load Balancers Adds Operational Complexity For Certificate Management, Health Tuning & Logging

---

## Reverse Proxy

### 1. Core Model
- **Unified Entry Point**: Centralizes Internal Services Behind A Single External-Facing Interface. Forwards Client Requests To Appropriate Backend Services & Returns Their Responses
- **Load Balancer vs Reverse Proxy**: Load Balancers Route To Pools Of Servers Serving The Same Function. Reverse Proxies Can Route To Different Services Based On Path Or Host. Tools Like NGINX & HAProxy Serve Both Roles Simultaneously

### 2. Benefits
- **Security**: Hides Backend Server Topology. Blacklists IPs. Limits Connections Per Client. Enforces Rate Limits
- **SSL Termination**: Centralizes TLS Certificate Management. Backends Communicate In Plain HTTP
- **Compression**: Applies gzip/Brotli Compression To Responses Reducing Network Bandwidth
- **Caching**: Serves Cached Responses For Repeated Requests Without Hitting Backends
- **Static Content Serving**: Directly Serves HTML, CSS, JS & Images Using Efficient Kernel-Level `sendfile` Transfers

### 3. Disadvantages
- **Increased Complexity**: An Additional Hop In The Request Path. Configuration Errors Can Cause Subtle Routing Bugs
- **Single Point Of Failure**: Requires Multiple Reverse Proxy Instances In Active-Active Or Active-Passive Configuration For High Availability

---

## Application Layer

### 1. Web Layer vs Application Layer Separation
- **Separation Of Concerns**: Decoupling Web Layer (HTTP Routing, TLS, Static Files) From Application Layer (Business Logic, API Processing) Allows Each To Scale Independently
- **Single Responsibility Principle**: Small, Autonomous Services Each Owning A Single Business Capability Enable Independent Deployment, Scaling & Failure Isolation
- **Benefits**: Adding A New API Endpoint Requires Adding Application Servers Without Necessarily Scaling Web Servers. Enables Teams To Own & Deploy Their Service Independently

### 2. Microservices
- **Definition**: Suite Of Independently Deployable, Small, Modular Services Each Running A Unique Process & Communicating Via Well-Defined Lightweight Mechanisms (REST, gRPC, Message Queues)
- **Service Decomposition Patterns**: Decompose By Business Capability Or By Subdomain Using Domain-Driven Design Bounded Contexts
- **Inter-Service Communication**:
  - **Synchronous (REST/gRPC)**: Direct Request-Response. Simpler But Creates Temporal Coupling. Cascading Failures Propagate If Downstream Services Are Slow
  - **Asynchronous (Message Queues)**: Producer Publishes Events. Consumers Process Independently. Decouples Services In Time & Failure Domains But Increases Operational Complexity

### 3. Service Discovery
- **Client-Side Discovery**: Services Query A Registry (Consul, Eureka) Directly & Perform Load Balancing Locally. Gives Clients Full Control But Couples Client To Registry API
- **Server-Side Discovery**: Clients Route To A Load Balancer Or Service Mesh That Queries The Registry Transparently. Clients Remain Unaware Of Discovery Mechanism
- **Service Registries**: Consul, etcd & ZooKeeper Maintain Registered Service Names, Addresses & Ports. Health Checks Verify Service Integrity Via HTTP Endpoints. Built-In Key-Value Stores Allow Sharing Configuration Values

### 4. API Gateway
- **Responsibilities**: Single Ingress Point Handling Authentication, Authorization, Rate Limiting, Request Routing, Protocol Translation, Request/Response Transformation & API Versioning
- **Backend For Frontend Pattern**: Separate API Gateway Instances Per Client Type (Mobile, Web, ...) Tailoring Response Shape & Protocol To Each Consumer's Needs
- **Gateway vs Service Mesh**: API Gateways Handle North-South (External) Traffic. Service Meshes Handle East-West (Service-To-Service) Traffic With mTLS, Observability & Traffic Policy

### 5. Disadvantages
- **Distributed Systems Complexity**: Requires Distributed Tracing, Centralized Logging & Service-Level Monitoring Across Dozens Of Independent Processes
- **Operational Overhead**: Each Microservice Requires Its Own Build Pipeline, Deployment Configuration, Health Checks & On-Call Rotation

---

## Database

### 1. Relational Databases
- **ACID Properties**: Atomicity (Transaction Is All-Or-Nothing), Consistency (Transactions Bring DB From 1 Valid State To Another), Isolation (Concurrent Transactions Produce Same Result As Serial Execution), Durability (Committed Transactions Persist Even After Crashes)
- **Use Cases**: Structured Data With Strict Schema, Complex Multi-Table Joins, Financial Transactions, Strong Consistency Requirements & Lookups By Index
- **Schema Design Principles**: Normalize To Third Normal Form To Eliminate Data Redundancy. Denormalize Strategically For Read-Heavy Workloads To Avoid Expensive Joins

### 2. Scaling RDBMS: Replication
- **Master-Slave Replication**: Master Handles All Reads & Writes & Asynchronously Replicates Writes To 1 Or More Read Replicas. If Master Goes Offline A Slave Can Be Promoted. Adds Read Throughput But Not Write Throughput
- **Master-Master Replication**: Both Masters Accept Reads & Writes & Coordinate With Each Other. Increases Write Availability But Requires Conflict Resolution For Simultaneous Writes To The Same Data. Risk Of Violating ACID If Loosely Consistent
- **Replication Lag**: Asynchronous Replication Means Replicas May Return Stale Reads. Critical For Use Cases Requiring Read-After-Write Consistency
- **Semi-Synchronous Replication**: Master Waits For At Least 1 Replica To Acknowledge Write Before Confirming To Client. Balances Durability Against Write Latency

### 3. Scaling RDBMS: Partitioning
- **Federation (Functional Partitioning)**: Splits Database By Function Into Separate Databases. Reduces Per-Database Load & Replication Lag. Cross-Database Joins Require Application-Level Logic Or Service Calls
- **Sharding (Horizontal Partitioning)**: Distributes Rows Across Multiple Database Instances Where Each Shard Owns A Subset Of Data. Common Shard Keys Include User ID Hash, Geographic Region & Date Range
  - **Advantages**: Linear Read/Write Scalability, Reduced Index Size Per Shard, Isolated Failure Domain
  - **Disadvantages**: Application Logic Complexity, Cross-Shard Joins Are Expensive, Rebalancing Shards During Cluster Growth Requires Careful Planning
  - **Consistent Hashing**: Minimizes Data Movement When Adding/Removing Shards By Mapping Keys & Nodes To A Hash Ring

### 4. Scaling RDBMS: Query Optimization
- **Denormalization**: Adds Redundant Data Copies Across Tables To Avoid Expensive Joins. Improves Read Performance At Cost Of Write Complexity & Storage. Materialized Views In PostgreSQL/Oracle Automate Keeping Redundant Copies Consistent
- **Indexing**: B-Tree Indexes Enable Lookups. Composite Indexes Accelerate Multi-Column WHERE Clauses. Covering Indexes Satisfy Queries Entirely From Index Without Table Lookups. Over-Indexing Slows Write Throughput
- **SQL Tuning**: Use `EXPLAIN`/`EXPLAIN ANALYZE` To Profile Query Execution Plans. Avoid `SELECT *`. Use `CHAR` Over `VARCHAR` For Fixed-Length Fields. Partition Hot Tables. Tune Query Cache & Connection Pool Sizes
- **Read Replicas For Analytics**: Route Long-Running OLAP Queries To Dedicated Read Replicas To Avoid Impacting OLTP Performance On Primary

### 5. NoSQL Databases
- **BASE Properties**: Basically Available (System Guarantees Availability), Soft State (State May Change Over Time Even Without Input), Eventual Consistency (System Will Become Consistent After Input Stops)
- **Key-Value Stores**: O(1) Reads & Writes. Backed By Memory Or SSD. Ideal For Caching, Session Storage & Metadata Lookup. Limited To Get/Set/Delete Operations. Examples: Redis, DynamoDB, Memcached
- **Document Stores**: Stores JSON/BSON/XML Documents Where Each Document Contains All Information For A Given Object. Flexible Schema Without Rigid Table Joins. Ideal For Product Catalogs, Content Management & User Profiles. Examples: MongoDB, CouchDB, Firestore
- **Wide Column Stores**: Data Organized As Rows With Dynamic Column Families. Keys Stored In Lexicographic Order Enabling Efficient Range Scans. High Availability & Scalability For Time-Series, IoT & Analytics Workloads. Examples: Cassandra, HBase, Bigtable
- **Graph Databases**: Nodes Represent Entities & Edges Represent Relationships. Optimized For Traversals With Many-To-Many Relationships. Ideal For Social Networks, Fraud Detection & Recommendation Engines. Examples: Neo4j, Amazon Neptune

### 6. SQL vs NoSQL Decision Matrix
- **Choose SQL When**: Data Is Structured With Strict Schema, Relationships Require Complex Joins, Transactions Are Critical, Team Prefers Mature Tooling & ACID Guarantees Are Non-Negotiable
- **Choose NoSQL When**: Schema Is Flexible Or Rapidly Evolving, Data Volume Reaches Multi-TB/PB Scale, Horizontal Write Scalability Is Required, Access Patterns Are Simple & Predictable, Or Use Case Demands Ultra-Low Latency At High Throughput

### 7. NewSQL & Distributed SQL
- **NewSQL Design Goal**: Combines ACID Guarantees Of Traditional RDBMS With Horizontal Scalability Of NoSQL. Replicates Via Raft Or Paxos Consensus
- **Examples**: Google Spanner (Globally Distributed ACID Transactions Using TrueTime API), CockroachDB (PostgreSQL-Compatible Distributed SQL), TiDB (MySQL-Compatible Horizontal Scaling)
- **Use Cases**: Global Financial Platforms, Multi-Region E-Commerce & Any Workload Requiring Both Strong Consistency & Horizontal Write Throughput

---

## Caching

### 1. Core Model & Cache Hierarchy
- **Purpose**: Reduces Latency & Database Load By Storing Frequently Accessed Data In Fast In-Memory Stores. Dispatcher Checks Cache Before Hitting Database
- **Cache Layers**: Client-Side (Browser Cache, OS Cache), CDN Cache, Web Server Cache (Varnish), Application Cache (Redis, Memcached), Database Query Cache & CPU Hardware Caches

### 2. Cache Update Strategies
- **Cache-Aside (Lazy Loading)**: Application Checks Cache First. On Miss, Loads From DB, Populates Cache & Returns Data. Only Requested Data Is Cached Avoiding Cold Data Accumulation
  - **Disadvantage**: 3 Round Trips On Cache Miss. Data Can Become Stale Between DB Writes & Cache Expiry
- **Write-Through**: Application Writes To Cache. Cache Synchronously Writes To DB Before Returning. Cache Always Consistent With DB
  - **Disadvantage**: Every Write Incurs DB Round-Trip Overhead. Rarely Accessed Data Wastes Cache Space
- **Write-Behind (Write-Back)**: Application Writes To Cache. Cache Asynchronously Flushes To DB In Background. Dramatically Improves Write Throughput
  - **Disadvantage**: Risk Of Data Loss If Cache Crashes Before Flushing. More Complex To Implement Correctly
- **Refresh-Ahead**: Cache Proactively Refreshes Entries Before TTL Expires Based On Access Pattern Prediction. Reduces Latency For Hot Keys
  - **Disadvantage**: Wasted Computation If Prediction Is Wrong. Increases Background Load On DB

### 3. Cache Invalidation & Consistency
- **TTL-Based Expiration**: Simplest Strategy. Data Automatically Invalidated After Fixed Duration. Choose TTL Based On Data Freshness Tolerance
- **Event-Driven Invalidation**: Database Changes Publish Events (Via CDC, Triggers Or Application Code) That Explicitly Delete Or Update Cache Entries. More Complex But Ensures Freshness
- **Cache Stampede (Thundering Herd)**: Simultaneous Expiration Of Popular Keys Causes Burst Of DB Queries. Mitigated Via Mutex Locks On Cache Misses, Request Coalescing Or Probabilistic Early Expiration (XFetch Algorithm)
- **Cache Poisoning**: Attackers Insert Malicious Responses Into Caches. Mitigated Via Input Validation, Cache-Key Normalization & Signed Content

### 4. Distributed Caching
- **Consistent Hashing For Cache Nodes**: Maps Cache Keys & Nodes To A Hash Ring To Minimize Key Remapping When Nodes Are Added Or Removed
- **Replication**: Replica Cache Nodes Provide High Availability. Reads Served From Replicas While Primary Handles Writes
- **Hot Key Problem**: Single Highly Requested Keys Exceeding Single-Node Throughput. Mitigated Via Local In-Process Caches, Read Replicas Or Key Sharding With Prefix Randomization

### 5. Caching At The Object Level
- **Serialized Object Caching**: Cache Fully Assembled Application Objects (User Sessions, Product Listings, Rendered HTML Fragments) Avoiding Repeated DB Queries & Join Processing
- **Avoid File-Based Caching**: File-Based Caches Complicate Server Cloning & Auto-Scaling. Prefer Distributed In-Memory Stores

### 6. Disadvantages
- **Cache Invalidation Complexity**: Keeping Cache Consistent With Database Is A Hard Problem. Stale Data Can Cause Incorrect Behavior
- **Memory Cost**: RAM Is Expensive. Cache Size Must Be Bounded. Eviction Policies (LRU, LFU, TTL) Determine What Gets Evicted Under Memory Pressure
- **Cold Start**: New Cache Instances Start Empty, Temporarily Increasing DB Load Until Warmed Up

---

## Asynchronism & Message Queues

### 1. Core Model
- **Motivation**: Decouples Producer & Consumer. Smooths Traffic Spikes. Enables Retry On Failure. Removes Long-Running Work From Synchronous Request Path. Improves User-Perceived Responsiveness
- **Message Queue Workflow**: Producer Publishes Job To Queue → Consumer Picks Up Job → Processes It → Signals Completion. User Is Not Blocked

### 2. Message Queue Patterns
- **Point-To-Point (Work Queue)**: Single Queue With Competing Consumers. Each Message Processed By Exactly 1 Consumer. Horizontal Scaling Achieved By Adding More Workers
- **Publish-Subscribe (Fan-Out)**: Producer Publishes To A Topic. Multiple Independent Consumers Each Receive A Copy. Enables Event-Driven Architecture Where Multiple Services React To The Same Event
- **Request-Reply**: Requestor Publishes To A Request Queue & Includes A Reply-To Queue Address. Responder Processes & Publishes Result To Reply Queue. Enables Asynchronous RPC Patterns

### 3. Delivery Semantics
- **At-Most-Once**: Message Acknowledged Before Processing. Risk Of Data Loss On Consumer Crash. Suitable For Metrics Or Logging Where Occasional Loss Is Acceptable
- **At-Least-Once**: Message Acknowledged After Processing. Duplicate Messages Possible On Retry. Requires Idempotent Consumers
- **Exactly-Once**: Most Complex. Requires Transactional Semantics Between Broker & Consumer State Store. Supported In Kafka With Idempotent Producers + Transactional API

### 4. Back Pressure
- **Problem**: If Producers Outpace Consumers, Queue Grows Until Memory Is Exhausted Causing Disk Spills, Cache Evictions & System Degradation
- **Solutions**: Return `503 Service Unavailable` With `Retry-After` Header When Queue Is Full, Implement Exponential Backoff On Producers, Apply Rate Limiting At Ingestion & Scale Consumers Based On Queue Depth Metrics

### 5. Task Queues
- **Definition**: Receive Tasks With Related Data, Execute Them (Potentially On Separate Workers) & Deliver Results. Support Scheduling & Computationally Intensive Background Jobs
- **Examples**: Celery (Python), Sidekiq (Ruby), Bull (Node.js), Temporal (Durable Workflows)
- **Use Cases**: Email Sending, Image Processing, Video Transcoding, PDF Generation, Scheduled Reports & Data Export Jobs

### 6. Disadvantages
- **Operational Complexity**: Message Brokers Add Infrastructure To Deploy, Monitor & Maintain
- **Observability Challenges**: Async Flows Are Harder To Trace End-To-End Without Distributed Tracing & Correlation IDs
- **Latency Addition**: For Low-Complexity Operations, Queue Overhead May Exceed Inline Processing Latency

---

## Communication Protocols

### 1. HTTP
- **Core Model**: Request-Response Protocol Where Clients Issue Requests With Verbs & Resource Paths. Servers Return Status Codes & Response Bodies
- **HTTP Verbs**: `GET` (Read, Idempotent, Safe), `POST` (Create Or Trigger), `PUT` (Create Or Replace, Idempotent), `PATCH` (Partial Update), `DELETE` (Remove, Idempotent)
- **HTTP/1.1 vs HTTP/2 vs HTTP/3**:
  - **HTTP/1.1**: 1 Request Per TCP Connection Without Pipelining. Head-Of-Line Blocking At The Application Layer
  - **HTTP/2**: Multiplexes Multiple Requests Over A Single TCP Connection. Header Compression Via HPACK. Server Push. Eliminates Application-Level Head-Of-Line Blocking
  - **HTTP/3**: Uses QUIC Over UDP. Eliminates TCP Head-Of-Line Blocking At Transport Layer. Faster Connection Establishment Via 0-RTT Resumption. Better Performance On Lossy Mobile Networks

### 2. TCP
- **Connection-Oriented Protocol**: 3-Way Handshake (SYN, SYN-ACK, ACK) Establishes Connection Before Data Transfer
- **Reliability Guarantees**: Sequence Numbers Ensure In-Order Delivery. Acknowledgments & Automatic Retransmission Handle Packet Loss. Checksum Fields Detect Corruption
- **Flow & Congestion Control**: Sliding Window Prevents Receiver Buffer Overflow. Congestion Control Algorithms (CUBIC, BBR) Probe Available Bandwidth
- **Use TCP When**: Data Integrity Is Critical (File Transfers, Database Connections, SSH, SMTP, HTTP)

### 3. UDP
- **Connectionless Protocol**: No Handshake. Datagrams Sent Independently. No Guaranteed Delivery, Ordering Or Duplication Avoidance
- **Performance**: Lower Latency & CPU Overhead Than TCP. Supports Broadcast & Multicast
- **Use UDP When**: Latency Is More Critical Than Reliability (DNS, VoIP, Video Streaming, Online Gaming, DHCP, QUIC)

### 4. Remote Procedure Call
- **Model**: Client Invokes A Procedure On A Remote Server As If It Were A Local Function Call. Framework Handles Serialization (Marshalling), Network Transport & Deserialization
- **Popular Frameworks**: gRPC (Protocol Buffers Over HTTP/2. Bi-Directional Streaming. Auto-Generated Client Stubs. Ideal For Internal Microservices), Thrift, Avro
- **Advantages**: Strong Typing Via IDL Contracts, Efficient Binary Serialization, Streaming Support In gRPC, Auto-Generated Client Libraries
- **Disadvantages**: Tight Coupling Between Client & Server Implementation. More Difficult To Debug Than REST. Not Easily Accessible From Browsers Without Proxies (gRPC-Web)

### 5. Representational State Transfer
- **Architectural Constraints**: Stateless (Server Stores No Client Session State), Client-Server Separation, Uniform Interface (Resources Identified By URIs), Cacheable Responses, Layered System
- **REST Principles**: Resources Identified By URIs. Actions Expressed Via HTTP Verbs. Responses Are Self-Descriptive Representations Of Resources. HATEOAS Links Enable Clients To Navigate APIs Dynamically
- **Advantages**: Human-Readable, Easily Cached, Browser-Compatible, Widely Understood & Tooled
- **Disadvantages**: Over-Fetching (Response Contains More Fields Than Client Needs), Under-Fetching (Multiple Round Trips Required To Assemble A Composite View), Not Ideal For Complex Hierarchical Or Graph Data Access Patterns

### 6. GraphQL
- **Model**: Query Language & Runtime Where Clients Specify Exactly What Data They Need In A Single Request. Server Returns Precisely That Shape
- **Advantages**: Eliminates Over-Fetching & Under-Fetching. Single Endpoint. Schema Acts As Contract. Enables Rapid Frontend Iteration Without API Versioning
- **Disadvantages**: Caching Is More Complex (POST Bodies Not Natively Cached By HTTP Intermediaries). N+1 Query Problem Requires DataLoader Batching. Query Complexity Attacks Require Depth & Complexity Limiting

### 7. WebSockets
- **Full-Duplex Communication**: Upgrades HTTP Connection To A Persistent Bi-Directional Channel. Server Can Push Messages To Client Without Client Polling
- **Use Cases**: Real-Time Chat, Live Dashboards, Collaborative Editing, Gaming, Live Sports Scores & Financial Tickers
- **Disadvantages**: Stateful Connections Complicate Horizontal Scaling. Requires Sticky Sessions Or Shared Pub/Sub Backbone (Redis Pub/Sub, Kafka) For Multi-Node Deployments

### 8. Server-Sent Events
- **Model**: Unidirectional Server-To-Client Event Stream Over HTTP. Client Subscribes To Event Stream. Server Pushes Updates Whenever They Occur
- **Use Cases**: Live Feeds, Notification Streams & Progress Updates Where Client-To-Server Communication Is Not Required
- **Advantages Over WebSockets**: Works Over Standard HTTP, Auto-Reconnect Built Into Browser API, Simpler Load Balancer Configuration

---

## Security

### 1. Core Security Principles
- **Defense In Depth**: Apply Multiple Independent Security Controls At Each Layer So Bypassing 1 Control Does Not Compromise The System
- **Principle Of Least Privilege**: Grant Each Component, Service & User The Minimum Permissions Required To Perform Its Function. Reduces Blast Radius Of Compromised Credentials
- **Zero Trust Model**: Never Trust, Always Verify. Authenticate & Authorize Every Request Even From Internal Services. Enforce Mutual TLS Between All Services

### 2. Authentication & Authorization
- **Authentication**: Verifying Identity Of A User Or Service. Methods Include Password + MFA, OAuth 2.0 + OIDC, SAML, API Keys, mTLS Certificates & SPIFFE/SPIRE For Workload Identity
- **Authorization**: Determining What An Authenticated Principal Is Permitted To Do. Approaches Include Role-Based Access Control, Attribute-Based Access Control & OPA/Rego Policy Engines
- **JSON Web Tokens**: Signed (JWS) Or Encrypted (JWE) Tokens Containing Claims. Stateless Verification Without DB Lookups. Short Expiry (15-60 Minutes) + Refresh Tokens For Session Management
- **OAuth 2.0 Flows**: Authorization Code + PKCE (Web/Mobile Apps), Client Credentials (Service-To-Service), Device Code (IoT/CLI) & Resource Owner Password (Legacy. Avoid)

### 3. Encryption
- **Encryption In Transit**: All External & Internal Traffic Encrypted Via TLS 1.2+. Enforce TLS 1.3 For New Systems. Mutual TLS (mTLS) Authenticates Both Client & Server. Certificate Rotation Via ACME Protocol Or cert-manager
- **Encryption At Rest**: Encrypt Disks, Database Volumes & Backups. Use Cloud KMS (AWS KMS, GCP Cloud KMS) Or HashiCorp Vault For Key Management. Separate Key Encryption Keys From Data Encryption Keys
- **Key Rotation**: Regularly Rotate Encryption Keys. Support Dual-Key Periods During Rotation To Avoid Downtime

### 4. Common Vulnerabilities (OWASP Top 10)
- **Injection (SQL, NoSQL, LDAP, OS Command)**: Use Parameterized Queries, Prepared Statements & ORM Frameworks. Never Concatenate User Input Into Query Strings
- **Cross-Site Scripting**: Encode All User-Provided Data Before HTML Rendering. Enforce Content Security Policy Header. Use `HttpOnly` & `Secure` Cookie Flags
- **Cross-Site Request Forgery**: Validate CSRF Tokens On State-Changing Requests. Use `SameSite=Strict` Cookie Attribute. Verify `Origin` & `Referer` Headers
- **Insecure Direct Object Reference**: Validate Authorization For Every Resource Access. Never Expose Sequential Integer IDs In Public APIs. Use UUIDs Or HMAC-Signed Identifiers
- **Security Misconfiguration**: Disable Default Credentials, Remove Debug Endpoints In Production, Enforce Security Headers (`X-Frame-Options`, `Strict-Transport-Security`, `X-Content-Type-Options`)

### 5. API Security
- **Rate Limiting**: Protect APIs From Abuse & DDoS Using Token Bucket Or Leaky Bucket Algorithms Enforced At API Gateway. Return `429 Too Many Requests` With `Retry-After` Header
- **Input Validation**: Reject Malformed Requests At API Gateway Before They Reach Business Logic. Validate JSON Schema, String Lengths, Allowed Character Sets & Numeric Ranges
- **Secrets Management**: Never Hardcode Secrets In Code Or Configuration Files. Use HashiCorp Vault, AWS Secrets Manager Or GCP Secret Manager. Inject Secrets At Runtime Via Environment Variables Or Mounted Volumes

### 6. Network Security
- **Network Segmentation**: Isolate Database Tier In Private Subnets With No Public Internet Exposure. Only Application Tier Accessible From DMZ. Use Security Groups & NACLs For Fine-Grained Ingress/Egress Control
- **Web Application Firewall**: Inspect & Filter HTTP Traffic Based On OWASP Rule Sets. Block SQLi, XSS & Bad Bot Patterns. Available In Cloud (AWS WAF, Cloudflare WAF) Or Self-Hosted (ModSecurity, NAXSI)
- **DDoS Mitigation**: Multi-Layered Approach Combining Cloud Scrubbing Services (AWS Shield, Cloudflare Magic Transit), Rate Limiting, IP Reputation Filtering & Anycast Network Diffusion

---

## Data Storage Systems

### 1. Object Storage
- **Model**: Flat Namespace Storing Binary Objects (Files, Images, Backups) Identified By Unique Keys. No Directory Hierarchy. Accessed Via HTTP REST API
- **Characteristics**: Massively Scalable (Exabytes), Cheap Per GB, High Durability (11 Nines), Strong Read-After-Write Consistency For New Objects
- **Use Cases**: Static Asset Hosting, Backup & Archival, Data Lake Storage, ML Training Data, Log Archive & Media Streaming
- **Examples**: AWS S3, Google Cloud Storage, Azure Blob Storage, MinIO (Self-Hosted)
- **Advanced Features**: Lifecycle Policies (Auto-Tier To IA/Glacier), Versioning, Cross-Region Replication, Presigned URLs (Temporary Access), Server-Side Encryption & Object Lock (WORM Compliance)

### 2. Block Storage
- **Model**: Exposes Raw Storage Volumes That Operating Systems Format With Filesystems. Low-Level, Low-Latency Random Access
- **Use Cases**: Database Primary Storage, VM Root Disks, Stateful Application Volumes
- **Examples**: AWS EBS, GCP Persistent Disk, Kubernetes Persistent Volumes Backed By CSI Drivers

### 3. File Storage (NAS)
- **Model**: Shared Filesystem Accessible By Multiple Clients Simultaneously Via NFS Or SMB Protocol
- **Use Cases**: Shared Configuration Files, Media Libraries Accessed By Multiple Servers, Legacy Application Migrations
- **Examples**: AWS EFS, GCP Filestore, Azure Files

### 4. Data Warehouses & Analytics
- **Model**: Columnar Storage Optimized For Online Analytical Processing Queries Scanning Billions Of Rows. Separate From OLTP Databases
- **Architecture**: Extract-Transform-Load Or Extract-Load-Transform Pipelines Feed Raw Data Into Warehouse. Dimensional Modeling (Star/Snowflake Schema) Organizes Data For Business Intelligence
- **Examples**: AWS Redshift, Google BigQuery, Snowflake, ClickHouse (OLAP For Real-Time Analytics)
- **Optimization**: Columnar Compression, Zone Maps (Min/Max Indexes Per Block), Materialized Views & Query Result Caching

---

## Scalability Patterns

### 1. Horizontal vs Vertical Scaling
- **Vertical Scaling (Scale Up)**: Adding More CPU, RAM Or Faster Disks To A Single Server. Simple But Has Hard Hardware Limits & Creates Single Points Of Failure. Good For Initial Growth Phases
- **Horizontal Scaling (Scale Out)**: Adding More Server Instances Behind A Load Balancer. Theoretically Unlimited Scalability. Requires Stateless Application Tier. More Cost-Efficient At Scale Using Commodity Hardware

### 2. Stateless Service Design
- **Requirements**: Servers Must Not Store Any Request-Specific State Locally. All State Stored In External Systems (Redis, DB, Blob Storage). Any Server Instance Can Handle Any Request
- **Session Externalization**: Store Session Tokens In Redis Or DynamoDB. Use Signed JWTs For Stateless Session Verification Without Server-Side Storage

### 3. Database Scalability Patterns
- **Read Replicas**: Offload Read Traffic To Replicas. Suitable For Read-Heavy Workloads (10:1 Or Higher Read-To-Write Ratios)
- **Command Query Responsibility Segregation**: Separate Write Model (Commands To RDBMS) From Read Model (Queries To Denormalized Read Store, Elasticsearch Or Redis). Each Side Scales Independently
- **Event Sourcing**: Store All State Changes As An Immutable Append-Only Log Of Events Rather Than Mutable Row Updates. Replay Events To Reconstruct Current State. Enables Audit Trails, Time Travel Queries & Event-Driven Downstream Projections

### 4. Caching For Scale
- **Multi-Level Cache**: L1 In-Process Cache (Small, Ultra-Fast) To L2 Distributed Cache (Redis/Memcached) To L3 Database. Higher Cache Hit Rate At Lower Tiers Reduces Downstream Pressure
- **Pre-Computation & Materialized Results**: Compute Expensive Aggregations Offline & Store Pre-Built Results For Fast Reads

### 5. Asynchronous Processing At Scale
- **Fan-Out On Write vs Fan-Out On Read**:
  - **Fan-Out On Write (Push Model)**: When A User Posts, Immediately Write To All Followers' Feeds. Low Read Latency But High Write Amplification For Accounts With Millions Of Followers (Celebrity Problem)
  - **Fan-Out On Read (Pull Model)**: Assemble User Feed At Read Time By Querying All Followees. No Write Amplification But Higher Read Latency. Hybrid Approach Commonly Used In Practice
- **Rate Limiting Strategies**: Token Bucket (Fixed Capacity, Refills At Constant Rate), Leaky Bucket (Fixed Output Rate), Fixed Window Counter (Simple But Susceptible To Burst At Window Boundary), Sliding Window Log (Accurate But Memory-Intensive)

### 6. Resilience Patterns
- **Circuit Breaker**: Prevents Cascading Failures By Detecting When A Downstream Service Is Unhealthy & Failing Fast With A Fallback Response Without Waiting For Timeout. States Transition Closed To Open To Half-Open
- **Bulkhead**: Isolates Resources (Thread Pools, Connection Pools) Per Downstream Dependency So Failure In 1 Service Does Not Exhaust Shared Resources Needed For Other Services
- **Retry With Exponential Backoff & Jitter**: Automatically Retry Transient Failures. Exponential Backoff Reduces Stampede On Recovery. Jitter Spreads Retries Across Time To Avoid Synchronized Retry Storms

---

## Distributed Systems Concepts

### 1. Consensus & Leader Election
- **Raft Consensus Algorithm**: Leader-Based Consensus Protocol Where Leader Replicates Log Entries To Followers. Majority Quorum (N/2 + 1) Must Acknowledge Writes Before Committing. Leader Election Via Randomized Timeouts. Used In etcd, Kafka KRaft & CockroachDB
- **Paxos**: Classic Multi-Phase Consensus Protocol. More Theoretical Foundation. Harder To Implement Correctly Than Raft. Forms Basis Of Google Chubby & Spanner
- **Quorum Writes**: Require Acknowledgment From `W` Nodes. Quorum Reads Require `R` Nodes. Strong Consistency When `R + W > N` (Total Nodes)

### 2. Distributed Clocks & Ordering
- **Lamport Timestamps**: Logical Clocks That Capture Causal Ordering Of Events Without Synchronized Physical Clocks. If Event A Happens Before Event B, Timestamp(A) < Timestamp(B)
- **Vector Clocks**: Extends Lamport Timestamps To Track Causal History Per Node. Enables Detection Of Concurrent Conflicting Writes In Systems Like DynamoDB & Riak
- **Google TrueTime API**: Hardware-Backed Atomic Clock + GPS In Google Datacenters Provides Bounded Clock Uncertainty. Spanner Uses TrueTime To Assign Globally Consistent External Timestamps

### 3. Distributed Transactions
- **2-Phase Commit**: Coordinator Sends Prepare To All Participants. All Vote Commit Or Abort. Coordinator Sends Global Commit/Abort. Blocking Protocol Where Coordinator Failure Leaves Participants Locked
- **Saga Pattern**: Replaces Distributed Transactions With A Sequence Of Local Transactions. Each Publishes An Event Or Message Triggering The Next. Compensating Transactions Handle Rollback On Failure. 2 Implementations: Choreography (Event-Driven) & Orchestration (Central Coordinator)
- **Outbox Pattern**: Atomically Writes Business Data & Outbox Event Record To Local DB In Same Transaction. Separate Relay Process Reads Outbox & Publishes Events To Message Broker. Guarantees At-Least-Once Event Publishing Without Distributed Transactions

### 4. Data Replication
- **Synchronous Replication**: Leader Waits For All Replicas To Confirm Write Before Acknowledging Client. Zero Data Loss But Higher Write Latency
- **Asynchronous Replication**: Leader Acknowledges Client Immediately After Local Write. Replicas Catch Up Asynchronously. Lower Latency But Risk Of Data Loss On Leader Failure
- **Change Data Capture**: Captures Row-Level Changes From Database Transaction Logs (MySQL Binlog, PostgreSQL WAL) & Streams Them As Events. Enables Real-Time Data Pipelines Without Application-Level Changes. Tools: Debezium, AWS DMS

### 5. Distributed Coordination
- **Distributed Locking**: Acquiring Locks Across Multiple Nodes To Prevent Concurrent Access To Shared Resources. Implemented Via Redis (`SET NX EX`), ZooKeeper ephemeral znodes Or Database Advisory Locks
- **Leader Election**: Services Compete For A Lock In A Coordination Service. Lock Holder Acts As Leader. On Leader Failure, Lock Expires & A New Election Occurs. Critical For Single-Writer Consistency In Stateful Systems
- **Service Mesh (Istio/Linkerd)**: Infrastructure Layer Providing Automatic mTLS, Traffic Management (Retries, Timeouts, Circuit Breaking), Distributed Tracing & Fine-Grained Traffic Splitting Without Application Code Changes

### 6. Observability In Distributed Systems
- **Distributed Tracing**: Assigns A Unique Trace ID To Each Request. Propagated Across All Microservices Via Headers (W3C TraceContext, B3). Collected By Jaeger, Zipkin Or Tempo. Enables End-To-End Latency Analysis & Root Cause Identification
- **Structured Logging**: Log Every Event As JSON With Consistent Fields (`trace_id`, `span_id`, `service`, `level`, `timestamp`). Enables Correlation Across Services & Efficient Search In Elasticsearch Or Loki
- **SLI/SLO/SLA**: Service Level Indicators (Measurable Metrics: p99 Latency, Error Rate), Service Level Objectives (Target Thresholds: p99 < 200ms, Error Rate < 0.1%), Service Level Agreements (Business Contract For Consequences Of Missing SLOs)
- **Error Budgets**: 100% Minus SLO = Error Budget. Track Error Budget Burn Rate. Burn Rate Over 1x In 30 Days = SLO At Risk. Use Remaining Budget To Guide Risk Tolerance For Deployments & Experiments

---

## Cross-Cutting System Design Themes

### Architecture Principles
- **Single Responsibility**: Each Service, Module & Component Should Have 1 Clear Job. Simplifies Testing, Deployment & Ownership
- **Idempotency**: Design All APIs & Message Consumers So Retrying An Operation Produces The Same Result. Critical For Reliable Distributed Systems Where Failures Are Normal
- **Immutability**: Prefer Immutable Data Structures & Append-Only Logs Over Mutable State. Simplifies Concurrency, Caching, Replication & Audit Trails
- **Fail Fast**: Detect & Surface Errors Immediately Rather Than Silently Propagating Corrupt State Downstream

### Reliability Engineering
- **Chaos Engineering**: Deliberately Inject Failures (Kill Random Pods, Introduce Latency, Drop Network Packets) In Production Or Staging To Validate System Resilience. Tools: Chaos Monkey, Gremlin, Litmus Chaos
- **Graceful Degradation**: System Continues Operating At Reduced Functionality When Downstream Dependencies Are Unavailable. Show Cached Results, Use Feature Flags To Disable Non-Critical Paths, Return Partial Responses
- **SLO-Based Alerting**: Alert On Error Budget Burn Rate Rather Than Low-Level Metrics. Correlates To Customer Impact. Reduces Alert Fatigue From Non-Actionable Infrastructure Noise
- **Blameless Post-Mortems**: Document Timeline, Root Cause, Impact, Mitigations & Action Items Without Assigning Blame. Drives Systemic Reliability Improvements

### Capacity Planning
- **Estimation Framework**: Define Daily Active Users, Average Actions Per User Per Day & Data Per Action. Derive QPS, Storage Growth Rate & Peak Traffic (3-5x Average)
- **Storage Estimation**: `Storage/Day = QPS x Data Per Request x Seconds Per Day`. Plan For 3-Year Storage Horizon With 20% Buffer For Replication & Operational Overhead
- **Network Bandwidth**: Estimate Inbound (Write QPS x Payload Size) & Outbound (Read QPS x Response Size) Separately. CDN Offload Ratio Critical For Media-Heavy Workloads

### Disaster Recovery
- **Recovery Time Objective**: Maximum Acceptable Time For System To Be Unavailable After A Failure. Drives HA Architecture Decisions
- **Recovery Point Objective**: Maximum Acceptable Data Loss Measured In Time. Drives Backup Frequency & Replication Lag Budget
- **Multi-Region Active-Active**: Lowest RTO & RPO. Highest Operational Complexity & Cost. Requires Conflict-Free Replication & Global Load Balancing
- **Multi-Region Active-Passive**: Primary Region Serves All Traffic. Standby Region Receives Async Replication. Failover Triggered Manually Or By Health Checks. Higher RTO Than Active-Active
- **Backup Strategies**: Full Snapshots + Incremental Change Logs. Store Backups In Separate Region & Account. Regularly Test Restore Procedures In Isolated Environment