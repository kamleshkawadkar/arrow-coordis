# ArrowCoordis

### Control Plane for Immutable Arrow Batches

ArrowCoordis is an Arrow-native coordination layer for ephemeral in-memory data batches.

It separates memory ownership from data access, allowing multiple analytical and AI systems to consume the same Arrow RecordBatch through governed leases without owning the underlying buffers.

ArrowCoordis is not a database, message queue, workflow engine, or query engine.
*Its sole responsibility is coordinating immutable Arrow batches across distributed systems.*

---

## Why ArrowCoordis?

Modern data and AI pipelines repeatedly serialize, copy, persist, and reconstruct data as it moves between systems.

A typical workflow looks like:

    Spark -> Kafka -> Python Service -> Feature Store -> Model Server -> Analytics

Each handoff introduces:

- Serialization overhead
- Memory duplication
- Schema conversion
- Additional latency
- Operational complexity

ArrowCoordis enables a different model:

                Arrow RecordBatch
                        │
                        ▼

                     Coordis

         ┌────────┬────────┬────────┐
         ▼        ▼        ▼        ▼

      Analytics   ML      Audit   Metrics

One immutable Arrow batch.

Multiple governed consumers.

No unnecessary data duplication.

---

## Core Principles

### Immutable Batches

Published batches are immutable.

Publish Once. Read Many.
      
      Transform → New Batch

Consumers never modify existing batches.

---

### Ownership ≠ Access

ArrowCoordis separates physical memory ownership from data access.

Batch Node    
          
    owns memory

Consumers
    
    obtain leases

This separation enables safe multi-consumer access while maintaining clear lifecycle management.

---

### Lease-Based Access

Consumers access data through leases rather than direct ownership.

Supported lease types:

BORROWED_READ
    
      Temporary read access with TTL

SHARED_READ
    
      Concurrent read access for multiple consumers

EXCLUSIVE_READ
    
      Optional exclusive access policy

---

### Governed Views

Consumers do not necessarily receive the entire batch.

The coordinator can enforce:

- Column projection
- PII masking
- Role-based views

Example:

Original Batch

      customer_id
      name
      email
      income
      risk_score
      city

Analytics View:

      customer_id
      risk_score
      city

Audit View:

      customer_id
      name
      email
      income
      risk_score
      city

All views operate on the same underlying batch.

---

### Control Plane / Data Plane Separation

Control Plane:

- Batch Registry
- Lease Management
- Routing
- TTL Tracking
- View Policies

Data Plane:

- Arrow Buffers
- Arrow Flight Transport
- Memory Ownership
- Batch Serving

The coordinator never owns Arrow payloads.

---

## Architecture Overview

                     +----------------------+
                     |     Coordinator      |
                     |----------------------|
                     | Batch Registry       |
                     | Lease Manager        |
                     | Routing Service      |
                     | View Policies        |
                     +----------+-----------+
                                |
                                |
                                ▼

      -------------------------------------------------
      |                     |                         |
      ▼                     ▼                         ▼
    +----------------+ +----------------+ +----------------+
    | Batch Node A   | | Batch Node B   | | Batch Node C   |
    |----------------| |----------------| |----------------|
    | Arrow Buffers  | | Arrow Buffers  | | Arrow Buffers  |
    | Flight Server  | | Flight Server  | | Flight Server  |
    | TTL Cleanup    | | TTL Cleanup    | | TTL Cleanup    |
    +----------------+ +----------------+ +----------------+

        ▲                    ▲                    ▲
        │                    │                    │

    Producers           Consumers          AI Systems

---

## Example

Publishing a batch:

``` Java 
BatchId batchId = client.publish(recordBatch);
```

Borrowing a batch:

```Java
Lease lease = client.borrow(batchId);

ArrowView view = lease.view();

process(view);

lease.release();
```

The consumer never owns the underlying Arrow buffers.

---

## Use Cases

### Analytics Query Routing

A producer publishes an Arrow batch.

The coordinator grants a SHARED_READ lease and routes consumers to the appropriate batch node.

---

### ML Inference

Feature batches are published once.

Inference services obtain BORROWED_READ leases and perform predictions without duplicating data.

---

### Sensitive Audit Views

The coordinator applies projection and masking policies.

Auditors receive restricted views while sharing the same underlying batch.

---

### Multi-Consumer Dataflows

One Arrow batch can simultaneously serve:

- Analytics
- ML Inference
- Audit
- Metrics
- Agent Systems

without duplicate ingestion or repeated serialization.

---

## Non-Goals

ArrowCoordis is not:

- A database
- A data warehouse
- A data lake
- A message queue
- A workflow engine
- A query engine
- A distributed compute runtime
- A storage platform

ArrowCoordis focuses exclusively on coordinating immutable Arrow batches.

---

## MVP Roadmap

### MVP 1

Single-node publish and consume flow.

Features:

- Batch Registry
- Publish API
- Consume API
- Arrow Flight Integration

Status: Planned 

---

### MVP 2

Lease enforcement and governed views.

Features:

- BORROWED_READ
- SHARED_READ
- TTL Management
- Sensitive View Projection
- Routing

Status: Planned 

---

### MVP 3

Distributed Fabric Cluster.

Features:

- Multiple Batch Nodes
- Distributed Routing
- Node Heartbeats
- Batch Location Tracking
- Distributed Lease Coordination

Status: Planned 

---

### MVP 4

High Availability and Read Scaling.

Features:

- Read Replicas
- Replica-Aware Routing
- Coordinator Clustering
- Metadata Replication

Status: Future 

---

## Project Status

ArrowCoordis [Coordis] is currently in the architecture and design phase.

The primary goal of the project is to explore a new abstraction:

> Shared immutable Arrow batches coordinated through ownership, leases, and governed access.

This project is intended as both a practical infrastructure prototype and an exploration of Arrow-native dataflow coordination.

---

## Documentation

Additional documentation:

docs/

VISION.md

ARCHITECTURE.md

MEMORY_MODEL.md

LEASE_MODEL.md

BATCH_LIFECYCLE.md

ROADMAP.md

---

## License

Apache 2.0
