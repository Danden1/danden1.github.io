---
title: Data Consistency in Distributed Systems
description: Exploring strategies for maintaining data consistency across distributed systems.
date: 2025-11-23 22:37:00 +0900
categories: [Backend, Distributed Systems]
tags: [Data Consistency, Distributed Systems]
math: true
# mermaid: true
---

# Data Consistency in Distributed Systems

I'm a backend developer with 2 years of experience.

At my company, I have used MongoDB, Redis and Kafka.

Most of theses systems write data to a primary(master, leader) node and then replicate it asynchronously to secondary nodes.

If a client reads from a secondary node before the replication is completed, it may read data that is not consistent.

Beause of this, I started studying data consistency.

## Data Consistency in Redis

Redis cluster consist of a master and replica nodes.

```
                 Redis Cluster 

                   ┌─────────────────────────────┐
                   │          Slot Range          │
                   │   (0 ~ 16383 hashed slots)   │
                   └─────────────────────────────┘

    ┌──────────────────┐        ┌─────────────────┐        ┌─────────────────┐
    │   Master A       │        │   Master B       │        │   Master C       │
    │  Slots: 0~5460   │        │ Slots: 5461~10922│        │ Slots:10923~16383│
    └───┬──────────────┘        └───┬──────────────┘        └───┬──────────────┘
        │ (async replication)         │ (async replication)         │ (async replication)
        ▼                             ▼                             ▼
    ┌───────────┐               ┌───────────┐               ┌───────────┐
    │ Replica A1│               │ Replica B1│               │ Replica C1│
    └───────────┘               └───────────┘               └───────────┘


```

[Redis Cluster](https://redis.io/docs/latest/operate/rs/clusters/optimize/wait/)


Redis asynchronously replicates data. 

Redis WAIT is a command that delays the completion of write operation until a specified number of replicas have acknowledged the write, or until a given timeout expires. It is used to reduce replicat lag and imporve data durability.

But It does not provide strong consistency.

[Not guarantee strong consistency](https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/)


## Data Consistency in MongoDB

MongoDB also replicates data asynchronously.

All writes go to the primary node first, and then they are replicated to secondary nodes.

To ensure data consistency, MongoDB provides **Write Concern** and **Read Concern**.


### Write Concern

Write concern determines how many replicas must acknowledge a write before MongoDB considers the operation successful.

### Read Concern

Read concern specifies the level of consistency guaranteed when reading data.


For example:

- majority: reads data that has been written to a majority of nodes.
(Other levels are also available.)

## Data Consistency in Kafka

Kafka uses a leader - follower architecture for each partition.

All writes to go the leader partition, and the follower replicas pull data from the leader to stay synchronized. Followers periodically fetch the leaders' log and append the new records to their own local log.

To ensure data consistency, Kafka provides Acks and ISR.

### ISR(In-Sync Replica)

The ISR consists of the leader and all follower replicas that are sufficiently caught up with the leader’s log.
If a follower falls too far behind, Kafka removes it from the ISR.

### Acks
- acks=0: The producer does not wait for any acknowledgment.
- acks=1: The leader acknowledges the write after writing the record. Follwoers may not have replicated it yet.
- acks=all(-1): The leader waits for all ISR to replicate the write before acknowledging.




## Conclusion

I studied how these distributed systems (MongoDB, Redis, Kafka) ensure data consistency. 

I believe they use fundamentally similar logic to ensure data consistency.

This exploration confirmed the **profound importance of Computer Science (CS) fundamentals**. 

**This experience showed me that by diving deep into one core concept, I can immediately see its application in seemingly different technologies.**
