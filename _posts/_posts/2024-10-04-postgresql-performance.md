---
layout: post
title: PostgreSQL - Checkpointing and Write-Ahead Logging (WAL)
tags: [postgresql]
cover-img: /assets/img/postgresql.md.png
author: Shlok
---

## Table of Contents

1. [Introduction to Write-Ahead Logging](#introduction-to-write-ahead-logging)
2. [WAL Internals](#wal-internals)
    - [WAL Records and Segments](#wal-records-and-segments)
    - [WAL Buffers](#wal-buffers)
    - [The WAL Writer Process](#the-wal-writer-process)
3. [Understanding Checkpointing](#understanding-checkpointing)
    - [Checkpoint Process](#checkpoint-process)
    - [Impact on Performance](#impact-on-performance)
4. [Configuring WAL and Checkpoints](#configuring-wal-and-checkpoints)
    - [Key Parameters](#key-parameters)
    - [Balancing Durability and Performance](#balancing-durability-and-performance)
5. [WAL Archiving and Recovery](#wal-archiving-and-recovery)
    - [Continuous Archiving](#continuous-archiving)
    - [Point-in-Time Recovery (PITR)](#point-in-time-recovery-pitr)
6. [Best Practices](#best-practices)
    - [Monitoring and Maintenance](#monitoring-and-maintenance)
    - [Avoiding Common Pitfalls](#avoiding-common-pitfalls)
7. [Conclusion](#conclusion)

---

## Introduction to Write-Ahead Logging

Write-Ahead Logging (WAL) is a fundamental feature of PostgreSQL that ensures data integrity and durability. The basic idea is simple: before any changes are made to the actual data files (heap files), those changes are first recorded in a log. This allows PostgreSQL to recover from crashes by replaying the log entries.

WAL provides several benefits:

- **Crash Recovery**: If the server crashes, WAL ensures that all committed transactions are restored.
- **Replication**: Streaming replication uses WAL to keep standby servers in sync.
- **Backup and Restore**: Point-in-time recovery relies on WAL files.

---

## WAL Internals

### WAL Records and Segments

WAL operates by writing **WAL records** to **WAL segments**:

- **WAL Records**: Each record describes a change to the database, such as an INSERT, UPDATE, or DELETE operation.
- **WAL Segments**: Files where WAL records are stored. By default, each segment is 16MB in size and named in a specific sequence.

The WAL files are located in the `pg_wal` directory (named `pg_xlog` in versions prior to PostgreSQL 10).

### WAL Buffers

To optimize disk I/O, WAL uses an in-memory buffer called **WAL buffers**:

- **Shared Memory Segment**: Configured via `wal_buffers`, defaulting to a value computed based on `shared_buffers`.
- **Flush Mechanism**: WAL buffers are periodically flushed to disk, either when full or when a transaction commits.

### The WAL Writer Process

PostgreSQL uses a background process called the **WAL writer** to manage WAL buffers:

- **Purpose**: Continuously flushes WAL buffers to disk to reduce latency during commits.
- **Configuration**: The frequency of writes can be influenced by `wal_writer_delay`.

---

## Understanding Checkpointing

### Checkpoint Process

A **checkpoint** is a crucial event in PostgreSQL where all dirty pages (modified data) in shared memory are written to disk. This process ensures that the number of WAL records that need to be replayed during recovery is limited.

**How it works:**

1. **Dirty Buffer Flush**: All modified buffers in shared memory are written to the data files.
2. **WAL Record**: A special checkpoint record is written to the WAL.
3. **Update Control File**: The control file (`pg_control`) is updated with the location of the checkpoint.

### Impact on Performance

While checkpoints are essential, they can impact performance:

- **I/O Spikes**: Writing a large number of dirty pages can cause a sudden increase in disk I/O.
- **Latency**: Applications may experience higher latency during checkpoints due to resource contention.

---

## Configuring WAL and Checkpoints

### Key Parameters

Proper configuration can mitigate performance issues and optimize behavior.

#### `checkpoint_timeout`

- **Description**: Maximum time between automatic checkpoints.
- **Default**: 5 minutes.
- **Consideration**: Longer intervals reduce the frequency of checkpoints but increase recovery time.

#### `checkpoint_completion_target`

- **Description**: Spreads out the checkpoint writes over a specified fraction of the checkpoint interval.
- **Default**: 0.5 (50% of the interval).
- **Recommendation**: Set closer to 0.9 to smooth out I/O over a longer period.

#### `max_wal_size` and `min_wal_size`

- **Description**: Controls the size of WAL files to retain.
- **Impact**: Larger `max_wal_size` can delay checkpoints triggered by WAL size.

#### `wal_buffers`

- **Description**: Size of the WAL buffers in shared memory.
- **Default**: Typically adequate, but can be increased for write-heavy workloads.

### Balancing Durability and Performance

Some settings allow you to trade off durability for performance:

#### `synchronous_commit`

- **Options**:
  - `on`: Wait for WAL to be flushed to disk on commit.
  - `off`: Transactions may be lost in a crash but improve performance.
- **Use Case**: Set to `off` for less critical data or when replication provides sufficient safety.

#### `commit_delay` and `commit_siblings`

- **Purpose**: Delays committing transactions to group multiple commits, reducing disk writes.
- **Usage**: Requires careful tuning; may not be beneficial for all workloads.

---

## WAL Archiving and Recovery

### Continuous Archiving

For robust backup strategies, WAL files can be archived:

- **Enable Archiving**: Set `archive_mode = on` and configure `archive_command`.
- **Archive Command**: Should copy WAL files to a safe location, e.g.,

  ```shell
  archive_command = 'cp %p /mnt/server/archivedir/%f'
  ```

### Point-in-Time Recovery (PITR)

PITR allows restoring the database to a specific moment:

- **Procedure**:
  1. Restore base backup.
  2. Configure recovery parameters (`recovery.conf` in versions prior to 12, `postgresql.conf` in 12+).
  3. Provide archived WAL files.
- **Recovery Target**: Can be a timestamp, transaction ID, or named restore point.

---

## Best Practices

### Monitoring and Maintenance

- **Use Monitoring Tools**: Track checkpoint frequency, WAL generation rate, and I/O patterns.
- **Logs**: Enable `log_checkpoints` to get detailed information in logs.

### Avoiding Common Pitfalls

- **Too Frequent Checkpoints**: Can cause excessive I/O. Monitor and adjust `max_wal_size`.
- **Long Recovery Times**: If checkpoints are too infrequent, recovery after a crash may take longer.
- **Archiving Failures**: Ensure `archive_command` is reliable; failures can halt WAL recycling.


*If you have experiences or tips related to PostgreSQL's WAL and checkpointing mechanisms, feel free to share them in the comments below!*
