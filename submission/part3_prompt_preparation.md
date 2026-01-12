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
| **What it does** | Its an async Kafka client for Python. Lets you produce and consume messages without blocking your event loop. |
| **Main components** | `AIOKafkaProducer` for publishing messages, `AIOKafkaConsumer` for reading them, plus a shared client layer that handles all the connection stuff |
| **Target users** | Backend developers working on microservices, data pipelines, that kind of thing. People using async frameworks like FastAPI or aiohttp who need Kafka. |
| **Problem it solves** | Regular Kafka clients block on network I/O which totally breaks asyncio's model. This gives you proper async/await support. |

The client layer is what manages TCP sockets to the Kafka brokers, figures out which broker has which partition, and implements the wire protocol. Consumer groups are pretty handy - they let you run multiple instances of your app and the partitions get shared automatically.

---

## 3.1.2 Pull Request Description

### What Changed

| Before | After |
|--------|-------|
| One socket per broker: `{node_id: conn}` | Two sockets per broker: `{(node_id, group): conn}` |
| Fetches and coordinator shared the same socket | Fetches go through `DEFAULT`, coordinator uses `COORDINATION` |
| Slow fetch could block heartbeats | Each group gets its own independent socket |

### Why It's Needed

Heres the thing about Kafka - its synchronous per socket. You send a request, wait for the response, then you can send the next one.

So picture this:
1. Fetch request starts, has a 500ms timeout
2. Meanwhile coordinator needs to send a heartbeat
3. Heartbeat sits there waiting behind the fetch
4. Broker thinks consumer is dead because no heartbeat
5. Triggers rebalance, everything breaks

This happens specifically when the coordinator broker is the same as your partition leader (same node, shared socket).

### The Fix

Add a `ConnectionGroup` enum with two values: `DEFAULT=0` and `COORDINATION=1`. Change the dictionary key from just `node_id` to `(node_id, group)`. Now coordinator requests get their own socket and dont have to wait.

---

## 3.1.3 Acceptance Criteria

| # | Criterion | Condition | Expected Behavior |
|---|-----------|-----------|-------------------|
| 1 | ConnectionGroup enum exists | Import client module | Should have `ConnectionGroup.DEFAULT=0` and `ConnectionGroup.COORDINATION=1` |
| 2 | Tuple keys in use | Look at `_conns` dict | Keys should be `(node_id, group)` tuples not plain `node_id` |
| 3 | Default parameter works | Call `_get_conn(node)` without specifying group | Should use `DEFAULT` group |
| 4 | Coordinator routing correct | Coordinator sends heartbeat or commit | Must use `COORDINATION` group |
| 5 | Coordinator discovery | FindCoordinator request | Uses `DEFAULT` because its really just metadata |
| 6 | Bootstrap format | Initial connection setup | Should be stored as `('bootstrap', DEFAULT)` |
| 7 | Groups are independent | One group's connection fails | Other group should keep working fine |
| 8 | Tests still pass | Run the test suite | No regressions after updating for tuple keys |

---

## 3.1.4 Edge Cases

| Edge Case | Scenario | Expected Behavior |
|-----------|----------|-------------------|
| **Same node situation** | Coordinator and partition leader both on broker 3 | Should have two separate sockets to node 3. Fetch shouldn't block heartbeat. |
| **Partial failure** | COORDINATION socket dies but DEFAULT still working | DEFAULT keeps doing its thing. Coordinator rediscovers on its own. |
| **Concurrent requests** | Two coroutines both want connections to same node but different groups | No race condition. Each gets their own socket. |
| **Version check** | `check_version()` iterates through connections | Only checks `DEFAULT` group since COORDINATION might not exist yet |
| **Shutdown** | `close()` called when multiple groups are active | All connections get closed. No sockets left hanging. |

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
