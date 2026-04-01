# Fairness Scheduling — Architecture Design Review & Improvement Proposal

**Author:** Architecture Review  
**Date:** 2026-04-01  
**Status:** Draft  
**Component:** `pkg/kthena-router` (router, datastore, scheduler)

---

## 1. Current Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                         Request Flow (Fairness ON)                     │
│                                                                        │
│  HTTP Request                                                          │
│      │                                                                 │
│      ▼                                                                 │
│  ┌──────────┐    ┌──────────────┐    ┌────────────────────────┐       │
│  │  Router   │───▶│ GetTokenCount│───▶│ InMemorySlidingWindow  │       │
│  │ Handler   │    │  (priority)  │    │    TokenTracker         │       │
│  └──────────┘    └──────────────┘    │  [user][model] → float  │       │
│      │                               └────────────────────────┘       │
│      ▼                                                                 │
│  ┌──────────────────────────────┐                                      │
│  │    store.Enqueue(Request)     │                                      │
│  │  ┌────────────────────────┐  │                                      │
│  │  │ Per-Model Priority Queue│  │   One queue per model               │
│  │  │  (min-heap by priority) │  │   Run() goroutine @ 100 QPS        │
│  │  │                        │  │                                      │
│  │  │  Less():               │  │                                      │
│  │  │   same user → FIFO     │  │                                      │
│  │  │   diff user → lower    │  │                                      │
│  │  │     token usage first  │  │                                      │
│  │  └────────────────────────┘  │                                      │
│  └──────────────────────────────┘                                      │
│      │                                                                 │
│      │ <── blocks on NotifyChan (max 60s) ──┐                          │
│      │                                      │                          │
│      ▼                               ┌──────┴──────┐                   │
│  ┌──────────┐                        │  Run() loop  │                   │
│  │doLoadbal │◀── close(NotifyChan) ──│  ticker @    │                   │
│  │  ance()  │                        │  10ms/tick   │                   │
│  └──────────┘                        └─────────────┘                   │
│      │                                                                 │
│      ▼                                                                 │
│  ┌──────────┐    ┌──────────────┐                                      │
│  │ Scheduler│───▶│ Proxy to Pod │                                      │
│  └──────────┘    └──────────────┘                                      │
│      │                                                                 │
│      ▼                                                                 │
│  ┌──────────────┐                                                      │
│  │UpdateTokenCnt│  (post-response, records completion tokens)          │
│  └──────────────┘                                                      │
└────────────────────────────────────────────────────────────────────────┘
```

### Current Components

| Component | Location | Responsibility |
|-----------|----------|----------------|
| `handleFairnessScheduling` | `router.go:965` | Entry point; extracts userId, gets priority, enqueues, blocks |
| `RequestPriorityQueue` | `fairness_queue.go` | Min-heap ordered by token usage; per-model goroutine dequeues at fixed QPS |
| `InMemorySlidingWindowTokenTracker` | `token_tracker.go` | Sliding-window weighted token accumulator per (user, model) |
| `EnableFairnessScheduling` | `router.go:68` | Global env-var kill switch (`ENABLE_FAIRNESS_SCHEDULING`) |
| Metrics | `metrics.go` | `fairness_queue_size`, `fairness_queue_duration_seconds` |

---

## 2. Identified Design Gaps

### Gap 1: Fixed-Rate Dequeue (Critical)

**Problem:** `Run()` dequeues at a **fixed QPS** (default 100) regardless of actual backend throughput or load. This creates two failure modes:

- **Under-provisioned:** If the model backends can serve 500 req/s, the queue artificially caps throughput at 100 req/s, adding unnecessary latency.
- **Over-provisioned:** If backends can only handle 20 req/s, the queue releases 100 req/s, overwhelming backends and negating the fairness benefit.

**Impact:** The dequeue rate is entirely disconnected from actual serving capacity.

### Gap 2: Priority Is a Point-in-Time Snapshot (Medium)

**Problem:** Priority is computed **once** at enqueue time via `GetTokenCount(userId, modelName)`. While the request waits in the queue (potentially up to 60s), other users' priorities change as their requests complete and tokens accumulate. The queued request's priority is never re-evaluated.

**Impact:** A user who had low token usage at enqueue time but whose other concurrent requests finish (accumulating tokens) will still be treated as low-usage, violating the fairness invariant.

### Gap 3: No Request Cancellation / Client Disconnect Handling (Medium)

**Problem:** When a client disconnects (TCP RST, context cancelled), the `Request` stays in the priority queue. `NotifyChan` is still closed when dequeued, and `doLoadbalance()` runs against a dead `gin.Context`. There is no mechanism to:
1. Remove cancelled requests from the queue.
2. Detect `c.Request.Context().Done()` during the wait.

**Impact:** Wasted backend capacity on dead requests; queue contains phantom entries that distort fairness ordering.

### Gap 4: Queue Goroutine Lifecycle Leak (Medium)

**Problem:** `Enqueue()` calls `go newQueue.Run(context.TODO(), defaultQueueQPS)` — the context is `context.TODO()` which never cancels. Queues are per-model and created lazily, but never cleaned up when a model route is deleted.

**Impact:** Goroutine leak proportional to the number of distinct model names ever seen. Each leaked goroutine also holds a ticker and channel.

### Gap 5: No Multi-Model Fairness (Low-Medium)

**Problem:** Fairness operates independently per model queue. A user consuming heavy resources across 10 different models is treated as a low-priority user on each individual model, because token counts are tracked per `(user, model)`.

**Impact:** Users can circumvent fairness by spreading load across models (relevant in multi-model deployments).

### Gap 6: In-Memory Token Tracker Not HA-Safe (Low)

**Problem:** `InMemorySlidingWindowTokenTracker` is local to a single router pod. In multi-replica deployments, each replica has independent token accounting. User A can get full priority on replica-1 while being heavy on replica-2.

**Impact:** Fairness enforcement weakens linearly with replica count.

### Gap 7: Hard-Coded 60-Second Timeout (Low)

**Problem:** The fairness wait timeout is hard-coded to 60 seconds. No configuration surface exists. For latency-sensitive workloads this may be too high; for batch workloads it may be too low.

### Gap 8: Token-Only Priority Without Request-Count Awareness (Low)

**Problem:** Priority is purely token-based. A user sending many small requests (low tokens each) can monopolize the queue position over a user sending fewer but larger requests, even if the small-request user is consuming disproportionate scheduling/compute overhead.

---

## 3. Proposed Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Improved Fairness Scheduling                         │
│                                                                         │
│  HTTP Request                                                           │
│      │                                                                  │
│      ▼                                                                  │
│  ┌──────────────────────────────────────────────────┐                   │
│  │              AdmissionController                  │                   │
│  │  1. Extract userId                                │                   │
│  │  2. Get priority (deferred to dequeue)            │                   │
│  │  3. Enqueue with gin.Context                      │                   │
│  └──────────────────────────────────────────────────┘                   │
│      │                                                                  │
│      ▼                                                                  │
│  ┌──────────────────────────────────────────────────┐                   │
│  │           FairnessQueue (per-model)               │                   │
│  │                                                   │                   │
│  │  ┌─────────────────────────────┐                  │                   │
│  │  │  Dequeue Strategy           │                  │                   │
│  │  │  ┌───────────────────────┐  │                  │                   │
│  │  │  │ Backpressure-Aware    │  │  Dequeue rate    │                   │
│  │  │  │ Semaphore:            │  │  = min(QPS cap,  │                   │
│  │  │  │  permits =            │  │    available     │                   │
│  │  │  │  activeUpstream       │  │    capacity)     │                   │
│  │  │  │  Capacity - inFlight  │  │                  │                   │
│  │  │  └───────────────────────┘  │                  │                   │
│  │  │                             │                  │                   │
│  │  │  Priority re-eval at pop:   │                  │                   │
│  │  │  fresh GetTokenCount()      │                  │                   │
│  │  └─────────────────────────────┘                  │                   │
│  │                                                   │                   │
│  │  Cancellation:                                    │                   │
│  │  - Request carries ctx from gin.Context           │                   │
│  │  - Cancelled entries skipped on dequeue           │                   │
│  │  - Periodic sweep removes stale entries           │                   │
│  └──────────────────────────────────────────────────┘                   │
│      │                                                                  │
│      │ close(NotifyChan)                                                │
│      ▼                                                                  │
│  ┌──────────────────────────────────────────────────┐                   │
│  │              doLoadbalance()                       │                   │
│  │              (unchanged)                           │                   │
│  └──────────────────────────────────────────────────┘                   │
│      │                                                                  │
│      ▼                                                                  │
│  ┌──────────────────────────────────────────────────┐                   │
│  │           TokenTracker (pluggable)                │                   │
│  │  ┌────────────┐  ┌──────────────┐                │                   │
│  │  │  InMemory   │  │ Redis-backed │  (future)     │                   │
│  │  │  (default)  │  │ (HA deploy)  │               │                   │
│  │  └────────────┘  └──────────────┘                │                   │
│  └──────────────────────────────────────────────────┘                   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Detailed Improvement Proposals

### 4.1 Backpressure-Aware Dequeue (Addresses Gap 1)

**Goal:** Replace the fixed-rate ticker with a capacity-based admission control.

**Design:**

```go
// FairnessQueueConfig holds configurable parameters for the fairness queue
type FairnessQueueConfig struct {
    // MaxConcurrent is the maximum number of in-flight requests 
    // allowed through the fairness gate for this model.
    // When 0, falls back to MaxQPS-based rate limiting.
    MaxConcurrent int

    // MaxQPS is the upper-bound dequeue rate (retained as a safety cap).
    MaxQPS int
}
```

Replace the fixed ticker in `Run()` with a **semaphore + signal** pattern:

```
On dequeue:
  1. Acquire semaphore permit (blocks if at capacity)
  2. Pop from heap
  3. Close NotifyChan → request proceeds to doLoadbalance()

On request completion (new callback):
  4. Release semaphore permit → unblocks next dequeue
```

**Capacity source:** `MaxConcurrent` can be derived from:
- Static configuration per model (simplest, recommended first).
- Dynamic: `sum(pod_count) × per_pod_concurrency` from ModelServer spec.

**Fallback:** When `MaxConcurrent = 0`, retain the existing QPS-ticker behavior for backward compatibility.

**Key change in `Request` struct:**

```go
type Request struct {
    // ... existing fields ...
    DoneChan chan struct{} // Closed by router after proxy completes (releases permit)
}
```

### 4.2 Dequeue-Time Priority Re-evaluation (Addresses Gap 2)

**Goal:** Ensure the request with the genuinely lowest token usage is dequeued, not the one that *had* the lowest usage at enqueue time.

**Design:**

Instead of a strict heap pop, use a **lazy re-evaluation** strategy:

```
On popWhenAvailable():
  1. Peek top-K items from heap (K = min(8, heap.Len()))
  2. For each item, refresh priority: item.Priority = GetTokenCount(item.UserID, item.ModelName)
  3. Re-heapify the peeked items
  4. Pop the true minimum
```

**Why top-K instead of full re-heap:** Full re-evaluation is O(n log n) on every pop. Top-K bounds the overhead to O(K log K) while capturing the most likely candidates (heap property guarantees the true minimum is within the top levels with high probability).

**Configuration:**

```go
// ReevalTopK controls how many top items are re-evaluated on each pop.
// 0 disables re-evaluation (current behavior).
ReevalTopK int  // default: 8
```

### 4.3 Request Cancellation Support (Addresses Gap 3)

**Goal:** Stop wasting backend capacity on requests whose clients have disconnected.

**Design:**

Add a context to `Request`:

```go
type Request struct {
    // ... existing fields ...
    Ctx context.Context // From c.Request.Context(); carries client lifecycle
}
```

**Three-point cancellation:**

1. **At enqueue:** Store `c.Request.Context()` in the `Request`.
2. **At dequeue (in `popWhenAvailable`):** After popping, check `req.Ctx.Err() != nil`. If cancelled, skip (decrement metrics, continue to next).
3. **At wait site (in `handleFairnessScheduling`):** Add `c.Request.Context().Done()` to the select:

```go
select {
case <-queueReq.NotifyChan:
    r.doLoadbalance(c, modelRequest)
    return nil
case <-c.Request.Context().Done():
    // Client disconnected; request will be skipped at dequeue
    return fmt.Errorf("client disconnected while waiting in fairness queue")
case <-time.After(r.fairnessTimeout):
    // timeout handling (unchanged)
}
```

### 4.4 Queue Lifecycle Management (Addresses Gap 4)

**Goal:** Prevent goroutine/resource leaks for deleted models.

**Design:**

1. Pass a **cancellable context** to `Run()` instead of `context.TODO()`:

```go
func (s *store) Enqueue(req *Request) error {
    // ...
    if !ok {
        ctx, cancel := context.WithCancel(s.rootCtx)
        newQueue := NewRequestPriorityQueue(nil)
        newQueue.cancel = cancel  // store cancel func
        go newQueue.Run(ctx, defaultQueueQPS)
    }
    // ...
}
```

2. Register a **model-delete callback** that cancels the queue context and removes it from the `sync.Map`:

```go
store.RegisterCallback("ModelRoute", func(data EventData) {
    if data.EventType == EventDelete {
        if val, ok := s.requestWaitingQueue.LoadAndDelete(data.ModelName); ok {
            queue := val.(*RequestPriorityQueue)
            queue.Close()  // triggers cancel + drains remaining requests with error
        }
    }
})
```

3. Add an **idle timeout** (e.g., 10 minutes with no enqueue) — if a queue is empty for the idle period, auto-close and remove.

### 4.5 Configurable Timeout (Addresses Gap 7)

**Goal:** Allow operators to tune the fairness wait timeout.

**Design:**

```go
// Environment variable: FAIRNESS_QUEUE_TIMEOUT (default: "60s")
type FairnessConfig struct {
    QueueTimeout time.Duration
}
```

Read from env in `NewRouter()`, store in the `Router` struct, and use in `handleFairnessScheduling`:

```go
case <-time.After(r.fairnessConfig.QueueTimeout):
```

### 4.6 Composite Priority Score (Addresses Gap 8)

**Goal:** Factor in request count alongside token usage for a more balanced priority.

**Design:**

Extend the priority formula:

```
priority = α × weighted_tokens + β × request_count_in_window
```

Where:
- `weighted_tokens` = current `GetTokenCount()` value
- `request_count_in_window` = number of requests from this user in the sliding window
- `α`, `β` = configurable weights (default `α=1.0`, `β=0.0` for backward compat)

This requires the `TokenTracker` to also track request counts. Extend the interface:

```go
type TokenTracker interface {
    GetTokenCount(user, model string) (float64, error)
    GetRequestCount(user, model string) (int, error)   // new
    UpdateTokenCount(user, model string, inputTokens, outputTokens float64) error
}
```

**Note:** `β=0.0` by default ensures no behavior change unless explicitly configured.

---

## 5. Implementation Phases

### Phase 1: Safety & Correctness (Recommended First)

| Item | Gap | Effort | Risk |
|------|-----|--------|------|
| Request cancellation support (4.3) | Gap 3 | Small | Low |
| Queue lifecycle management (4.4) | Gap 4 | Small | Low |
| Configurable timeout (4.5) | Gap 7 | Trivial | None |

These are bug-fixes / resource-leak fixes with no behavioral change to existing users.

### Phase 2: Core Fairness Improvement

| Item | Gap | Effort | Risk |
|------|-----|--------|------|
| Backpressure-aware dequeue (4.1) | Gap 1 | Medium | Medium — needs load testing |
| Dequeue-time priority re-eval (4.2) | Gap 2 | Small | Low |

These improve the quality of fairness scheduling. The backpressure change should be gated behind a configuration flag initially.

### Phase 3: Advanced (Future)

| Item | Gap | Effort | Risk |
|------|-----|--------|------|
| Composite priority (4.6) | Gap 8 | Small | Low |
| Cross-model fairness (Gap 5) | Gap 5 | Medium | Medium |
| Distributed token tracker (Gap 6) | Gap 6 | Large | Medium — needs Redis dependency |

---

## 6. Recommended Design

**Phase 1 + Phase 2 (items 4.1–4.5)** should be implemented together as a single coherent release. The recommended target architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                 Target State Summary                         │
├───────────────────────┬─────────────────────────────────────┤
│ Dequeue strategy      │ Semaphore-gated (capacity-aware)    │
│                       │ with QPS cap as safety bound        │
├───────────────────────┼─────────────────────────────────────┤
│ Priority evaluation   │ Lazy top-K re-eval at dequeue time  │
├───────────────────────┼─────────────────────────────────────┤
│ Cancellation          │ Context-propagated; skip at dequeue │
├───────────────────────┼─────────────────────────────────────┤
│ Queue lifecycle       │ Cancellable context + idle timeout   │
├───────────────────────┼─────────────────────────────────────┤
│ Timeout               │ Configurable via env var             │
├───────────────────────┼─────────────────────────────────────┤
│ Token tracker         │ InMemory (unchanged); interface      │
│                       │ ready for Redis backend              │
├───────────────────────┼─────────────────────────────────────┤
│ Backward compatibility│ All new behaviors gated by config;   │
│                       │ defaults preserve current behavior   │
└───────────────────────┴─────────────────────────────────────┘
```

This design addresses the critical throughput mismatch (Gap 1), correctness issues (Gaps 2-4), and operability (Gap 7) while keeping the implementation incremental and backward-compatible.
