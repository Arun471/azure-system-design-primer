# Azure System Design Primer

A practical reference for designing scalable, reliable, and secure systems on Microsoft Azure. Inspired by the general "system design primer" format, but focused entirely on Azure-native services, patterns, and trade-offs.

---

## Table of Contents

1. [Introduction](#introduction)
2. [Core Azure Compute Options](#core-azure-compute-options)
3. [Storage Services](#storage-services)
4. [Database Services](#database-services)
5. [Networking & Content Delivery](#networking--content-delivery)
6. [Load Balancing in Azure](#load-balancing-in-azure)
7. [Caching Strategies](#caching-strategies)
8. [Messaging & Event-Driven Architecture](#messaging--event-driven-architecture)
9. [Scalability Patterns](#scalability-patterns)
10. [High Availability & Disaster Recovery](#high-availability--disaster-recovery)
11. [Security Best Practices](#security-best-practices)
12. [Monitoring & Observability](#monitoring--observability)
13. [Common Architecture Patterns](#common-architecture-patterns)
14. [Example Walkthrough: Designing a URL Shortener on Azure](#example-walkthrough-designing-a-url-shortener-on-azure)
15. [Cost Optimization Notes](#cost-optimization-notes)
16. [Further Resources](#further-resources)

---

## Introduction

System design on Azure is about choosing the right combination of managed services to meet requirements for **scalability**, **availability**, **latency**, **consistency**, and **cost** — while minimizing operational overhead. Azure's advantage over raw infrastructure is its breadth of managed PaaS offerings, which let teams avoid undifferentiated heavy lifting (patching VMs, managing failover manually, etc.).

Before choosing services, always clarify:

- **Read vs. write ratio** — heavy-read systems favor caching and read replicas; heavy-write systems favor partitioning and queues.
- **Consistency requirements** — strong vs. eventual consistency changes database and caching choices.
- **Latency budget** — global low-latency needs push toward CDN, Front Door, and geo-replication.
- **Availability target (SLA)** — 99.9% vs. 99.99% changes whether you need multi-region active-active.
- **Data volume and growth rate** — determines partitioning/sharding strategy.

---

## Core Azure Compute Options

| Service | Best For | Notes |
|---|---|---|
| **Azure App Service** | Web apps, REST APIs | Fully managed PaaS, built-in autoscale, deployment slots |
| **Azure Functions** | Event-driven, short-lived workloads | Consumption plan scales to zero; good for glue code, webhooks |
| **Azure Kubernetes Service (AKS)** | Microservices, containerized workloads | Full control over orchestration; higher operational complexity |
| **Azure Container Apps** | Microservices without managing Kubernetes | Built on Kubernetes/Dapr under the hood, simpler than AKS |
| **Azure Container Instances (ACI)** | Isolated, on-demand single containers | No orchestration; fast startup, billed per second |
| **Virtual Machines (IaaS)** | Full OS control, legacy apps, custom stacks | Highest operational burden; use Virtual Machine Scale Sets (VMSS) for elasticity |
| **Azure Batch** | Large-scale parallel/batch compute jobs | HPC-style workloads, rendering, simulations |

**Rule of thumb:** Prefer PaaS (App Service, Functions, Container Apps) unless you need low-level control, custom networking, or specific OS/kernel dependencies — then drop to AKS or VMs.

---

## Storage Services

| Service | Data Shape | Use Case |
|---|---|---|
| **Blob Storage** | Unstructured (files, images, video, backups) | Hot/Cool/Archive tiers for cost-tiered retention |
| **Azure Files** | Shared file system (SMB/NFS) | Lift-and-shift apps needing shared drives |
| **Disk Storage** | Block storage for VMs | Managed Disks (Standard HDD/SSD, Premium SSD, Ultra Disk) |
| **Queue Storage** | Simple message queue | Lightweight decoupling; simpler than Service Bus |
| **Table Storage** | NoSQL key-value | Cheap, high-throughput semi-structured data (largely superseded by Cosmos DB Table API) |

**Design tip:** Use Blob Storage lifecycle management policies to auto-tier or delete aging data (e.g., move logs to Cool after 30 days, Archive after 90).

---

## Database Services

### Relational
- **Azure SQL Database** — fully managed SQL Server, built-in HA, auto-tuning, elastic pools for multi-tenant workloads.
- **Azure Database for PostgreSQL / MySQL** — managed OSS relational engines, good for existing OSS-based apps.

### NoSQL
- **Azure Cosmos DB** — globally distributed, multi-model (SQL, MongoDB, Cassandra, Gremlin, Table APIs). Offers tunable consistency levels (Strong, Bounded Staleness, Session, Consistent Prefix, Eventual). Best for low-latency global apps needing horizontal scale.

### Choosing between them

| Requirement | Recommended |
|---|---|
| Strong ACID transactions, complex joins | Azure SQL Database |
| Global multi-region writes, flexible schema | Cosmos DB |
| Existing OSS app (Postgres/MySQL) | Azure Database for PostgreSQL/MySQL |
| Simple key-value at massive scale, low cost | Table Storage or Cosmos DB Table API |
| Analytical/OLAP workloads | Azure Synapse Analytics |

**Partitioning tip:** In Cosmos DB, partition key choice is the single most important design decision — pick a key with high cardinality and even access distribution to avoid "hot partitions."

---

## Networking & Content Delivery

- **Virtual Network (VNet)** — isolated private network; segment with subnets and Network Security Groups (NSGs).
- **Azure Load Balancer** — Layer 4 (TCP/UDP), regional, for VM/VMSS traffic distribution.
- **Application Gateway** — Layer 7, supports SSL termination, URL-based routing, and integrated Web Application Firewall (WAF).
- **Azure Front Door** — global Layer 7 load balancing + CDN + WAF, ideal for multi-region web apps needing lowest-latency routing.
- **Traffic Manager** — DNS-based global routing (no data path involvement); useful for non-HTTP or failover-only scenarios.
- **Azure CDN** — caches static content at edge nodes to reduce latency and origin load.
- **Private Link / Private Endpoint** — exposes PaaS services (e.g., Storage, SQL) privately inside a VNet, avoiding public internet exposure.

**Common pattern:** Front Door (global entry + WAF) → Application Gateway (regional Layer 7 routing) → App Service/AKS backend.

---

## Load Balancing in Azure

| Layer | Service | Scope |
|---|---|---|
| DNS-level | Traffic Manager | Global |
| Global HTTP(S) | Azure Front Door | Global |
| Regional HTTP(S) | Application Gateway | Regional |
| Regional TCP/UDP | Azure Load Balancer | Regional |

Use **health probes** at every tier so unhealthy instances are automatically removed from rotation. Combine Front Door (global failover) with Application Gateway (regional routing/WAF) for defense-in-depth in multi-region designs.

---

## Caching Strategies

- **Azure Cache for Redis** — managed Redis; use for session state, leaderboard/ranking data, rate limiting, and read-through/write-through caching in front of databases.
- **CDN edge caching** — for static assets (images, JS/CSS, video).
- **In-memory/local caching** — within App Service or AKS pods for ultra-low-latency, non-shared data.

**Patterns:**
- *Cache-aside*: app checks cache first, falls back to DB on miss, then populates cache.
- *Write-through*: writes go to cache and DB simultaneously — stronger consistency, higher write latency.
- Always set a **TTL** and have a plan for cache invalidation on data mutation.

---

## Messaging & Event-Driven Architecture

| Service | Pattern | Use Case |
|---|---|---|
| **Azure Queue Storage** | Simple FIFO-ish queue | Lightweight task queues, low cost |
| **Azure Service Bus** | Enterprise messaging (queues + topics/subscriptions) | Ordered delivery, dead-lettering, sessions, transactions |
| **Azure Event Grid** | Reactive pub/sub for discrete events | Serverless event routing (e.g., "blob created" → trigger Function) |
| **Azure Event Hubs** | High-throughput event streaming | Telemetry ingestion, IoT, log/clickstream pipelines (Kafka-compatible) |

**Decision guide:**
- Need guaranteed ordered processing and transactions? → **Service Bus**
- Need to react to discrete state-change events at scale? → **Event Grid**
- Need to ingest millions of events/sec for streaming analytics? → **Event Hubs**
- Need the simplest possible decoupling? → **Queue Storage**

---

## Scalability Patterns

- **Horizontal scaling (scale-out):** Add more instances via VMSS, AKS pod autoscaling, or App Service autoscale rules based on CPU/memory/queue length.
- **Vertical scaling (scale-up):** Increase instance size — simpler but has ceilings and causes downtime on resize (for IaaS).
- **Database read replicas:** Offload read traffic from primary using Azure SQL read replicas or Cosmos DB's multi-region read capability.
- **Sharding/partitioning:** Split data across multiple databases or Cosmos DB logical partitions to scale writes beyond a single node's limits.
- **Asynchronous processing:** Use queues (Service Bus/Storage Queue) to decouple slow operations from the request path, smoothing traffic spikes.
- **CQRS (Command Query Responsibility Segregation):** Separate read and write models/data stores when read and write scaling needs diverge significantly.

---

## High Availability & Disaster Recovery

| Concept | Azure Mechanism |
|---|---|
| **Availability Zones** | Physically separate datacenters within a region; protects against datacenter-level failure |
| **Availability Sets** | Fault/update domains within a single datacenter; protects against rack-level failure |
| **Paired Regions** | Azure pairs regions (e.g., East US ↔ West US) for platform-level DR and staggered updates |
| **Geo-redundant Storage (GRS)** | Asynchronously replicates blob data to a paired region |
| **Azure Site Recovery** | Orchestrates VM-level failover/failback for DR |
| **Cosmos DB multi-region writes** | Active-active global writes with automatic conflict resolution |

**RTO/RPO planning:**
- Define **Recovery Time Objective** (how long can you be down) and **Recovery Point Objective** (how much data loss is acceptable) *before* choosing replication strategy — GRS async replication has a non-zero RPO, while synchronous zone-redundant storage has near-zero RPO but only protects within a region.

---

## Security Best Practices

- **Microsoft Entra ID (formerly Azure AD)** — centralized identity for users and workloads (Managed Identities eliminate stored credentials for service-to-service auth).
- **Azure Key Vault** — store secrets, keys, and certificates; reference via Managed Identity rather than embedding secrets in code/config.
- **Network Security Groups (NSGs)** — subnet/NIC-level firewall rules.
- **Web Application Firewall (WAF)** — on Front Door or Application Gateway, protects against OWASP Top 10 threats.
- **Private Endpoints** — keep PaaS traffic off the public internet.
- **Role-Based Access Control (RBAC)** — least-privilege access at resource, resource group, or subscription scope.
- **Encryption** — at rest (enabled by default on most services) and in transit (enforce TLS 1.2+).

---

## Monitoring & Observability

- **Azure Monitor** — unified platform for metrics and logs across all Azure resources.
- **Application Insights** — APM for distributed tracing, dependency mapping, exception tracking, and live metrics.
- **Log Analytics Workspace** — central query engine (KQL) for aggregating logs across services.
- **Azure Alerts + Action Groups** — trigger notifications, auto-scaling, or remediation runbooks on threshold breaches.

**Design tip:** Instrument distributed tracing (correlation IDs) early — retrofitting observability into a microservices system is far more expensive than building it in from day one.

---

## Common Architecture Patterns

- **Three-tier web app:** Front Door/App Gateway → App Service/AKS → Azure SQL/Cosmos DB, with Redis cache in front of the database.
- **Microservices on AKS:** Services communicate via internal load balancers or a service mesh; Service Bus/Event Grid for async communication; Dapr for cross-cutting concerns (state, pub/sub, bindings).
- **Serverless event pipeline:** Blob upload → Event Grid → Azure Function → Cosmos DB, entirely consumption-based with no idle compute cost.
- **Big data/streaming pipeline:** IoT/app events → Event Hubs → Stream Analytics/Databricks → Synapse Analytics/Data Lake for storage and BI.
- **Global multi-region active-active:** Front Door for global routing → regional App Service/AKS deployments → Cosmos DB multi-region writes for globally consistent low-latency data access.

---

## Example Walkthrough: Designing a URL Shortener on Azure

**Requirements:** High read throughput, low write volume, low latency redirects, must scale to billions of URLs.

1. **API layer:** Azure Functions (HTTP trigger) or App Service — stateless, autoscaling.
2. **Data store:** Cosmos DB — partition key = short code (high cardinality, even distribution); single-digit millisecond reads.
3. **Caching:** Azure Cache for Redis in front of Cosmos DB for hot URLs — most traffic hits a small subset of popular links.
4. **Short code generation:** Base62 encoding of an auto-incrementing counter (via a dedicated counter service) or a hash of the long URL with collision-checking.
5. **Global distribution:** Azure Front Door routes users to the nearest healthy region; Cosmos DB multi-region replication keeps data close to users.
6. **Analytics (click tracking):** Redirect events pushed to Event Hubs asynchronously (non-blocking) → Stream Analytics → Data Lake/Synapse for reporting.
7. **Rate limiting/abuse prevention:** Redis-backed counters at the API layer; WAF rules on Front Door for malicious traffic.

This design keeps the hot path (redirect) extremely fast (cache + Cosmos DB single-partition read) while decoupling everything non-critical (analytics) via async messaging.

---

## Cost Optimization Notes

- Use **Consumption/Serverless tiers** (Functions, Cosmos DB serverless, Container Apps) for spiky or low-traffic workloads.
- Use **Reserved Instances/Savings Plans** for predictable, steady-state compute.
- Apply **Blob lifecycle policies** to auto-tier cold data to Cool/Archive.
- Right-size with **Azure Advisor** recommendations and autoscale rules instead of static over-provisioning.
- Watch **egress bandwidth costs** — data transfer out of Azure (and across regions) is often an overlooked cost driver in multi-region designs.

---

## Further Resources

- [Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/)
- [Azure Well-Architected Framework](https://learn.microsoft.com/en-us/azure/well-architected/)
- [Azure Reference Architectures](https://learn.microsoft.com/en-us/azure/architecture/browse/)
- [Azure Service Comparison Tables](https://learn.microsoft.com/en-us/azure/architecture/guide/technology-choices/technology-choices-overview)

---

*This primer is a starting point for interviews, architecture discussions, or personal study — always validate specific service limits, pricing, and features against current Microsoft documentation, since Azure services evolve frequently.*
