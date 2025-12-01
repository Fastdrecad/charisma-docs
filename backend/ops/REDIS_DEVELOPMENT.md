## Redis Development Guide

**Purpose:** Guide for working with Redis in the development environment.  
**Use cases:** Debugging, clearing queues, inspecting jobs.

---

## 1. Connecting to Redis

### 1.1 Option 1: Using Docker (recommended)

If you're using Docker Compose, Redis is running in a container:

```bash
# Connect to Redis container
docker compose exec cache redis-cli

# Or if using docker-compose (older version)
docker-compose exec cache redis-cli
```

### 1.2 Option 2: Direct connection (if Redis is running locally)

```bash
# Connect to local Redis
redis-cli

# Or with specific host/port
redis-cli -h localhost -p 6379
```

### 1.3 Option 3: Using `REDIS_URL` from environment

```bash
# If using Upstash or remote Redis
redis-cli -u $REDIS_URL
```

---

---

## 2. Common Redis Commands for BullMQ

### 2.1 List keys (find queue keys)

```bash
# List all keys (might be many)
KEYS *

# List only BullMQ queue keys
KEYS bull:sessions:*

# List specific queue keys
KEYS bull:sessions:waiting
KEYS bull:sessions:active
KEYS bull:sessions:completed
KEYS bull:sessions:failed
```

### 2.2 Inspect queue jobs

```bash
# Get waiting jobs count
LLEN bull:sessions:waiting

# Get active jobs count
LLEN bull:sessions:active

# Get completed jobs count
ZCARD bull:sessions:completed

# Get failed jobs count
ZCARD bull:sessions:failed

# Get all job IDs in waiting queue
LRANGE bull:sessions:waiting 0 -1

# Get all job IDs in active queue
LRANGE bull:sessions:active 0 -1
```

### 2.3 Inspect a specific job

```bash
# Get job data (replace JOB_ID with actual job ID)
HGETALL bull:sessions:JOB_ID

# Get job state
HGET bull:sessions:JOB_ID state

# Get job data
HGET bull:sessions:JOB_ID data
```

### 2.4 Clear queue jobs

#### 2.4.1 Clear all waiting jobs

```bash
# Delete waiting queue
DEL bull:sessions:waiting

# Or clear all waiting jobs
LTRIM bull:sessions:waiting 1 0
```

#### 2.4.2 Clear all active jobs

```bash
# Delete active queue
DEL bull:sessions:active
```

#### 2.4.3 Clear all completed jobs

```bash
# Delete completed jobs sorted set
DEL bull:sessions:completed
```

#### 2.4.4 Clear all failed jobs

```bash
# Delete failed jobs sorted set
DEL bull:sessions:failed
```

#### 2.4.5 Clear all queue data (use with caution)

```bash
# Delete all BullMQ keys for sessions queue
KEYS bull:sessions:* | xargs redis-cli DEL

# Or step by step:
DEL bull:sessions:waiting
DEL bull:sessions:active
DEL bull:sessions:completed
DEL bull:sessions:failed
DEL bull:sessions:meta
DEL bull:sessions:events
DEL bull:sessions:stalled-check
```

### 2.5 Remove a specific job

```bash
# Remove job from waiting queue
LREM bull:sessions:waiting 1 JOB_ID

# Remove job data
DEL bull:sessions:JOB_ID

# Remove job from completed set
ZREM bull:sessions:completed JOB_ID

# Remove job from failed set
ZREM bull:sessions:failed JOB_ID
```

---

## 3. Quick Cleanup Script

Example helper script to clear the sessions queue:

```bash
# backend/scripts/clear-sessions-queue.sh
#!/bin/bash

echo "Clearing sessions queue..."

# Connect to Redis and clear all queue keys
redis-cli <<EOF
KEYS bull:sessions:* | xargs DEL
EOF

echo "✅ Sessions queue cleared!"
```

Make it executable:

```bash
chmod +x backend/scripts/clear-sessions-queue.sh
```

Usage:

```bash
./backend/scripts/clear-sessions-queue.sh
```

---

## 4. Debugging Job Issues

### 4.1 Check if a job exists

```bash
# Check if job exists
EXISTS bull:sessions:JOB_ID

# Get job state
HGET bull:sessions:JOB_ID state
```

### 4.2 Check queue status

```bash
# Get queue metadata
HGETALL bull:sessions:meta

# Get queue stats
INFO keyspace
```

### 4.3 Monitor queue in real time

```bash
# Monitor all Redis commands
MONITOR

# Watch specific keys
WATCH bull:sessions:waiting
```

---

## 5. Useful Redis Commands

### 5.1 General info

```bash
# Get Redis info
INFO

# Get memory info
INFO memory

# Get all keys count
DBSIZE

# Flush all data (⚠️ DANGEROUS - deletes everything!)
FLUSHALL

# Flush current database only
FLUSHDB
```

### 5.2 Pattern matching

```bash
# Count keys matching pattern
KEYS bull:sessions:* | wc -l

# Get all keys matching pattern
KEYS bull:sessions:*
```

---

## 6. Using BullMQ / Redis from Node.js

You can also create a Node.js script to interact with Redis and BullMQ:

```typescript
// backend/scripts/clear-queue.ts
import { getRedisConnection } from '../src/core/queue/connection.js';
import { createSessionsQueue } from '../src/modules/sessions/jobs/queue.js';

async function clearQueue() {
  const queue = createSessionsQueue();

  // Get all jobs
  const waiting = await queue.getWaiting();
  const active = await queue.getActive();
  const completed = await queue.getCompleted();
  const failed = await queue.getFailed();

  console.log(`Waiting: ${waiting.length}`);
  console.log(`Active: ${active.length}`);
  console.log(`Completed: ${completed.length}`);
  console.log(`Failed: ${failed.length}`);

  // Clean all jobs
  await queue.obliterate({ force: true });

  console.log('✅ Queue cleared!');

  await queue.close();
  process.exit(0);
}

clearQueue().catch(console.error);
```

---

## 7. Important Notes

1. Never use `FLUSHALL` in production – it deletes all Redis data.
2. Be careful with `DEL` – ensure you are deleting the intended keys.
3. BullMQ uses specific key patterns – use `bull:QUEUE_NAME:*` when working with keys.
4. Active jobs – if you delete active jobs, workers may still be processing them.
5. Completed/failed jobs – stored in sorted sets; use `ZREM` or BullMQ APIs to remove them.

---

## 8. Common Scenarios

### 8.1 Clear all old test jobs

```bash
redis-cli
> KEYS bull:sessions:* | xargs DEL
> exit
```

### 8.2 Check what jobs are waiting

```bash
redis-cli
> LRANGE bull:sessions:waiting 0 -1
> exit
```

### 8.3 Remove a specific failed job

```bash
redis-cli
> ZREM bull:sessions:failed JOB_ID
> DEL bull:sessions:JOB_ID
> exit
```

---

## 9. References

- [BullMQ Documentation](https://docs.bullmq.io/)
- [Redis CLI Commands](https://redis.io/commands/)
- [BullMQ Key Patterns](https://docs.bullmq.io/patterns/redis-keys)
