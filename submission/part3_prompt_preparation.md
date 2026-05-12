# Part 3: Prompt Preparation Document

## Selected PR: [PR #196 — Added Separate Socket Groups to Client](https://github.com/aio-libs/aiokafka/pull/196)
**Repository:** aio-libs/aiokafka

---

## 3.1.1 Repository Context

`aiokafka` is an open-source Python library that lets developers produce and consume messages from Apache Kafka using Python's `asyncio` framework. Apache Kafka itself is a distributed message broker — a system where services publish events to named "topics," and other services subscribe to those topics to react to them. Because Kafka is so widely used in high-throughput, real-time data pipelines, having a Python client that fits natively into the async event loop is critical for modern Python backends.

The library exposes two main classes: `AIOKafkaProducer`, which sends messages, and `AIOKafkaConsumer`, which reads them. Internally, these classes rely on `AIOKafkaClient`, a lower-level connection manager that handles all TCP communication with the Kafka brokers. The client maintains persistent TCP connections to broker nodes, sends requests according to the Kafka binary protocol, and routes responses back to the appropriate caller.

The intended users are Python backend engineers building microservices, stream-processing pipelines, and event-driven applications on top of Kafka. These systems typically run as long-running async services where latency and reliability matter — dropped heartbeats or slow commits can cascade into service disruptions. The library is part of the `aio-libs` organization, a curated collection of high-quality asyncio-compatible libraries, so correctness and adherence to the Kafka protocol specification are first-class concerns. The codebase uses `asyncio` coroutines throughout, relies on `kafka-python` for protocol encoding/decoding, and employs Cython for performance-critical paths.

---

## 3.1.2 Pull Request Description

**What specific changes does this PR introduce?**

This PR changes how `aiokafka`'s client manages TCP connections to Kafka broker nodes. Before this change, the client held exactly one TCP socket per broker node. After this PR, the client can hold multiple sockets per broker — one per "socket group." Two groups are defined: a `DEFAULT` group (used by the Fetcher for reading messages) and a `COORDINATION` group (used exclusively by the Group Coordinator for sending heartbeats, committing offsets, and managing group membership).

**Why are these changes needed?**

The Kafka protocol is request–response and synchronous at the socket level: only one outstanding request per socket connection is allowed. When a `FetchRequest` — which is a long-polling request that can block for up to `fetch_max_wait_ms` milliseconds (typically 500ms) — was in-flight on the single socket to a broker node, any coordination messages the Group Coordinator needed to send to that same broker (heartbeats, commits) were forced to queue behind it. This caused commits to be delayed by hundreds of milliseconds and heartbeat responses to time out. Kafka interprets missed heartbeat deadlines as a dead consumer, which triggers a consumer group rebalance — a disruptive event that pauses all message consumption.

**Previous vs. New Behavior:**

- **Before:** 1 socket per broker node. All request types (fetch, commit, heartbeat) share the same socket. Coordination requests can be blocked by long-poll fetch requests.
- **After:** Up to 2 sockets per broker node (one per socket group). Coordination requests travel on their own dedicated connection and are never blocked by fetcher traffic.

---

## 3.1.3 Acceptance Criteria

The following criteria define a successful implementation of this PR:

✓ **When a `FetchRequest` is in-flight to a broker node, a `HeartbeatRequest` to the same node must be sent immediately** — it must not be queued behind the fetch request. It should travel on the COORDINATION socket group.

✓ **When `seek_to_end` or commit is called while a long-poll fetch is active, the commit must complete within the `session_timeout_ms` window** — there must be no artificial delay caused by socket contention.

✓ **The client must lazily create a new socket connection for the COORDINATION group** when the first coordination request is dispatched to a node that only has a DEFAULT connection.

✓ **The `GroupCoordinator` must pass `socket_group=COORDINATION` (or equivalent) to every request it issues** — including `HeartbeatRequest`, `OffsetCommitRequest`, `JoinGroupRequest`, `SyncGroupRequest`, `LeaveGroupRequest`, and `FetchOffsetRequest`.

✓ **The `AIOKafkaClient.send()` method must accept a `socket_group` parameter** and route the request to the correct connection for that `(node_id, socket_group)` pair.

✓ **Existing functionality must remain intact:** the Fetcher's `DEFAULT` group requests must continue to work correctly with no regression. All previously passing tests must still pass.

✓ **If a broker node has both socket groups active, closing or failing one socket must not affect the other** — each group must have fully independent connection lifecycle management.

---

## 3.1.4 Edge Cases

The following edge cases must be specifically considered:

**Edge Case 1 — Coordinator and Fetcher target the same broker node (most common scenario)**
In Kafka, the group coordinator is a specific broker elected per consumer group. It is common for this broker to also be the leader of one or more partitions the consumer is fetching from. This is the primary scenario this PR targets. The implementation must correctly maintain two independent sockets to the same node ID. Tests should explicitly simulate this scenario — setting up a Kafka cluster where the coordinator node is also a fetch leader.

**Edge Case 2 — Coordinator node changes (rebalance / broker failover)**
When a Kafka broker fails or a rebalance occurs, the group coordinator may be reassigned to a different broker. The client must cleanly handle tearing down the COORDINATION socket to the old coordinator node and establishing a new one to the new coordinator. If the new coordinator happens to be a node the client already has a DEFAULT socket to, a new COORDINATION socket must still be created independently. The state machine must not get confused when the coordinator node identity changes mid-session.

**Edge Case 3 — Connection failure on the COORDINATION socket**
If the COORDINATION socket to a node drops (network error, timeout), the client must reconnect the COORDINATION group socket independently without affecting the DEFAULT socket (which may still be healthy and serving fetch requests). The reconnection logic must apply the standard backoff strategy. Coordination requests that were in-flight at the time of failure must be properly failed and retriable by the coordinator without hanging indefinitely.

**Edge Case 4 — Consumer with `group_id=None` (no consumer group)**
When a consumer is created without a `group_id`, there is no Group Coordinator and no coordination traffic. In this case, the COORDINATION socket group should never be created. The implementation must verify that the socket group is only instantiated on demand and only when coordination requests are actually needed.

**Edge Case 5 — High concurrency: many coordination requests queued simultaneously**
Under high load, many coordination messages (e.g., rapid re-commits) may queue up simultaneously on the COORDINATION socket. Since the Kafka protocol allows pipelining (multiple in-flight requests on a socket if correlation IDs are managed), the implementation must correctly track `correlation_id` → future mappings per socket group independently.

---

## 3.1.5 Initial Prompt

You are an expert Python asyncio developer. Your task is to implement a specific architectural fix in the [`aiokafka`](https://github.com/aio-libs/aiokafka) library — an open-source asyncio Kafka client for Python.

**Repository Context:**
`aiokafka` allows Python applications to produce and consume Kafka messages using `asyncio`. Its core component, `AIOKafkaClient` ([`aiokafka/client.py`](https://github.com/aio-libs/aiokafka/blob/master/aiokafka/client.py)), manages all TCP connections to Kafka broker nodes. The `GroupCoordinator` ([`aiokafka/group_coordinator.py`](https://github.com/aio-libs/aiokafka/blob/master/aiokafka/group_coordinator.py)) handles consumer group heartbeats and commits, while the `Fetcher` pulls messages via long-poll requests. All I/O is non-blocking (`async def`/`await`).

**Problem:**
Currently the client holds exactly **one TCP socket per broker node**. Since Kafka's protocol allows only one outstanding request per socket at a time, a long-poll `FetchRequest` (up to 500ms) monopolizes the socket. The `GroupCoordinator` sending heartbeats or commits to the same broker is blocked behind the fetch, causing heartbeat timeouts and unnecessary consumer group rebalances (tracked in issues [#137](https://github.com/aio-libs/aiokafka/issues/137) and [#128](https://github.com/aio-libs/aiokafka/issues/128)).

**Your Task — implement [PR #196](https://github.com/aio-libs/aiokafka/pull/196):**

1. **Add a `SocketGroup` concept** in `aiokafka/client.py` with at least two named groups: `DEFAULT` (used by the Fetcher) and `COORDINATION` (used by the GroupCoordinator).

2. **Update `AIOKafkaClient`** to key its connection pool by `(node_id, socket_group)` instead of `node_id` alone. The `send()` method must accept an optional `socket_group` parameter (default: `DEFAULT`).

3. **Update `GroupCoordinator`** to pass `socket_group=COORDINATION` on every outbound request: `HeartbeatRequest`, `OffsetCommitRequest`, `JoinGroupRequest`, `SyncGroupRequest`, `LeaveGroupRequest`, `FetchOffsetRequest`.

4. **Manage connection lifecycle independently per group** — lazily create, monitor, and close each group's socket without affecting sibling group sockets on the same node.

**Acceptance Criteria (all must pass):**
- A `HeartbeatRequest` is never blocked by a concurrent in-flight `FetchRequest` to the same node
- `GroupCoordinator` uses `socket_group=COORDINATION` on all its requests
- `AIOKafkaClient.send()` correctly routes by `socket_group`
- COORDINATION socket connections are created lazily and destroyed independently of DEFAULT sockets
- All pre-existing tests pass with no regression

**Edge Cases to handle:**
- Coordinator and fetch leader are the **same broker node** (primary use case)
- Coordinator node changes mid-session (failover/rebalance) — clean up old COORDINATION socket, open new one
- `group_id=None` — COORDINATION sockets must **never** be created
- COORDINATION socket failure — reconnect independently with backoff; fail in-flight requests cleanly
- Per-socket `correlation_id` tracking must remain independent between groups

**Testing:**
- Integration test: heartbeat is not delayed while a long-poll fetch is active to the same broker
- Unit tests: `GroupCoordinator` always passes `socket_group=COORDINATION`
- Unit tests: socket group connections are created and closed independently

Follow existing `asyncio` coroutine patterns, PEP 8 style, and do not introduce unnecessary public API surface.

---

## Integrity Declaration

> "I declare that all written content in this assessment is my own work, created without the use of AI language models or automated writing tools. All technical analysis and documentation reflects my personal understanding and has been written in my own words."
