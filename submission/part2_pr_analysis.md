# Part 2: Pull Request Analysis

## Integrity Declaration

"I declare that all written content in this assessment is my own work, created without the use of AI language models or automated writing tools. All technical analysis and documentation reflects my personal understanding and has been written in my own words."

---

## Selected PRs Overview

| PR | Repository | Title | Author | Merged | Commits |
|----|------------|-------|--------|--------|---------|
| [#196](https://github.com/aio-libs/aiokafka/pull/196) | aiokafka | Added separate socket groups to client | tvoinarovskyi | July 31, 2017 | 2 |
| [#3478](https://github.com/beetbox/beets/pull/3478) | beets | Implement parallel replaygain analysis | ybnd | Dec 14, 2020 | 20 |

---

## PR 1: aiokafka #196 — Separate Socket Groups

### The Problem

Kafka protocol is synchronous per socket. One connection = one request at a time. The client had only one socket per broker. When a fetch request with 500ms timeout was running, everything else (heartbeats, commits) had to wait.

Worst case: coordinator lives on same broker as partition leader. Heartbeats get blocked → coordinator thinks consumer is dead → triggers rebalance → chaos.

Related issues: [#137](https://github.com/aio-libs/aiokafka/issues/137), [#128](https://github.com/aio-libs/aiokafka/issues/128)

### The Solution

Split connections into groups. Fetch uses `DEFAULT` group, coordinator uses `COORDINATION` group. Same broker, two sockets, no blocking.

### Files Changed

| File | Changes | What Changed |
|------|---------|--------------|
| `aiokafka/client.py` | +34/-21 | Added `ConnectionGroup` enum, changed connection key from `node_id` to `(node_id, group)` tuple |
| `aiokafka/group_coordinator.py` | +19/-9 | All coordinator requests now specify `COORDINATION` group |
| `tests/test_client.py` | modified | Updated for new tuple-based connection IDs |
| `tests/test_consumer.py` | modified | Adjusted coordinator tests |

### How It Works

**Before:**
```
connections = {
    node_id: socket
}
```

**After:**
```
connections = {
    (node_id, DEFAULT): socket_for_fetch,
    (node_id, COORDINATION): socket_for_coordinator
}
```

Coordinator calls `send(group=COORDINATION)` → gets its own socket → never blocked by fetches.

### Impact

| Aspect | Details |
|--------|---------|
| **Fixes** | Slow commits, missed heartbeats, unnecessary rebalances |
| **Affected** | Client connection pool, coordinator communication |
| **Breaking changes** | None (internal change only) |
| **Coverage** | 96.96% → 97.02% |

---

## PR 2: beets #3478 — Parallel Replaygain

### The Problem

Replaygain analysis scans audio files to calculate loudness. It's CPU-heavy. Original code did files one-by-one. Large library = hours of waiting.

Related issue: [#2224](https://github.com/beetbox/beets/issues/2224)

### The Solution

Use thread pool to analyze multiple files at once. New `--threads` flag controls parallelism. Defaults to CPU core count (same as convert plugin does).

### Files Changed

| File | Changes | What Changed |
|------|---------|--------------|
| `beetsplug/replaygain.py` | primary | Added `ThreadPool`, `--threads` arg, `ExceptionWatcher` class, callbacks for results |
| `test/test_replaygain.py` | tests | Parallel execution tests, mocked sqlite for thread safety |
| `docs/plugins/replaygain.rst` | docs | Documented `threads` option |
| `docs/changelog.rst` | docs | Added changelog entry |

### How It Works

```
Main thread                    Worker threads
     |                              |
     |---> dispatch file 1 -------> analyze loudness
     |---> dispatch file 2 -------> analyze loudness
     |---> dispatch file 3 -------> analyze loudness
     |                              |
     |<--- callback (result 1) <----|
     |<--- callback (result 2) <----|
     |---> write to DB              |
```

Key design choices:
- Workers only calculate loudness (CPU work)
- Main thread handles DB writes (avoids sqlite threading issues)
- `ExceptionWatcher` catches errors in workers (threads swallow exceptions otherwise)
- `threads: 0` = single-threaded mode for backward compatibility

### Impact

| Aspect | Details |
|--------|---------|
| **Fixes** | Slow imports on large libraries |
| **Affected** | Replaygain plugin, CPU/memory usage during import |
| **Breaking changes** | None (parallel is default but configurable) |
| **Concerns** | More memory with many threads, I/O contention on slow disks |

---

## Comparison

| Aspect | aiokafka #196 | beets #3478 |
|--------|---------------|-------------|
| **Problem** | Resource contention | Sequential bottleneck |
| **Solution** | Connection separation | Thread parallelism |
| **Complexity** | Low (2 commits) | High (20 commits) |
| **Technique** | Composite dict keys | ThreadPool + callbacks |
| **Breaking** | No | No |

---

## References

- https://github.com/aio-libs/aiokafka/pull/196
- https://github.com/aio-libs/aiokafka/issues/137
- https://github.com/beetbox/beets/pull/3478
- https://github.com/beetbox/beets/issues/2224
