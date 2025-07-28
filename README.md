# Real-Time Sports Odds Infrastructure Design

## Sr. Infrastructure Engineer Take-Home Assessment

---


## Executive Summary

This infrastructure design outlines a scalable, resilient, and cost-conscious system for processing and distributing real-time sports odds updates across NFL, NBA, and MLB games. Given the stringent latency and uptime requirements—particularly during live games—adopt a **cloud-native, event-driven microservices architecture** that prioritizes low-latency streaming, horizontal scalability, and robust observability.

The system ingests diverse real-time sports feeds, processes on-field updates via containerized backend services on **Amazon EKS**, and publishes odds updates to a **Kafka broker** for downstream consumers in multiple cloud environments. Simultaneously, each odds update is stored in a **high-performance analytical database** (ClickHouse) to enable quantitative analysis and trend tracking.

leverage **Infrastructure as Code (Terraform)**, automated CI/CD pipelines, and centralized observability tooling to ensure operational consistency, rapid deployment, and system-wide visibility.

---


## Table of Contents

- [1. System Requirements Analysis](#1-system-requirements-analysis)
- [2. High-Level Architecture Design](#2-high-level-architecture-design)
- [3. Cloud Platform & Infrastructure Foundation](#3-cloud-platform--infrastructure-foundation)
- [4. Data Ingestion Strategy](#4-data-ingestion-strategy)
- [5. Kubernetes Platform Design](#5-kubernetes-platform-design)
- [6. Data Processing & Computing Layer](#6-data-processing--computing-layer)
- [7. Data Storage Solutions](#7-data-storage-solutions)
- [8. Message Streaming & Event Architecture](#8-message-streaming--event-architecture)
- [9. Observability & Monitoring](#9-observability--monitoring)
- [10. Infrastructure as Code & Automation](#10-infrastructure-as-code--automation)
- [11. Security, Compliance & Access Control](#11-security-compliance--access-control)
- [12. Cost Optimization & Scaling Strategy](#12-cost-optimization--scaling-strategy)
- [13. Risk Mitigation & Disaster Recovery](#13-risk-mitigation--disaster-recovery)
- [14. Delivery Summary & Final Recommendations](#14-delivery-summary--final-recommendations)
- [Appendices](#appendices)

---

## Key Decision Questions

#
## What is the primary architectural philosophy?

adopt a **cloud-native, event-driven microservices architecture**:

- **Microservices** for modularity, fault isolation, and team autonomy across sports and services.
- **Event-driven pipelines** (Kafka-centric) to enable real-time, loosely coupled data flows.
- **Kubernetes (EKS)** for orchestration, supporting containerized workloads in Go, Python, and Rust.
- **Immutable infrastructure + GitOps** for repeatable deployments and rollback safety.

---

#
## What are the critical success metrics we're optimizing for?

1. **Availability**
   - 99%+ uptime overall
   - 100% uptime during game hours (failover-ready ingest + autoscaling)

2. **Latency**
   - P50 latency: ≤ 3 seconds
   - P99 latency: < 5 seconds from ingest to Kafka + ClickHouse

3. **Scalability**
   - Ingest peak: 20K+ updates/sec (NFL)
   - Horizontal autoscaling at all layers (ingest, compute, streaming, storage)

4. **Data Integrity**
   - Exactly-once delivery to Kafka and analytics DB
   - Deduplication and out-of-order correction logic

5. **Operational Excellence**
   - Centralized observability
   - Automated IaC provisioning
   - Fast rollback and disaster recovery

---

#
## How do balance cost, performance, and reliability?

- **Performance-first where it counts**:
  - Low-latency Kafka
  - Fast compute in Go/Rust
  - ClickHouse for real-time analytics

- **Cost control via elasticity**:
  - Spot instances
  - Autoscaling node groups
  - Resource-efficient service design

- **Reliability through redundancy and failover**:
  - Multi-AZ EKS deployment
  - Kafka replication and partition tuning
  - Stateful backups and RTO/RPO planning

- **Avoid premature optimization**:
  - Start with colocated workloads (e.g., ClickHouse on EKS)
  - Maintain a clean path to decoupling as scale demands

---


## 1. System Requirements Analysis

#
## 1.1 Traffic Analysis & Capacity Planning

This section outlines the expected data volumes, system throughput, and planning considerations to ensure reliable and scalable handling of real-time sports odds updates.

##
## Expected Traffic Per Sport

| Sport | Updates per Game | Duration | Avg Updates/sec | Notes |
|-------|------------------|----------|------------------|-------|
| NFL   | 300M             | ~4 hrs   | ~21,000/sec      | Peak during game plays, touchdown drives, etc. |
| NBA   | 400K             | ~2 hrs   | ~55/sec          | Constant flow, minimal burstiness |
| MLB   | 200K             | ~3 hrs   | ~18/sec          | Moderate bursts (e.g., multiple hits/plays in an inning) |

**Assumed concurrent games during peak hours**:
- NFL: 4–6 games
- NBA: 5–8 games
- MLB: 3–5 games

##
## 🧮 Peak Concurrent Load Estimate

| Layer              | Estimated Throughput |
|-------------------|----------------------|
| Ingestion          | ~100K messages/sec (peak NFL + other feeds) |
| Odds processing    | ~100K compute ops/sec |
| Kafka production   | ~100K records/sec |
| Analytics ingest   | ~100K inserts/sec (ClickHouse or buffer→batch if needed) |

---

##
## 🔄 Handling Traffic Spikes (e.g., Playoffs, Super Bowl)

design for **burst tolerance** and **horizontal elasticity**:
- **Kubernetes HPA** for compute pods (based on CPU/memory/Kafka lag)
- **Kafka topic partitioning** for scaling producers and consumers
- **Redis buffering layer** for smoothing ingest-to-compute handoff during bursts
- **Dedicated autoscaling node groups** per service (ingest, calc, stream, analytics)
- **Over-provisioned ClickHouse write replicas** to absorb bursty writes

---

##
## 📦 Data Retention Requirements

| Data Type           | Retention | Purpose |
|---------------------|-----------|---------|
| Raw on-field updates | 7–14 days (optional) | Debugging, audit trail |
| Processed odds updates | 30–90 days hot | Live dashboards, quant research |
| Aggregated trends    | 1+ year (cold) | Long-term modeling, historical backtests |

Retention policies can be enforced via:
- TTL settings in ClickHouse
- Lifecycle rules for S3-based cold storage
- Partitioning on time (day/hour) to simplify purging

---

##
## 🧮 Storage Capacity Planning (Historical Odds)

Assume:
- ~300 bytes per odds update (player/team ID, timestamp, odds fields, metadata)
- ~100M–200M processed odds updates per NFL Sunday (~30–60 GB/day NFL alone)
- Multiply by 3 sports + playoff traffic → ~100 GB/day
- 30 days hot = ~3 TB
- 12 months cold archive = ~30–40 TB

**Storage Breakdown**:
- **Hot storage (ClickHouse)**: 3–5 TB (replicated x2 = 6–10 TB)
- **Cold archive (S3)**: ~40 TB/year with compression (~$800–$1200/year)

---

#
## Summary

- Peak capacity: **100K messages/sec** ingest & compute
- ClickHouse and Kafka are sized for **high-throughput, low-latency write pipelines**
- Horizontal scalability, TTL-based retention, and batch archival ensure the system grows sustainably
- Infrastructure designed for **real-world peak loads**, not just theoretical averages

---


## 2. High-Level Architecture Design

#
## 2.1 System Architecture Overview

This system follows a **cloud-native, event-driven microservices architecture** deployed on **Amazon EKS**, designed to process and distribute real-time sports odds updates at high scale. Services are organized by sport and function, and communication between components is orchestrated via Kafka to ensure loose coupling, scalability, and high availability.

```mermaid
flowchart TD

%% External Feeds
NFLKafka[External NFL Kafka Feed]
NBAPush[External NBA Push Stream]
MLBPubSub[External MLB Pub/Sub Feed]

%% Adapters
NFLKafka --> NFLIngest[NFL Ingest Adapter\n(Go Service)]
NBAPush --> NBAIngest[NBA Ingest Adapter\n(Go Service)]
MLBPubSub --> MLBIngest[MLB Ingest Adapter\n(Go Service)]

%% Internal Kafka - Raw
NFLIngest --> RawKafka[Internal Kafka Cluster\nTopics: nfl.raw, nba.raw, mlb.raw\nPartitions: 50/10/5\nReplication: 3x]
NBAIngest --> RawKafka
MLBIngest --> RawKafka

%% Event Normalizer
RawKafka --> Normalizer[Event Normalizer\n• Schema validation\n• Format standardization\n• Metadata enrichment\n• Deduplication]

%% Internal Kafka - Normalized
Normalizer --> NormKafka[Internal Kafka Cluster\nTopics: nfl.normalized, nba.normalized, mlb.normalized]

%% Odds Calculators
NormKafka --> NFLOdds[NFL Odds Calculator\n(Python/Rust)]
NormKafka --> NBAOdds[NBA Odds Calculator\n(Python/Rust)]
NormKafka --> MLBOdds[MLB Odds Calculator\n(Python/Rust)]

%% Odds Publisher
NFLOdds --> Publisher[Odds Publisher Service\n• Writes to:\n- External Kafka\n- ClickHouse\n- Redis]
NBAOdds --> Publisher
MLBOdds --> Publisher

%% Output Destinations
Publisher --> Redis[Redis Cache\nCluster Mode - 3 Nodes]
Publisher --> ClickHouse[ClickHouse Analytics Platform\nReplacingMergeTree + S3 Tiering]
Publisher --> ExtKafka[External Kafka\n(Cross-Account)]

%% External Kafka Consumers
ExtKafka --> AccountA[Account A - Consumer 1]
ExtKafka --> AccountB[Account B - Consumer 2]
```


##
## Decision: Architecture Style
- use a **pure event-driven microservices approach** to enable real-time responsiveness, modular deployments, and system resilience.
- Each microservice is stateless and independently scalable, mapped to ingestion, odds calculation, and publishing/storage responsibilities.

##
## Decision: Data Flow from Ingestion to Output
- Ingest services receive and normalize external feeds (Kafka, Push, Pub/Sub).
- Processed messages are routed through Kafka and simultaneously stored in ClickHouse.
- Kafka enables asynchronous communication with downstream systems and fan-out to external consumers.

##
## Decision: Isolation by Sport
- Pipelines are isolated by sport (NFL, NBA, MLB) at every layer: services, Kafka topics, Kubernetes deployments, and compute resources.
- This enables differentiated scaling, rollout strategies, and fault domains, with no cross-contamination between high-traffic games like NFL and lower-throughput sports.

##
## Decision: Fault Tolerance and Graceful Degradation
- Kafka ensures decoupled service communication and message durability.
- Redis-based buffering or disk queues handle temporary Kafka outages.
- Circuit breakers, retries, and DLQs (Dead Letter Queues) allow for robust failure handling without cascading errors.
- Stateful components like ClickHouse are replicated and distributed to maintain high availability.

---

#
## 2.2 Data Flow Architecture

The odds update pipeline spans from external feed ingestion through to streaming and analytics outputs, designed for consistency, low latency, and scalability.

##
## Decision: Pipeline Structure

Feed Input (Kafka/Push/PubSub)  
  ↓  
Ingest Service (per sport)  
  ↓  
Normalization & Validation  
  ↓  
Odds Calculation Service  
  ↓  
Publish to Kafka + Store in ClickHouse

- Ingestion services normalize disparate feed formats into a unified schema.
- Processing services calculate odds and enrich data before routing to Kafka and ClickHouse.

##
## Decision: Data Transformation and Enrichment
- Convert raw feed formats into a standard protobuf or JSON schema.
- Enrich messages with metadata (player/team info, game context, latency metrics).
- Add internal tracking data (event time, source ID, versioning).

##
## Decision: Data Validation and Error Handling
- Schema and business rule validation occurs in the ingestion layer.
- Invalid or malformed messages are logged and routed to DLQs.
- Transient errors are retried with exponential backoff.
- Anomalies or persistent failures are quarantined for manual inspection.

##
## Decision: Deduplication and Out-of-Order Handling
- Deduplication via event IDs, sequence numbers, or checksums.
- Kafka and ClickHouse both support idempotent writes.
- Event timestamps ensure chronological integrity in analytics even if arrival order is incorrect.
- Time-windowed sorting during ingestion is used when precise ordering is needed.

---

This architecture ensures:
- Modular and sport-specific pipelines
- Real-time, scalable, and fault-tolerant processing
- Durable message flows and consistent analytics ingestion
- Minimal latency with high throughput and traceable lineage

---


## 3. Cloud Platform & Infrastructure Foundation

#
## 3.1 Cloud Provider Selection

##
## Decision: Cloud Platform
choose **Amazon Web Services (AWS)** as the primary cloud provider due to:
- Native support for Kafka (MSK), EKS, and Redis/RDS
- Tight integration with ClickHouse Cloud or EC2-based self-managed deployments
- Global availability zones for multi-region failover
- Granular IAM for secure multi-account architecture

AWS's global backbone, mature container/Kubernetes offerings, and Kafka-native services make it ideal for low-latency real-time applications with cross-account streaming needs.

##
## Decision: Multi-Cloud vs. Multi-Region
- start with **single-cloud, multi-region** deployment for simplicity and redundancy.
- Multi-region allows geographic fault tolerance and regional autoscaling during major sporting events.
- Multi-cloud is deferred unless there are specific latency, regulatory, or partner-driven constraints.

##
## Decision: Kafka Integration Across Accounts
- Cross-account Kafka (e.g., MSK → Confluent Cloud, or MSK → external client accounts) is enabled via:
  - **PrivateLink** or **Transit Gateway** for secure, low-latency communication
  - **IAM role assumption** or token-based auth for producer/consumer auth
  - **TLS + ACLs** for end-to-end encryption and access control

##
## Decision: Network Implications of Cloud Choice
- AWS offers the most robust tooling for hybrid/multi-account Kafka integration.
- Leveraging **Transit Gateway + VPC Peering** gives secure, isolated cross-VPC Kafka traffic.
- **Elastic Load Balancing (ELB)** + **Route 53** handles multizone DNS resolution and failover.

---

#
## 3.2 Network Architecture

##
## Decision: VPC Design
- Each environment (dev, staging, prod) is provisioned in its own dedicated **AWS account** under a shared AWS Organization, and contains an isolated **VPC**.
- Each VPC has public and private subnets across **at least 3 AZs**.
- **NAT Gateways** enable secure outbound access for private subnets.
- **VPC Flow Logs** feed into observability stack for traffic auditing.

##
## Decision: Cross-Account & Cross-Cloud Connectivity
- **AWS Transit Gateway** connects multiple VPCs within the org.
- **VPC Peering** or **PrivateLink** enables secure integration with external Kafka sources or partners.
- If hybrid or partner cloud integrations are required, use **VPN tunnels or AWS Direct Connect**.

##
## Decision: Network Segmentation and Security Groups
- **Security groups** enforce least-privilege access per service.
- **Network ACLs** layer defense around subnet boundaries.
- Services are segmented by function and sensitivity (e.g., public API vs. internal odds compute).
- **Service mesh ingress/egress policies** are optionally enforced using tools like Istio or Cilium.

##
## Decision: DNS, Load Balancing, and Service Discovery
- **Route 53** provides global DNS routing and health-based failover.
- **AWS NLB + ALB** handle service exposure at the TCP and HTTP layers.
- **CoreDNS** (via Kubernetes) enables in-cluster service discovery.
- **Service Mesh (e.g., Istio)** may be layered in later for advanced routing and observability.

---

This foundation ensures:
- Secure, multi-AZ VPC design  
- Scalable and resilient cross-account connectivity  
- Clean service separation and discoverability  
- Strong integration with Kafka and Kubernetes-native services

---


## 4. Data Ingestion Strategy

#
## 4.1 Integration Path Selection

##
## Decision: Can Kafka be used for all integrations?

While Kafka is the preferred ingestion backbone due to its performance, durability, and decoupling capabilities, **cannot use Kafka for all integrations** without intermediate adapters. Some upstream feeds (e.g., Push Streams or third-party Pub/Sub services) do not natively support Kafka. Therefore:

- **Kafka is the unified internal messaging backbone**, but...
- **adapt non-Kafka sources** using lightweight ingest gateways that convert Push or Pub/Sub protocols into Kafka messages.

##
## Integration Strategy per Feed

| Feed Type  | Best Ingestion Method        | Notes |
|------------|------------------------------|-------|
| NFL (high-volume, structured) | **Kafka (native or MSK)** | Ideal for throughput and stream partitioning |
| NBA (medium-volume, web-socket push) | **Push Stream → Adapter → Kafka** | Use WebSocket or HTTP push handlers to convert to Kafka |
| MLB (low-volume, Pub/Sub) | **Pub/Sub → Adapter → Kafka** | Use pull-based polling service to convert messages |

##
## Message Format and Protocol Handling
- All adapters normalize external formats (e.g., XML, JSON, Protobuf) to an **internal unified schema** before publishing to Kafka.
- Lightweight Go or Python services can be used for protocol adaptation and validation.

##
## Latency Considerations
- Kafka supports sub-second delivery and millisecond-level publish/consume latency.
- Push/PubSub adapters must minimize transformation latency using in-memory processing.
- **End-to-end latency is optimized through backpressure buffering and horizontal scaling**.

##
## Backpressure and Flow Control
- Kafka's internal queueing manages spikes and temporary load imbalances.
- Producers are configured with `acks=all`, and consumers are scaled via HPA based on Kafka lag metrics.
- Redis or memory buffer can be used as a shock absorber for extreme burst handling.

---

#
## 4.2 Ingestion Layer Design

The ingestion layer is designed for reliability, scalability, and standardization across all feed types.

##
## Rate Limiting and Traffic Shaping
- **API Gateway (e.g., AWS API Gateway or Envoy)** applies request rate limits per IP/feed source.
- **In-memory or Redis-based queue buffers** regulate incoming event pressure before processing.

##
## Connection Failures and Retry Strategy
- Push-based handlers retry on socket failure with exponential backoff.
- Polling adapters for Pub/Sub retry on HTTP error or empty payload.
- Kafka producers use `retry.backoff.ms` and `delivery.timeout.ms` to maintain reliable delivery.

##
## Message Ordering and Deduplication
- Kafka partitions are designed to group messages by `game_id` or `player_id` to preserve ordering.
- Ingest services include `message_id` or `hash` to deduplicate downstream.
- Kafka consumer groups maintain offset tracking for exactly-once processing.

##
## Monitoring and Alerting
- Ingest service metrics: throughput, latency, queue depth
- Kafka metrics: lag, partition health, replication status
- Alerts on:
  - Feed downtime or HTTP/socket disconnects
  - Excessive retries or queue overflow
  - Consumer lag thresholds
- Visualized using **Prometheus + Grafana**, with logs to **CloudWatch or OpenSearch**

---

This ingestion strategy provides:
- Protocol-agnostic entry into a unified Kafka backbone  
- Modular adapters for evolving third-party formats  
- High-throughput, fault-tolerant, and observable ingestion layer

---


## 5. Kubernetes Platform Design

#
## 5.1 Kubernetes Distribution Selection

##
## Decision: Kubernetes Distribution
select **Amazon EKS (Elastic Kubernetes Service)** as the managed Kubernetes distribution due to:
- Tight AWS ecosystem integration with IAM, VPC, CloudWatch, and MSK
- Automatic control plane management, version upgrades, and patching
- Support for EC2, Fargate, and ARM-based instances for cost optimization
- Out-of-the-box security via IAM Roles for Service Accounts (IRSA)

##
## Decision: Managed vs. Self-Managed
- choose **fully managed EKS** to reduce operational overhead and improve stability.
- EKS supports upstream Kubernetes, enabling portability and compliance with CNCF standards.
- Self-managed clusters are avoided to reduce the burden of upgrades, security, and HA configurations.

##
## Decision: Cluster Lifecycle Management
- use **eksctl** or **Terraform** for cluster creation and version upgrades.
- **Amazon EKS Blueprints** and GitOps tooling like ArgoCD or Flux manage add-ons and environments.
- CI/CD pipelines validate infrastructure changes in staging before promoting to production.

##
## Decision: Autoscaling & Node Management
- **Cluster Autoscaler** dynamically scales EC2 nodes based on pending pod capacity.
- **Karpenter** or **Fargate** is evaluated for bursty, ephemeral workloads.
- Spot and on-demand node groups optimize cost and availability.

---

#
## 5.2 Cluster Architecture & Operations

##
## Decision: Cluster Design
- deploy **separate clusters per environment** (dev, staging, prod) for isolation and policy enforcement.
- **Single cluster with namespace isolation per sport** is sufficient due to shared infra needs and reduced cost/complexity.
- Multi-cluster federation is reserved for future multi-region or latency-critical scaling needs.

##
## Decision: Networking & Service Mesh
- **Amazon VPC CNI** plugin provides native pod networking with VPC IPs.
- **Calico** is used for network policy enforcement if needed.
- **Istio** or **Linkerd** is layered in for observability, mTLS, and traffic shaping as the platform matures.

##
## Decision: Secrets & Configuration Management
- **AWS Secrets Manager** and **SSM Parameter Store** manage runtime secrets and configs.
- Secrets are mounted via IRSA or Kubernetes Secrets encrypted with KMS.
- ConfigMaps and Helm values manage non-sensitive application parameters.

##
## Decision: Disaster Recovery & Continuity
- Backups of EBS volumes and stateful components are taken with **Velero** or AWS Backup.
- ClickHouse and Postgres replicas use snapshotting and multi-AZ replication.
- Runbooks define restoration processes, RTOs, and RPOs for critical services.

---

#
## 5.3 Workload Orchestration

##
## Decision: Pod Autoscaling
- **Horizontal Pod Autoscaler (HPA)** adjusts replicas based on CPU, memory, and custom metrics.
- **Vertical Pod Autoscaler (VPA)** is enabled in staging to monitor and recommend resource tuning.

##
## Decision: Resource Strategy
- define CPU/memory **requests and limits** per container to ensure fair scheduling and avoid noisy neighbors.
- **ResourceQuota** and **LimitRange** enforce team-level boundaries.

##
## Decision: Workload Placement
- **Node affinity/anti-affinity** ensures logical separation (e.g., ingest vs. analytics).
- Taints and tolerations steer long-lived stateful services to specialized nodes.

##
## Decision: Multi-Runtime Management
- Base container images are optimized per language: Go (distroless), Python (slim w/ pip cache), Rust (alpine/musl).
- CI/CD pipeline builds and tests multi-runtime containers in parallel.
- Runtime isolation is not required beyond container boundaries; workload behaviors are managed via metrics and limits.

---

This Kubernetes strategy delivers:
- Scalable, secure, managed infrastructure with minimal overhead  
- Isolation and resource governance across teams and services  
- Resilient operations with mature upgrade, observability, and DR tooling

---


## 6. Data Processing & Computing Layer

#
## 6.1 Microservices Architecture

##
## Decision: Microservice Decomposition
- Odds computation is decomposed by **sport and function**:
  - Ingest Service (per sport)
  - Event Normalization Service
  - Odds Calculation Service
  - Odds Publisher Service
- Each sport has its own pipeline to isolate traffic and domain logic (e.g., NFL has different update frequency and structure vs. MLB).
- Stateless services enable horizontal scaling, circuit breaking, and resilience.

##
## Decision: Service Boundaries
- **Functional decomposition** ensures clear responsibility and team ownership:
  - Ingest: transforms and validates raw feed
  - Normalizer: standardizes formats
  - Calculator: computes new odds
  - Publisher: publishes to Kafka and stores in ClickHouse
- Services can be scaled independently based on workload.

##
## Decision: Service Communication
- Services communicate asynchronously using **Kafka topics**.
- Critical services use **sync fallback (gRPC/HTTP)** for control paths (e.g., health checks or on-demand odds recalculations).
- Inter-service dependency is minimized to reduce cascading failures.

##
## Decision: Data Consistency & Transactions
- **Eventual consistency** is preferred in streaming workflows.
- For stateful data like Redis or Postgres, use **transaction boundaries** scoped to individual services.
- Retry logic and idempotency are key for maintaining correctness under failure.

---

#
## 6.2 Processing Engine Design

##
## Decision: Real-Time Stream Processing
- Each odds calculator service consumes from Kafka, processes updates, and writes results to:
  - Kafka output topic (for downstream consumers)
  - ClickHouse (for analytical storage)
- Kafka ensures **replayability** and **durability**, enabling fault recovery and historical reprocessing.

##
## Decision: Multi-language Pipeline Strategy
- All services are containerized and orchestrated by Kubernetes.
- Services can be implemented in Go (high performance), Python (ML or data heavy), or Rust (low-latency edge logic).
- Common interfaces (protobuf or Avro schemas) ensure interoperability across services.

##
## Decision: Processing Order and Exactly-Once Semantics
- Kafka consumers use **committed offsets** and idempotent writes.
- ClickHouse supports deduplicated inserts via `ReplacingMergeTree` and materialized views.
- Out-of-order events are reconciled using event timestamps, and sequence windows ensure order-sensitive computations.

##
## Decision: Caching and Freshness
- **Redis** is used as a read-through cache for computed odds with short TTL (e.g., 5–10 seconds).
- Cache key is scoped by sport, game ID, and player ID.
- Cache invalidation is event-driven: updates to odds automatically refresh Redis entries.

---

This layer ensures:
- High-performance, horizontally scalable processing  
- Language flexibility with standardized APIs  
- Streamlined fault recovery and replay logic  
- Fresh, accurate, and fas

---


## 7. Data Storage Solutions

#
## 7.1 Operational Data Storage

##
## Decision: PostgreSQL Architecture
- use **Amazon RDS for PostgreSQL** in multi-AZ configuration with automatic failover for high availability.
- Read replicas are provisioned to offload analytical or internal queries from the primary writer.
- **Connection pooling** (e.g., PgBouncer or RDS Proxy) ensures consistent throughput during spikes.

##
## Decision: Sharding and Partitioning
- Data is **partitioned by sport and game_id** to reduce index size and improve query locality.
- Horizontal sharding is deferred unless write throughput or dataset size becomes a bottleneck.
- Future sharding can be implemented using Citus or Postgres FDW.

##
## Decision: Redis Clustering and Persistence
- **Amazon ElastiCache Redis Cluster** is deployed with multi-node replicas and automatic failover.
- In-memory TTL-based caching is used for frequently accessed odds and lookup tables.
- AOF (Append-Only File) persistence is enabled for write durability, though Redis is considered non-source-of-truth.
- Redis metrics and key TTLs are monitored for usage optimization.

##
## Decision: Backup and Recovery
- Postgres snapshots and WAL logs are retained using **RDS automated backups** and manual snapshots before deployments.
- Redis snapshots (RDB) are enabled with daily retention and point-in-time recovery (PITR) via ElastiCache.
- Database schema changes are tracked via Flyway or Liquibase for versioned migrations.

---

#
## 7.2 Analytics Platform Design

##
## Decision: Analytics Platform
- use **ClickHouse** as the primary analytics engine due to its columnar storage, ultra-fast OLAP querying, and compatibility with real-time data ingestion.
- ClickHouse is deployed on dedicated EC2 instances with NVMe and S3 tiered storage for production environments or is managed via Altinity.Cloud for simplified ops.

##
## Decision: Data Structure for Analytics
- Data is organized using **ReplacingMergeTree** and **Materialized Views** for deduplication and real-time rollups.
- Tables are partitioned by date and sport, and sorted by game_id, player_id for query speed.
- Aggregates (e.g., min/max odds, volatility) are precomputed and stored in separate summary tables.

##
## Decision: Lifecycle & Archival
- Hot data (1–7 days) is stored in ClickHouse with fast SSD-backed storage.
- Warm data (up to 30 days) is retained on slower disk or object-backed tiered storage.
- Cold data is offloaded to **Amazon S3** in Parquet format for archival via scheduled ETL jobs.

##
## Decision: Real-Time vs Batch Analytics
- Real-time analytics uses Kafka consumers writing to ClickHouse in near real-time.
- Batch jobs run via scheduled Kubernetes CronJobs or Spark-on-K8s for offline ETL, historical reprocessing, or complex joins.
- Analysts query ClickHouse via JDBC/ODBC or a BI tool like Superset or Metabase.

---

This storage strategy ensures:
- Fast-access operational data with resilient PostgreSQL + Redis design  
- Scalable, real-time analytics with cost-efficient data lifecycle management  
- Ready-to-query historical and real-time odds data for quant research

---


## 8. Message Streaming & Event Architecture

#
## 8.1 Internal Kafka Architecture

##
## Decision: Kafka Broker Design
- use **Amazon MSK (Managed Streaming for Apache Kafka)** for internal event streaming to ensure scalability, high availability, and integration with AWS IAM and networking.
- MSK is deployed across **3 AZs** with automatic replication and failover.
- Kafka is tuned for low latency and high throughput with SSD storage, optimized network throughput, and high partition counts.

##
## Decision: Topic Design and Partitioning
- Topics are created per sport and event type: `nfl.odds`, `nba.ingest`, `mlb.analytics`, etc.
- Partitioning is based on `game_id` or `player_id` to maintain message ordering where necessary.
- Replication factor is set to 3 for HA, with `min.insync.replicas` enforced to protect durability.

##
## Decision: Cluster Management & Monitoring
- Kafka is monitored using **Prometheus + Grafana**, integrated via JMX exporters or MSK metrics.
- Consumer lag, throughput, error rates, and partition distribution are tracked.
- Alerts are configured for:
  - Broker unavailability
  - High lag or ISR shrinkage
  - Topic under-replication or skew

##
## Decision: Schema Management
- **Confluent Schema Registry** or AWS Glue Schema Registry manages Avro/Protobuf schemas.
- Schemas are versioned with compatibility rules (`BACKWARD`, `FULL`) for safe evolution.
- Producers validate messages against schemas before publishing.

---

#
## 8.2 Cross-Account Integration

##
## Decision: Secure Kafka Connectivity
- **PrivateLink** is used to expose MSK across AWS accounts securely, avoiding public endpoints.
- **TLS encryption** and **SASL/IAM auth** ensure end-to-end encrypted and authenticated traffic.
- Kafka ACLs restrict access to specific roles per topic and partition.

##
## Decision: Serialization & Compatibility
- All messages use **Avro or Protobuf** with embedded schema versions.
- Schema Registry is shared (read-only) with external consumers to decode messages.
- Fields are annotated for compatibility to prevent breaking changes across environments.

##
## Decision: Exactly-Once Guarantees
- Kafka producers use `enable.idempotence=true` and transactions API for guaranteed delivery.
- Consumers commit offsets **only after downstream writes (e.g., ClickHouse) are successful**.
- Retry + DLQ patterns are implemented for failures across network or storage systems.

##
## Decision: External Integration Monitoring
- External access latency, delivery lag, and consumer group offsets are monitored.
- Logs of connection failures, auth errors, and retries are sent to CloudWatch or OpenSearch.
- Alerting includes:
  - External consumer dropout
  - Schema deserialization failures
  - Sustained message delivery delay

---

This message streaming design enables:
- High-throughput, low-latency internal pub/sub architecture  
- Reliable and secure delivery to external Kafka consumers  
- Safe schema evolution and observability for all streaming pipelines

---


## 9. Observability & Monitoring

#
## 9.1 Single Pane of Glass Design

##
## Decision: Unified Observability Stack
- deploy the **Grafana/Loki/Prometheus/Tempo** stack for a unified observability plane, integrated with AWS services.
- **Grafana** serves as the main dashboard layer, with integrations for metrics (Prometheus), logs (Loki), and traces (Tempo).
- AWS-native metrics (e.g., CloudWatch) are federated into Grafana via the AWS data source.

##
## Decision: Distributed Tracing
- All services are instrumented with **OpenTelemetry** SDKs to emit traces.
- **Tempo** is used to collect and store traces, enabling request-path visibility across services.
- Traces are linked to logs and metrics for full-stack correlation.

##
## Decision: Log Aggregation & Analysis
- **Loki** aggregates logs from all services using Fluent Bit or Vector agents.
- Logs are structured (JSON) to facilitate search and correlation with request IDs or trace context.
- Application logs, system logs, and audit trails are retained with defined TTLs (e.g., 7–30 days depending on sensitivity).

##
## Decision: Correlation of Signals
- Grafana dashboards use **Explore mode** to pivot from metrics to logs to traces.
- Alert annotations are automatically added to dashboards for incident root cause analysis.
- Unified correlation helps teams triage errors by linking symptoms (latency) to root cause (e.g., Kafka lag or Redis spikes).

---

#
## 9.2 Alerting & SLA Monitoring

##
## Decision: SLA Monitoring
- Latency (P50, P99) and uptime metrics are gathered per service using Prometheus histograms and summaries.
- **SLOs** are defined with error budgets, and alerts are configured to trigger when burn rates exceed thresholds.
- Kafka consumer lag, odds delivery latency, and Redis hit ratio are monitored as part of SLA enforcement.

##
## Decision: Alert Thresholds & Escalation
- Prometheus alert rules trigger alerts based on thresholds (e.g., P99 > 5s for odds delivery).
- Alerts are routed via **Alertmanager** to Slack, PagerDuty, or Opsgenie.
- Escalation policies are tiered (e.g., engineering on-call, then infra leads).

##
## Decision: Business-Level Monitoring
- Business KPIs (e.g., odds update rate, accuracy %, update freshness) are emitted by services and collected in Prometheus.
- Dashboards show rolling aggregates and anomalies (e.g., no odds updates for a live game).
- Error budgets are computed per game or sport to contextualize SLAs.

##
## Decision: Dashboards & Stakeholder Views
- Grafana dashboards are curated for:
  - Engineers: per-service CPU/mem, latency, error rate
  - SREs: system-wide SLA, Kafka lag, cluster health
  - Analysts: odds throughput, freshness lag
- Dashboards are versioned and deployed as code using Jsonnet or Terraform.

---

This observability design ensures:
- Full system visibility from infra to business metrics  
- Fast issue diagnosis with rich correlation across logs, metrics, and traces  
- Proactive SLA enforcement and alerting aligned to real-world game performance

---


## 10. Infrastructure as Code & Automation

#
## 10.1 IaC Tooling and Structure

##
## Decision: Tool Selection
- **Terraform** provides the core language for defining infrastructure.
- **Terragrunt** wraps Terraform to:
  - Reuse module code across accounts and regions
  - Manage dependency graphs and remote state
  - Encapsulate environment-specific configurations
- **Helm** packages Kubernetes manifests for application delivery.
- **ArgoCD** manages GitOps-driven, declarative deployment of Helm-based applications to Kubernetes.

##
## Decision: Terragrunt Directory Structure
adopt a hierarchy based on **AWS account → region → component**, allowing clean separation of environments, cross-account policies, and modular deployment.

```text
live/
├── dev/                  # Dev AWS Account
│   └── us-east-1/
│       ├── vpc/
│       ├── eks/
│       ├── rds/
│       ├── redis/
│       ├── msk/
│       └── clickhouse/
├── staging/              # Staging AWS Account
│   └── us-west-2/
│       ├── vpc/
│       ├── eks/
│       ├── rds/
│       ├── redis/
│       ├── msk/
│       └── clickhouse/
└── prod/                 # Production AWS Account
    └── us-east-1/
        ├── vpc/
        ├── eks/
        ├── rds/
        ├── redis/
        ├── msk/
        └── clickhouse/
```

Each component folder contains a `terragrunt.hcl` file that:
- References reusable infrastructure modules from `infrastructure-modules/`
- Configures remote state (S3 + DynamoDB) and provider details
- Defines dependency blocks (e.g., `eks` depends on `vpc`)

##
## Decision: Module & Backend Strategy
- Modules are stored in a separate `infrastructure-modules/` repo (or folder) and versioned independently.
- Remote state is stored per environment and region:
    - s3://infra-terraform-states/prod/us-east-1/eks/terraform.tfstate
- Locking is managed via DynamoDB tables (`terraform-state-lock-prod`, etc.), one per account.

##
## Decision: Secrets & Config Management
- Secrets (DB passwords, Kafka tokens) are stored in **AWS Secrets Manager** and accessed via IRSA in Kubernetes.
- Application configs (e.g., Kafka topics, Redis TTLs) are passed as `Helm values.yaml` via ArgoCD.

---

#
## 10.2 Deployment Pipelines

##
## Decision: CI/CD Strategy
- **GitHub Actions** is used for both infrastructure and application pipelines:
- Runs `terraform validate`, `plan`, and `terragrunt run-all apply`
- Validates Helm charts (`helm lint`, `template`)
- Publishes Helm charts to **GitHub Container Registry (GHCR)**
- ArgoCD monitors Git for Helm chart version bumps or config changes and syncs them automatically.

##
## Decision: ArgoCD Integration
- Each service has a corresponding `Application` or `ApplicationSet` YAML tracked in Git.
- Charts are deployed declaratively via ArgoCD, using Helm value overlays per environment.
- Rollbacks and diffs are managed natively by ArgoCD, providing Git-based audit trails.

##
## Decision: Deployment Strategy
- Infrastructure updates use **rolling updates** or **blue/green** where supported (e.g., MSK, Redis).
- Applications use **Helm** with health checks and readiness gates to ensure zero-downtime rollout.
- ArgoCD supports automated sync or manual promotion per environment.

##
## Decision: Environment Management
- Each AWS account maps to an environment (`dev`, `staging`, `prod`).
- Separate ArgoCD Projects and Namespaces isolate workloads and permissions.
- Ephemeral or preview environments can be spun up using branch-based ArgoCD + Terragrunt orchestration.

---

This setup provides:
- **Clear separation of environments** by AWS account and region  
- **Reusable, composable modules** for scalable infrastructure  
- **Automated, declarative app delivery** via Helm and ArgoCD  
- **Secure, DRY, and auditable infrastructure workflows** through GitHub Actions and GitOps

---


## 11. Security, Compliance & Access Control

#
## 11.1 Infrastructure Security

##
## Decision: Network Security
- All services are deployed in **private subnets** within AWS VPCs.
- Only public-facing components (e.g., ALBs) reside in public subnets with strict inbound rules.
- **Security Groups and NACLs** enforce least privilege at the network boundary level.
- Inter-service traffic is encrypted using **TLS** within the cluster (mTLS via Istio or Linkerd optional).

##
## Decision: Secrets & Credentials
- Application secrets (e.g., DB passwords, Kafka tokens, API keys) are stored in **AWS Secrets Manager**.
- Kubernetes workloads access secrets using **IRSA (IAM Roles for Service Accounts)** — no static credentials on disk.
- Helm charts reference secrets via environment variables or mounted volumes.
- All secret access is audited and rotated on a defined schedule (e.g., 90-day rotation policy).

##
## Decision: Dependency Isolation
- Critical components (Kafka, RDS, Redis) run in dedicated subnets and security groups with explicit ingress rules.
- K8s workloads are restricted using **network policies**, pod security standards (restricted), and namespaces for isolation.
- RBAC is enabled for internal service-to-service communication (e.g., gRPC).

---

#
## 11.2 Access Control & Identity Management

##
## Decision: AWS IAM Architecture
- IAM is organized by account and team role, following the principle of **least privilege**.
- **Terraform** and **Terragrunt** manage IAM policies and roles centrally per account.
- Read-only and operator roles are granted via **IAM federation** (SSO or IdP integration).

##
## Decision: Kubernetes RBAC
- Access to clusters is governed by **Kubernetes RBAC**, scoped per namespace and role.
- Developers are granted access to specific namespaces only (`dev-team-a` → `namespace-a`).
- Admin privileges are restricted to a small SRE group and audited regularly.

##
## Decision: GitOps Access
- ArgoCD is configured with **SSO integration** and supports role-based access to Projects and Applications.
- All infrastructure and app changes are gated by GitHub PRs and protected branches.
- ArgoCD's audit logs and sync history provide traceability for every deployment.

---

#
## 11.3 Compliance, Auditability & Best Practices

##
## Decision: Auditing & Logging
- All AWS access is logged via **CloudTrail** and aggregated into a centralized log bucket.
- Kubernetes audit logs are enabled and shipped to **Loki** for search and retention.
- ArgoCD sync logs, Helm release histories, and Terraform state changes are versioned and timestamped.

##
## Decision: Compliance Considerations
- Data-at-rest is encrypted using **KMS** for S3, EBS, RDS, and MSK.
- Data-in-transit is encrypted with TLS 1.2+ by default across all services.
- Infrastructure follows **CIS Benchmarks for AWS and Kubernetes**, validated using `kube-bench` and `tfsec`.

##
## Decision: Vulnerability Management
- Container images are scanned with **Trivy** or **Grype** in CI workflows.
- Terraform modules and Helm charts are checked for known CVEs.
- Patch cadence is enforced monthly for base images, dependencies, and OS-level packages.

---

This security architecture ensures:
- Strong isolation between environments and services  
- Auditable, least-privilege access across teams and tools  
- Compliance with modern cloud and Kubernetes security standards

---


## 12. Cost Optimization & Scaling Strategy

#
## 12.1 Compute & Autoscaling Strategy

##
## Decision: Compute Scaling
- **EKS node groups** use a mix of **on-demand and spot instances** to balance reliability and cost.
- **Karpenter** is used for intelligent, workload-aware node provisioning based on pod requirements.
- HPA (Horizontal Pod Autoscaler) and VPA (Vertical Pod Autoscaler) are configured per workload:
  - Odds processors: CPU/memory-based HPA
  - Ingestion gateways: request-per-second and latency-based scaling
- Node groups are right-sized by workload class (compute-intensive, memory-bound, etc.).

##
## Decision: Multi-cluster Strategy
- Lower environments (`dev`, `staging`) use smaller shared clusters or node pools.
- Production runs in isolated, high-availability clusters to reduce blast radius and improve resource isolation.
- Optional GPU/ARM-based node pools can be introduced for specialized workloads (e.g., ML pipelines).

---

#
## 12.2 Storage Optimization

##
## Decision: Tiered Storage
- PostgreSQL uses **GP3 volumes** with autoscaling IOPS and throughput for predictable performance.
- ClickHouse uses **EBS + S3 tiering** (via [ClickHouse's S3-backed storage] or external volumes).
- Logs and metrics are retained on cost-efficient tiers:
  - Short-term in Loki and Prometheus TSDB
  - Long-term archive in **S3 Glacier** for compliance

##
## Decision: Data Lifecycle Management
- TTLs are enforced at the ClickHouse table and Redis key level to avoid unbounded growth.
- S3 lifecycle rules transition analytics data to Glacier after 30–90 days.
- PostgreSQL partitions historical odds data by game/date and uses scheduled cleanup for cold partitions.

---

#
## 12.3 Cost Visibility & Guardrails

##
## Decision: Cost Monitoring
- AWS **Cost Explorer** and **CloudWatch billing metrics** are integrated with Grafana dashboards.
- **Kubecost** runs in each EKS cluster to track workload-level cost, spot vs. on-demand breakdown, and unused resources.
- Alerts are configured for:
  - Budget overruns by team or environment
  - Unscheduled compute spikes
  - Underutilized persistent volumes or IPs

##
## Decision: Reserved & Savings Plans
- Production RDS, MSK, and baseline EC2 workloads use **1-year Reserved Instances or Savings Plans**.
- Spot interruption tolerances are modeled for ingestion and analytics workloads.
- Scheduled scaling is used during off-peak hours to reduce compute spend (e.g., pause non-critical cron jobs).

---

This strategy ensures:
- Efficient autoscaling across environments and services  
- Granular cost attribution for infrastructure and application workloads  
- Sustainable long-term growth without overspending

---


## 13. Risk Mitigation & Disaster Recovery

#
## 13.1 Failure Scenarios & Fault Tolerance

##
## Decision: Availability Strategy
- All production infrastructure is deployed across **multiple Availability Zones (AZs)** for high availability.
- Critical services (Kafka, RDS, Redis, ClickHouse) are deployed in **multi-AZ or clustered configurations**.
- Kubernetes workloads use **PodDisruptionBudgets (PDBs)** and **anti-affinity rules** to ensure redundancy.

##
## Decision: Service Isolation
- Each sport feed (NFL, NBA, MLB) runs in separate **namespaces** or **workload groups** to prevent failure propagation.
- Ingestion and processing pipelines are **loosely coupled via Kafka**, allowing for graceful degradation.
- Circuit breakers and retries with backoff are implemented in inter-service communication (e.g., gRPC or REST calls).

##
## Decision: Stateful Workload Protection
- PostgreSQL is configured for **automated failover** using AWS RDS Multi-AZ or Aurora.
- Kafka brokers are deployed with **3+ replicas per topic partition** for durability and leader election.
- Redis uses **cluster mode with persistence and AOF** for fast recovery and failover.

---

#
## 13.2 Backup & Restore Strategy

##
## Decision: Backup Cadence
- RDS/PostgreSQL: **daily snapshots**, 7–30 day retention, PITR (Point-in-Time Recovery) enabled.
- ClickHouse: incremental backups using **S3 snapshots or backup-manager**.
- Redis: daily backup of RDB snapshots, stored in S3.

##
## Decision: Restore Testing
- Backups are **automatically verified and checksum validated** during off-peak hours.
- Quarterly **disaster recovery drills** simulate full-region failovers and backup restores.
- Application config and secrets are versioned and stored in Git, enabling reproducible bootstrap.

---

#
## 13.3 Disaster Recovery Plan (DRP)

##
## Decision: Recovery Objectives
- **RPO (Recovery Point Objective)**:
  - Odds data: ≤ 5 minutes
  - Ingestion queues: ≤ 1 minute
  - State stores (Redis): ≤ 5 minutes
- **RTO (Recovery Time Objective)**:
  - Partial component outage: < 15 minutes
  - Full AZ outage: < 1 hour
  - Full region failover: ≤ 4 hours (with manual promotion)

##
## Decision: Regional Redundancy
- Cross-region disaster recovery is supported for Kafka, RDS, and object storage via:
  - **MSK MirrorMaker2** for topic replication
  - **RDS Read Replicas** promoted as failover targets
  - **S3 cross-region replication** for logs and backups
- ArgoCD can redeploy infrastructure and workloads to a secondary region using replicated Git state and `terragrunt run-all`.

---

This approach ensures:
- Rapid recovery from failures with tested RTO/RPO targets  
- Data durability and resiliency via multi-AZ and multi-region replication  
- Confidence in operational continuity through routine DR testing

---


## 14. Delivery Summary & Final Recommendations

#
## 14.1 Recap of Architectural Approach

This solution is designed to deliver a highly available, low-latency, and scalable platform for ingesting, processing, and distributing real-time sports odds data across multiple leagues (NFL, NBA, MLB). Key architectural decisions include:

- **Event-driven microservices** architecture built on Kubernetes (EKS)
- **Kafka** as the core messaging backbone with cross-cloud support
- **ClickHouse** for high-performance analytical storage
- **PostgreSQL + Redis** for operational data and caching
- **Helm + ArgoCD** for GitOps-driven, declarative Kubernetes deployments
- **Terraform + Terragrunt** for modular, multi-account infrastructure-as-code
- **Multi-AZ deployments** and tested **disaster recovery strategies**
- Observability via a **unified monitoring stack** (Grafana, Loki, Prometheus, ArgoCD)

---

#
## 14.2 How Requirements Are Met

| Requirement                         | Fulfillment Strategy                                                                 |
|-------------------------------------|----------------------------------------------------------------------------------------|
| **99%+ Uptime (100% during games)** | Multi-AZ architecture, HA services, rolling upgrades with PDBs, and GitOps rollbacks  |
| **P99 < 5s / P50 1–3s latency**     | Kafka streaming, real-time odds processors with autoscaling, Redis + optimized caching|
| **Cross-account Kafka**            | Encrypted MSK with IAM auth and VPC peering or Transit Gateway + MirrorMaker2         |
| **Quant analytics platform**       | ClickHouse with S3-tiered storage and fast queries; ArgoCD-deployed analytical tooling|
| **Real-time ingestion**            | Kafka consumers + optional push/PubSub integration fallback                           |
| **Secure & compliant**             | IAM, IRSA, KMS, encrypted backups, audit logging, CIS benchmarks                      |
| **Cost efficiency**                | Spot instances, Kubecost, S3 lifecycle rules, Savings Plans, VPA/HPA, ClickHouse tiering|
| **Observability**                  | Single pane via Grafana, Prometheus, Loki, ArgoCD; alerts tied to SLAs & budgets      |
| **Fast disaster recovery**         | Multi-AZ + regional backup replication, PITR for RDS, DR drills and automation         |

---

#
## 14.3 Final Recommendations

- **Start small, iterate fast**: Begin with core ingestion → odds → Kafka → analytics pipeline; expand multi-sport support incrementally.
- **Codify all environments**: Use Terragrunt for consistent AWS account segregation and reproducibility.
- **Adopt GitOps early**: Helm + ArgoCD ensures deployment consistency, rapid rollback, and visibility.
- **Run performance tests with simulated traffic**: Validate latency targets under realistic load conditions.
- **Continuously refine DR plan**: Schedule DR tests and publish RPO/RTO metrics per system tier.
- **Treat observability as a product**: Provide dashboards tailored to engineering, SRE, and business teams.
- **Review cost reports monthly**: Use Kubecost and billing exports to manage efficiency and eliminate waste.

---

This approach creates a robust foundation for scaling real-time sports data operations while ensuring reliability, visibility, and velocity across all engineering teams.

---


## Appendices

#
## Appendix A: Technology Decision Matrix
*Comparative analysis of key technology choices*

| Category            | Option Considered          | Final Choice       | Rationale                                                                 |
|---------------------|----------------------------|---------------------|---------------------------------------------------------------------------|
| Kubernetes Platform | GKE, AKS, EKS              | **EKS**             | Deep AWS integration, IAM via IRSA, regional presence, enterprise support |
| IaC Tooling         | Terraform, CDK, Pulumi     | **Terraform + Terragrunt** | Most mature, modular reuse, account + region hierarchy support     |
| App Deployment      | Kustomize, Flux, ArgoCD    | **Helm + ArgoCD**   | GitOps friendly, environment-specific overlays, secure RBAC               |
| Messaging Backbone  | AWS SNS/SQS, NATS, Kafka   | **Kafka (MSK)**     | Durable event streaming, wide ecosystem, MirrorMaker2 for cross-cloud     |
| Analytics Engine    | Redshift Serverless, Druid, BigQuery | **ClickHouse** | Best-in-class OLAP performance and real-time inserts                    |
| Observability       | Cloud-native, Datadog, OpenTelemetry stack | **Prometheus + Grafana + Loki** | Fully open source, Kubernetes-native     |

---

#
## Appendix B: Cost Estimation Model
*Projected monthly costs (USD, approximate)*

| Component              | Environment | Service Tier        | Estimated Cost/Month |
|------------------------|-------------|----------------------|----------------------|
| EKS Cluster            | Prod        | Multi-AZ, autoscaling| $2,000               |
| MSK (Kafka)            | Prod        | 3-broker, 100MBps    | $3,500               |
| RDS PostgreSQL         | Prod        | Multi-AZ, 2x r6g.large| $800                |
| Redis (Elasticache)    | Prod        | Cluster mode enabled | $400                 |
| ClickHouse on dedicated EC2 with NVMe-backed volumes and S3 cold tiering   | Prod        | 3 nodes + storage    | $1,200               |
| S3 Storage (Analytics) | Prod        | 5TB + Glacier tiering| $150                 |
| ArgoCD / Prom / Grafana| Shared      | Containerized        | $200                 |
| Spot Instance Savings  | Shared      | ~40% savings         | -$1,000              |
| **Total**              |             |                      | **~$7,250/month**    |

*Note: Dev and staging environments use reduced compute tiers and shared clusters for cost efficiency.*

---

#
## Appendix C: Risk Assessment & Mitigation

| Risk                            | Impact          | Likelihood | Mitigation Strategy                                            |
|----------------------------------|------------------|-------------|-----------------------------------------------------------------|
| Kafka Consumer Lag              | High             | Medium      | Autoscaling consumers, monitor lag with alerts                  |
| Region Outage                   | Very High        | Low         | Cross-region DR strategy with ArgoCD + replicated data          |
| Data Skew or Inconsistency      | Medium           | Medium      | Validations in enrichment layer, schema enforcement via Avro    |
| Secrets Leakage                 | Critical         | Low         | Use AWS Secrets Manager + IRSA, audit logs enabled              |
| Cost Overrun                    | Medium           | Medium      | Kubecost, monthly budget caps, lifecycle policies               |
| SLA Violation (P99 > 5s)        | High             | Low         | Load testing, HPA/VPA tuning, pod priority + preemption         |
| Toolchain Drift / Drift in IaC  | Medium           | Medium      | Terraform plan reviews, `driftctl`, state locking via DynamoDB  |

---

#
## Appendix D: Implementation Timeline
*6-week phased rollout plan*

| Phase       | Duration   | Milestones                                                                 |
|-------------|------------|----------------------------------------------------------------------------|
| **Phase 1** | Week 1–2   | IaC repo structure, Terragrunt hierarchy, baseline networking (VPC, IAM)   |
| **Phase 2** | Week 3     | EKS cluster setup, ArgoCD bootstrap, GitOps pipeline configured            |
| **Phase 3** | Week 4     | Kafka (MSK) provisioning, odds ingestion prototype                         |
| **Phase 4** | Week 5     | PostgreSQL, Redis, ClickHouse deployed; initial pipeline integration       |
| **Phase 5** | Week 6     | Load testing, DR testing, cost dashboards, full environment readiness      |
| **Post-MVP**| Ongoing    | Incremental support for new sports feeds, deeper analytics capabilities    |

---

This structured approach ensures all architectural, operational, and financial factors are accounted for, while providing a clear delivery roadmap with iterative validation and risk mitigation checkpoints.

---

## AWS Account Structure & Environment Parity

Each environment (dev, staging, prod) is deployed in its own AWS account under a shared AWS Organization. All environments maintain **identical infrastructure topology** — VPCs, EKS clusters, MSK/Kafka, Redis, RDS, and ClickHouse — differing only in compute size, autoscaling thresholds, and data retention. This ensures infrastructure parity across environments, reducing risk during promotions and simplifying operational support.

### AWS Organization Layout

```text
Root (UnderdogFantasy AWS Org)
├── dev-account (dev@...)
│   └── us-east-1 (VPC, EKS, MSK, RDS, Redis, ClickHouse)
├── staging-account (staging@...)
│   └── us-west-2 (VPC, EKS, MSK, RDS, Redis, ClickHouse)
└── prod-account (prod@...)
    └── us-east-1 (VPC, EKS, MSK, RDS, Redis, ClickHouse)
```

---

## IAM Design & Security Boundaries

- **Account Isolation**: Separate AWS accounts ensure strong isolation and blast radius control per environment.
- **Service-level Access Control**:
  - Use IAM Roles for Service Accounts (IRSA) in EKS for scoped pod permissions via OIDC.
  - Terraform restricts resource creation to scoped roles per environment.
- **Cross-Account Access**:
  - Dev has no write access to staging or prod.
  - Terraform assumes environment-specific roles using STS with least privilege and time-bound sessions.
- **Monitoring & Guardrails**:
  - AWS Config, CloudTrail, GuardDuty, and AWS Organizations SCPs enabled across all accounts.
  - SCPs block risky services (e.g., unrestricted IAM policies, public S3 buckets).
- **Secrets Management**:
  - AWS Secrets Manager with KMS encryption stores all sensitive credentials (Kafka auth, RDS passwords, etc.)
  - Access is limited to environment-specific roles and never stored in Git.

---

## Security Scanning Strategy

All container images are scanned in CI using [Trivy](https://github.com/aquasecurity/trivy), with policies that block promotion of builds containing high or critical vulnerabilities. For deeper security workflows, especially in regulated environments, this approach can be extended using commercial offerings such as AquaSec Enterprise or Snyk.

### Tool Comparison Overview

| Feature / Tool                | **Trivy (OSS)**            | **Grype (OSS)**           | **AquaSec Enterprise**       | **Snyk**                         |
|------------------------------|----------------------------|----------------------------|------------------------------|----------------------------------|
| **License**                  | Apache 2.0                 | Apache 2.0                 | Commercial (from AquaSec)    | Commercial (Snyk.io)             |
| **Cost**                     | Free                       | Free                       | $$$ (license-based)          | $$$ (seat-based & usage-tiered)  |
| **Vuln Scanning**            | ✅ OS + libs + IaC          | ✅ OS + libs               | ✅ Advanced policies, CI/CD   | ✅ Deep language scanning         |
| **SBOM Support**             | ✅ SPDX, CycloneDX          | ✅ Syft-native             | ✅ Full audit trail, inventory| ✅ Supports SBOM + dependency graphs |
| **CI/CD Integration**        | ✅ Easy GitHub/GitLab       | ✅ Easy                    | ✅ Centralized scanning       | ✅ Native GitHub/GitLab/Bitbucket |
| **Runtime Protection**       | ❌                         | ❌                         | ✅ (workload sensor + runtime)| ❌ (Snyk doesn’t run on prod)     |
| **SCA (Software Comp. Analysis)** | ✅ Language-aware     | ✅                          | ✅ Advanced, curated feeds    | ✅ Best-in-class (npm, pip, Go, etc.) |
| **Container Image Scanning** | ✅                         | ✅                         | ✅ Real-time scanning         | ✅ Integrated with Docker Hub     |
| **IaC Scanning (Terraform/K8s)** | ✅                      | ❌                         | ✅ (policy-as-code)           | ✅ With Snyk IaC module           |
| **Policy Engine**            | ❌ Basic CLI filters        | ❌                         | ✅ OPA-compatible             | ✅ Org-wide vulnerability gating  |
| **Team Collaboration**       | ❌                         | ❌                         | ✅ User roles, audit trails   | ✅ PR annotations, Jira/Slack integration |

Snyk is particularly effective in developer-centric environments requiring rich language dependency scanning and CI/CD integration. AquaSec is preferred for organizations seeking centralized policy enforcement, auditability, and runtime protection at scale.

---

## Terragrunt Directory Layout

To support strong environment isolation and reusability of infrastructure components, the following directory structure is used:

```text
live/
├── dev/                  # Dev AWS Account
│   └── us-east-1/
│       ├── vpc/
│       ├── eks/
│       ├── rds/
│       ├── redis/
│       ├── msk/
│       └── clickhouse/
├── staging/              # Staging AWS Account
│   └── us-west-2/
│       ├── vpc/
│       ├── eks/
│       ├── rds/
│       ├── redis/
│       ├── msk/
│       └── clickhouse/
└── prod/                 # Production AWS Account
    └── us-east-1/
        ├── vpc/
        ├── eks/
        ├── rds/
        ├── redis/
        ├── msk/
        └── clickhouse/
```

Each module directory (e.g., `vpc/`, `eks/`) contains a `terragrunt.hcl` that calls reusable infrastructure modules.

### Example: `live/prod/us-east-1/eks/terragrunt.hcl`

```hcl
include {
  path = find_in_parent_folders()
}

terraform {
  source = "git::https://github.com/org/terraform-modules.git//eks?ref=v1.2.0"
}

inputs = {
  cluster_name           = "prod-eks"
  region                 = "us-east-1"
  node_groups = {
    general = {
      instance_types = ["m5.large"]
      min_size       = 2
      max_size       = 10
    }
  }
  enable_irsa            = true
  oidc_provider_enabled  = true
}
```

This promotes consistency and enables centralized module updates while maintaining strong environment segmentation across AWS accounts.

---

## AWS Account Structure & Environment Parity

Each environment (dev, staging, prod) is deployed in its own AWS account under a shared AWS Organization. All environments maintain **identical infrastructure topology** — VPCs, EKS clusters, MSK/Kafka, Redis, RDS, and ClickHouse — differing only in compute size, autoscaling thresholds, and data retention. This ensures infrastructure parity across environments, reducing risk during promotions and simplifying operational support.

### Updated AWS Organization Layout

```text
Root (UnderdogFantasy AWS Org)
├── dev-account (dev@...)
│   └── us-east-1 (VPC, EKS, MSK, RDS, Redis, ClickHouse)
├── staging-account (staging@...)
│   └── us-west-2 (VPC, EKS, MSK, RDS, Redis, ClickHouse)
└── prod-account (prod@...)
    └── us-east-1 (VPC, EKS, MSK, RDS, Redis, ClickHouse)
```

---

## Terragrunt Directory Layout

To support strong environment isolation and reusability of infrastructure components, the following directory structure is used:

```text
live/
├── dev/                  # Dev AWS Account
│   └── us-east-1/
│       ├── vpc/
│       ├── eks/
│       ├── rds/
│       ├── redis/
│       ├── msk/
│       └── clickhouse/
├── staging/              # Staging AWS Account
│   └── us-west-2/
│       ├── vpc/
│       ├── eks/
│       ├── rds/
│       ├── redis/
│       ├── msk/
│       └── clickhouse/
└── prod/                 # Production AWS Account
    └── us-east-1/
        ├── vpc/
        ├── eks/
        ├── rds/
        ├── redis/
        ├── msk/
        └── clickhouse/
```

Each module directory (e.g., `vpc/`, `eks/`) contains a `terragrunt.hcl` that calls reusable infrastructure modules.

### Example: `live/prod/us-east-1/eks/terragrunt.hcl`

```hcl
include {
  path = find_in_parent_folders()
}

terraform {
  source = "git::https://github.com/org/terraform-modules.git//eks?ref=v1.2.0"
}

inputs = {
  cluster_name           = "prod-eks"
  region                 = "us-east-1"
  node_groups = {
    general = {
      instance_types = ["m5.large"]
      min_size       = 2
      max_size       = 10
    }
  }
  enable_irsa            = true
  oidc_provider_enabled  = true
}
```