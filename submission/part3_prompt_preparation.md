# Part 3: Prompt Preparation

## Integrity Declaration

"I declare that all written content in this assessment is my own work, created without the use of AI language models or automated writing tools. All technical analysis and documentation reflects my personal understanding and has been written in my own words."

---

## Selected PR

| Field | Value |
|-------|-------|
| Repository | [aio-libs/aiokafka](https://github.com/aio-libs/aiokafka) |
| PR | [#196](https://github.com/aio-libs/aiokafka/pull/196) |
| Title | Added separate socket groups to client |
| Issues | [#137](https://github.com/aio-libs/aiokafka/issues/137), [#128](https://github.com/aio-libs/aiokafka/issues/128) |

---

## 3.1.1 Repository Context

| Aspect | Description |
|--------|-------------|
| **What it does** | Async Kafka client for Python. Produces and consumes messages without blocking the event loop. |
| **Main components** | `AIOKafkaProducer` (publish), `AIOKafkaConsumer` (read), shared client layer for connections |
| **Target users** | Backend devs building microservices, data pipelines, event-driven systems with async frameworks (FastAPI, aiohttp) |
| **Problem it solves** | Traditional Kafka clients block on network I/O. This breaks asyncio's cooperative multitasking. aiokafka gives native async/await support. |

The client layer manages TCP sockets to Kafka brokers, handles metadata discovery, and implements the wire protocol. Consumer groups let multiple app instances share partition workload automatically.

---

## 3.1.2 Pull Request Description

### What Changed

| Before | After |
|--------|-------|
| One socket per broker: `{node_id: conn}` | Two sockets per broker: `{(node_id, group): conn}` |
| Fetches and coordinator share socket | Fetches use `DEFAULT`, coordinator uses `COORDINATION` |
| Slow fetch blocks heartbeats | Each group has independent socket |

### Why It's Needed

Kafka protocol = synchronous per socket. Send request → wait for response → send next.

Problem scenario:
1. Fetch request starts with 500ms timeout
2. Coordinator needs to send heartbeat
3. Heartbeat waits behind fetch
4. Coordinator thinks consumer died → triggers rebalance

This happens when coordinator broker = partition leader broker (same node, shared socket).

### The Fix

Add `ConnectionGroup` enum: `DEFAULT=0`, `COORDINATION=1`. Change dict key from `node_id` to `(node_id, group)`. Coordinator requests go through dedicated socket.

---

## 3.1.3 Acceptance Criteria

| # | Criterion | Condition | Expected Behavior |
|---|-----------|-----------|-------------------|
| 1 | ConnectionGroup enum | Import client module | `ConnectionGroup.DEFAULT=0`, `ConnectionGroup.COORDINATION=1` available |
| 2 | Tuple keys | Check `_conns` dict | Keys are `(node_id, group)` not just `node_id` |
| 3 | Default parameter | Call `_get_conn(node)` without group | Uses `DEFAULT` group automatically |
| 4 | Coordinator routing | Coordinator sends heartbeat/commit | Uses `COORDINATION` group |
| 5 | Coordinator discovery | FindCoordinator request | Uses `DEFAULT` group (it's metadata, not coordination) |
| 6 | Bootstrap format | Initial connection | Stored as `('bootstrap', DEFAULT)` |
| 7 | Independence | One group's connection fails | Other group unaffected |
| 8 | Tests pass | Run test suite | No regressions after updating for tuple keys |

---

## 3.1.4 Edge Cases

| Edge Case | Scenario | Expected Behavior |
|-----------|----------|-------------------|
| **Same node** | Coordinator and partition leader on broker 3 | Two sockets to node 3. Fetch doesn't block heartbeat. |
| **Partial failure** | COORDINATION socket dies, DEFAULT still active | DEFAULT keeps working. Coordinator rediscovers separately. |
| **Concurrent requests** | Two coroutines request connections to same node, different groups | No race condition. Both get their own socket. |
| **Version check** | `check_version()` iterates connections | Only checks `DEFAULT` group (COORDINATION may not exist yet) |
| **Shutdown** | `close()` called with multiple groups active | All connections closed. No orphaned sockets. |

---

## 3.1.5 Initial Prompt

```
Implement connection group separation for aiokafka to prevent coordinator
operations from being blocked by fetch operations.

CONTEXT:
- aiokafka = async Kafka client for Python
- Problem: Kafka protocol is synchronous per socket. Long fetch requests
  block heartbeats when coordinator is on same broker.
- Result: Consumer gets kicked from group due to missed heartbeats.

TASK:
1. Add ConnectionGroup enum to client.py:
   - DEFAULT = 0 (fetch, produce, metadata)
   - COORDINATION = 1 (heartbeat, commit, join, sync, leave)

2. Change connection storage:
   Before: {node_id: connection}
   After:  {(node_id, group): connection}

3. Update methods in client.py:
   - _get_conn(node, group=DEFAULT)
   - ready(node, group=DEFAULT)
   - send(node, request, group=DEFAULT)

4. Update group_coordinator.py:
   - Import ConnectionGroup
   - All coordinator requests use group=COORDINATION
   - Coordinator lookup still uses DEFAULT (it's metadata)

5. Fix bootstrap connection:
   Before: 'bootstrap'
   After:  ('bootstrap', DEFAULT)

6. Update check_version() to filter by DEFAULT group only

FILES TO MODIFY:
- aiokafka/client.py
- aiokafka/group_coordinator.py
- tests/test_client.py
- tests/test_consumer.py

ACCEPTANCE CRITERIA:
- ConnectionGroup enum exists with correct values
- Dict uses tuple keys
- Methods default to DEFAULT
- Coordinator uses COORDINATION for heartbeat/commit/join/sync/leave
- Coordinator discovery uses DEFAULT
- Bootstrap uses tuple format
- Groups are independent (failure isolation)
- All tests pass

EDGE CASES TO HANDLE:
- Coordinator on same node as partition leader
- Connection failure in one group doesn't affect other
- Concurrent connection requests to same node
- Graceful shutdown closes all groups

TEST:
- Update existing tests for tuple-based connection IDs
- Add test: coordination requests not blocked by pending fetches
```

---

## References

- https://github.com/aio-libs/aiokafka
- https://github.com/aio-libs/aiokafka/pull/196
- https://github.com/aio-libs/aiokafka/issues/137
- https://github.com/aio-libs/aiokafka/issues/128
