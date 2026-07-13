# Distributed Data Access

```mermaid
mindmap
  root((Chapter 10<br>Distributed Data Access))

    Problem
      Monolith
        Single Database
        SQL Join
        Strong Consistency
      Microservices
        Database per Service
        Data Ownership
        Cross-Service Query
      Core Challenge
        Service Boundary Easy
        Data Access Hard

    Tradeoffs
      Consistency
      Availability
      Scalability
      Performance
      Simplicity
      Coupling

    Pattern 1
      Interservice Communication
        REST
        gRPC
        GraphQL

      Advantages
        Real-Time Data
        Strong Freshness
        Simple To Implement

      Disadvantages
        Network Latency
        Request Fan-Out
        N Plus 1 Problem
        Cascading Failure
        Tight Runtime Dependency

      Suitable For
        Balance
        Inventory
        Payment Status
        Real-Time Information

    Pattern 2
      Column Schema Replication

      Concept
        Replicate Needed Fields
        Local Storage
        Event Synchronization

      Advantages
        Fast Query
        No Network Call
        High Availability

      Disadvantages
        Data Duplication
        Sync Complexity
        Stale Data Risk

      Suitable For
        Product Description
        User Nickname
        Store Information
        Reporting Data

    Pattern 3
      Replicated Caching

      Concept
        Distributed Cache
        Data Shared In Memory
        Local Access

      Technologies
        Hazelcast
        Ignite
        Redis Cluster

      Advantages
        Extremely Fast
        High Availability
        Excellent Scalability
        Reduced Service Dependencies

      Disadvantages
        Memory Consumption
        Cache Invalidation
        Eventual Consistency

      Best Fit
        High Read
        Low Update
        Large Traffic Systems

    Pattern 4
      Data Domain Pattern

      Concept
        Shared Database Domain
        Multiple Services Access Same Data

      Advantages
        Strong Consistency
        Easy Join
        ACID Transactions

      Disadvantages
        High Coupling
        Lost Service Autonomy
        Central Bottleneck

      Suitable For
        Legacy Systems
        Transitional Architecture

    Sysops Squad Example

      Requirement
        Wishlist Needs Product Description

      Options Evaluated
        Interservice Call
        Schema Replication
        Shared Domain
        Replicated Cache

      Final Choice
        Replicated Cache

      Reasons
        Product Changes Rarely
        Reads Very Frequent
        Performance Critical
        Eventual Consistency Acceptable

    Architecture Principles

      No Free Lunch

      If Strong Consistency
        Less Scalability
        Less Availability

      If High Performance
        Data Replication
        Eventual Consistency

      If Low Coupling
        More Infrastructure
        More Synchronization

    Practical Guidance

      Read Heavy
        Cache
        Replication

      Write Heavy
        Service Ownership

      Real Time
        Direct Service Call

      Enterprise Integration
        Event Driven Architecture
        CDC
        Kafka

    RCA Platform Mapping

      Bad Design
        RCA Engine
          Calls Prometheus
          Calls SkyWalking
          Calls Dynatrace
          Calls SAP
          Calls MySQL

      Problems
        Fan-Out
        High Latency
        Cascading Failure

      Better Design
        Plugin Collectors
          Metrics
          Traces
          Logs
          Topology

        Unified Data Layer
          Redis
          Hazelcast
          Kafka Cache

        RCA Engine
          Local Query
          Fast Analysis
          High Reliability
```

             Distributed Data Access
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
   Strong Consistency   High Performance   Low Coupling
        │                  │                  │
        ▼                  ▼                  ▼
 Interservice Call   Replicated Cache    Event Driven
 Shared Domain       Schema Replica      Data Sync
        │                  │                  │
   Fresh Data         Eventually         More Infra
                      Consistent
