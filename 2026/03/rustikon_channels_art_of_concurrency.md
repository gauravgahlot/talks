---
theme: ferris.json
author: ""
paging: "@Rustikon"
date: Mar 20, 2026
---

# Channels in Rust: The Art of Concurrency

---

# Channels in Rust: The Art of Concurrency

❯ $whoami

```yaml
Name:   Gaurav Gahlot
Role:   Software Engineer, IONOS Cloud
X:      _gauravgahlot
Web:    https://gauravgahlot.in/
GitHub: gauravgahlot
OSS:
  - Akri (maintainer, CNCF Sandbox)
  - kube-dra (WIP)
  - Tinkerbell (ex-maintainer, CNCF Sandbox)
  - falcosidekick
  - fission
  ...
```

---

## Kubernetes Operators — Event-Driven by Design

```
~~~graph-easy
[Cluster Events] - watch -> [Operator]
[Operator] - reconcile -> [Desired State]
[Desired State] - apply -> [Kubernetes]
~~~
```

- Operators watch for resource changes and react
- Multiple events arrive concurrently from across the cluster
- Reconciliation must be reliable — missed events mean drift
- Written in Rust: ownership makes the concurrency model explicit

---

## The Right Mental Model

| Use                  | When                                     | Example                                                               |
| -------------------- | ---------------------------------------- | --------------------------------------------------------------------- |
| Mutex                | Truly shared state, many readers         | Shared counter, connection pool, in-memory cache, rate limiter state  |
| -------------------- | ---------------------------------------- | -------------------------------                                       |
| Channels             | Event-driven, pipeline-style concurrency | Request → handler, log aggregation, task queue, fan-out notifications |
| -------------------- | ---------------------------------------- | -------------------------------                                       |

```

```

📌 Kubernetes operators are event-driven and pipeline-style

Channels are the right mental model here.

---

## Channels — Ownership Transfer

📌 Channels transfer ownership — the compiler enforces it

```go
let (tx, rx) = std::sync::mpsc::channel();

let msg = String::from("hello");
tx.send(msg).unwrap();

println!("{msg}"); // ❌ compile error: msg was moved
```

---

## The Standard Library

```go
use std::sync::mpsc;

let (tx, rx) = mpsc::channel();
tx.send(msg).unwrap();
```

📌 The `send(T)` is non-blocking.

📌 `tx` can be cloned freely. `rx` cannot.

---

## The Standard Library

```go
use std::sync::mpsc;

let (tx, rx) = mpsc::sync_channel(buffer);
tx.send(msg).unwrap();
```

📌 The `send(T)` blocks the OS thread if buffer is full.

📌 The `try_send(T)` returns Err immediately if buffer is full.

📌 `tx` can be cloned freely. `rx` cannot.

---

## Channel Closure as a Signal

📌 Receiver `rx` dropped:

```go
let result = tx.send("hello".to_string());
// result == Err(SendError("hello"))
//                        ^^^^^^^ you get your value back
```

- `SendError<T>` returns the unsent value — you decide what to do with it.
- `TrySendError::Disconnected<T>` - receiver is gone.
- `TrySendError::Full<T>` - buffer is full.

```

```

📌 Sender(s) `tx` dropped:

```go
let result = rx.recv();
// result == Err(RecvError)  ← channel is empty AND closed
// result == Ok(value)       ← a value was available
```

---

## Scenario: Events at Scale

```
~~~graph-easy
[Deployment: 0 → 500] - 500 events -> [event queue ⚠]
[event queue ⚠] - process -> [1 Worker: falling behind]
~~~
```

---

## Crossbeam

📌 True multi-producer, multi-consumer

```go
let (tx, rx) = std::sync::mpsc::channel::<PodEvent>();
let rx2 = rx.clone(); // ❌ COMPILE ERROR — Receiver<T>: !Clone
```

```go
use crossbeam_channel::bounded;

let (tx, rx) = bounded::<PodEvent>(64);

for _ in 0..worker_count {
    let rx = rx.clone(); // ✅ crossbeam Receiver is Clone
    std::thread::spawn(move || {
        while let Ok(event) = rx.recv() {
            process(event);
        }
    });
}
```

📌 Each event goes to exactly one worker — no duplication, no mutex

📌 Both `std::sync::mpsc` and `crossbeam` block the OS thread

---

## Going async w/ `tokio`

📌 Async needs its own channels

```go
// Blocks the OS thread — kills Tokio's cooperative scheduling:
let value = std::sync::mpsc::channel().1.recv().unwrap(); // ❌ in async

// Yields the task — thread stays free for other work:
let value = tokio::sync::mpsc::channel(32).1.recv().await; // ✅
```

- `std::sync::mpsc::recv()` parks the OS thread.
- `tokio::sync::mpsc::recv().await` suspends only _this task_.

---

## tokio::sync::mpsc

```go
// Bounded — backpressure built in
let (tx, mut rx) = mpsc::channel::<Event>(10);
// .send().await pauses when buffer is full — producer slows down naturally

// Unbounded — no backpressure, grows until OOM
let (tx, mut rx) = mpsc::unbounded_channel::<Event>();
// .send() is sync and always succeeds (as long as receiver is alive)
```

---

## tokio::sync::mpsc

```go
// Producer
async fn producer(tx: mpsc::Sender<Event>) {
    loop {
        let event = detect_change().await;
        if tx.send(event).await.is_err() {
            return; // receiver gone — stop producing
        }
        // buffer full? .send().await yields the task (backpressure)
    }
}

// Consumer
async fn consumer(mut rx: mpsc::Receiver<Event>) {
    while let Some(event) = rx.recv().await {
        process(event).await;
    }
    // None — all senders dropped and buffer drained
}
```

---

## Scenario: kubectl exec

```
~~~graph-easy
[kubectl exec] - spawn + oneshot tx -> [Remote Command Task]
[Remote Command Task] - streams output -> [stdout / stderr]
[Remote Command Task] - oneshot: Status -> [kubectl exec]
~~~
```

📌 Output streams continuously — but there is exactly one final Status

---

## Oneshot: The Request-Reply Pattern

📌 `mpsc` for streams of work — `oneshot` for a single response

```go
use tokio::sync::oneshot;

let (tx, rx) = oneshot::channel::<Status>();
// tx: Sender<T> — can only send once, then is consumed
// rx: Receiver<T> — can only receive once

tokio::spawn(async move {
    let status = run_remote_command().await;
    tx.send(status).ok(); // send() consumes tx — no clone, no second send
});

let status = rx.await.unwrap(); // await the Receiver directly
```

```

```

- `oneshot::Sender` is not `Clone`.
- Exactly one send.
- The Receiver is a `Future`.

---

## Scenario: Configuration Reconcile Triggers

```
~~~graph-easy
[Pod watcher] - resource changed -> [Reconciler]
[Deployment watcher] - resource changed -> [Reconciler]
[ConfigMap watcher] - resource changed -> [Reconciler]
~~~
```

📌 Each watcher holds a cloned sender — one reconciler processes all triggers

---

## Fan-in

📌 `mpsc` is designed for fan-in: many producers, one consumer

```
Pod watcher        ──► tx.clone() ──┐
Deployment watcher ──► tx.clone() ──┼──► rx (Reconciler)
ConfigMap watcher  ──► tx.clone() ──┘
```

```go
let tx_pod        = tx.clone();
let tx_deployment = tx.clone();
let tx_configmap  = tx.clone();

// each watcher sends when resources change — reconciler processes all triggers
while let Some(config_ref) = rx.recv().await {
    reconcile(config_ref).await;
}
```

📌 Next: what if you need to notify _many_ recipients simultaneously?

---

## Scenario: Node Goes NotReady

```
~~~graph-easy
[Node: NotReady] - event -> [Informer]
[Informer] - broadcast -> [Pod Eviction Controller]
[Informer] - broadcast -> [Endpoint Controller]
[Informer] - broadcast -> [NodeLifecycle Controller]
~~~
```

📌 One event, many independent reactions — each controller acts on its own

---

## Fan-out: Broadcast

📌 Every subscriber gets every message

```go
use tokio::sync::broadcast;

let (tx, _) = broadcast::channel::<NodeEvent>(16);

let mut rx1 = tx.subscribe(); // Pod Eviction Controller
let mut rx2 = tx.subscribe(); // Endpoint Controller
let mut rx3 = tx.subscribe(); // NodeLifecycle Controller

// node goes NotReady — every controller receives it
tx.send(node_event)?;
```

---

## The Silent Failure in Broadcast

📌 `tokio::sync::broadcast` silently drops messages when the buffer is full.

```go
let (tx, _) = broadcast::channel::<NodeEvent>(10);
```

```go
match node_event_rx.recv().await {
    Ok(event)                 => { /* react to node event */ }

    Err(RecvError::Lagged(n)) => {
        // missed n node events — re-sync state
        warn!("missed {n} node events — re-syncing");
        resync_node_state().await;
    }

    Err(RecvError::Closed)    => return,
}
```

📌 If 11 node events arrive before any controller wakes up — the oldest is silently overwritten. Receivers see `Lagged(1)`.

---

## The Fix: `async_broadcast`

```go
use async_broadcast::broadcast;

let (tx, rx) = broadcast::<NodeEvent>(10);

// buffer full? tx.broadcast(event).await pauses the sender
// no receiver ever lags silently
tx.broadcast(node_event).await?;
```

```

```

|                | `broadcast`              | `async_broadcast`            |
| -------------- | ------------------------ | ---------------------------- |
| Buffer full    | Drops oldest, silently   | Backpressure — sender pauses |
| -------------- | ------------------------ | --------------------------   |
| Message loss   | Possible                 | None                         |
| -------------- | ------------------------ | --------------------------   |

📌 kube-rs uses `async_broadcast` in its reflector for exactly this reason.

---

## Scenario: Controller Racing Multiple Inputs

```
~~~graph-easy
[Reconcile Queue] - work_rx -> [Controller loop]
[Node Events] - node_event_rx -> [Controller loop]
[Shutdown] - token.cancelled -> [Controller loop]
~~~
```

📌 A controller never blocks on one channel — it races all its inputs

---

## tokio::select!

> Races multiple async futures — resolves to whichever is ready first.
> Add `biased;` to check arms in order (useful for shutdown priority).

```go
loop {
    tokio::select! {
        Some(work) = work_rx.recv() => {
            process(work).await;
        }

        Ok(event) = node_event_rx.recv() => {
            update_state(event);
        }

        _ = token.cancelled() => {
            return; // shutdown — clean exit
        }
    }
}
```

---

## Scenario: Node Conditions

```
~~~graph-easy
[kubelet] - publishes -> [node conditions tx]
[node conditions tx] - latest state -> [Scheduler]
[node conditions tx] - latest state -> [NodeLifecycle Controller]
~~~
```

📌 10 rapid condition flips — consumers only see the final state

---

## watch: Only the Latest Value Matters

```
broadcast: [msg1] [msg2] [msg3] [msg4]  ← subscribers can be behind
watch:                           [msg4] ← always the latest, nothing else
```

```

```

| Channel     | Use case                                                                |
| ----------- | ----------------------------------------------------------------------- |
| `broadcast` | Every event matters (node events, config changes)                       |
| ----------- | --------------------------                                              |
| `watch`     | Only current state matters (node conditions, health flag, drain signal) |
| ----------- | --------------------------                                              |

---

## tokio::sync::watch

```go
use tokio::sync::watch;

let (tx, _rx) = watch::channel(vec![]); // current node conditions

// Producer
loop {
    tokio::select! {
        _ = tx.closed()          => return, // all receivers dropped — stop
        _ = token.cancelled()    => return, // shutdown requested
        conditions = sync_conditions() => {
            tx.send_replace(conditions); // latest wins — old value overwritten
        }
    }
}

// Consumer
let mut rx = tx.subscribe();
loop {
    rx.changed().await?; // waits until a new value is sent
    let conditions = rx.borrow_and_update().clone();
    reconcile(conditions).await;
}
```

📌 `tx.closed()` completes when all receivers are dropped — implicit shutdown for free

---

## Backpressure

📌 A product decision, not just a buffer size

```
~~~graph-easy
[Deployment: 0 → 500] - 500 Pod events -> [event queue ⚠]
[event queue ⚠] - process -> [Reconciler: falling behind]
~~~
```

```go
let (tx, mut rx) = mpsc::channel::<ReconcileEvent>(10);

// Every time you choose a strategy, you are making a statement about
// what your system values when it is under pressure.
```

---

## Backpressure — Block

📌 Events must not be lost

```go
// Producer pauses until there is space.
// Backpressure propagates up the call chain.
tx.send(event).await?;
```

**Use this when:** every event must be processed.

Pod scheduling events — missing one means a workload never gets placed.

---

## Backpressure — Drop

📌 Throughput over completeness

```go
match tx.try_send(event) {
    Ok(())                       => {}

    Err(TrySendError::Full(v))   => {
        // Buffer full. You get the value back. Decide what to do.
        metrics::increment_counter("events_dropped");
        // value v is dropped here
    }

    Err(TrySendError::Closed(_)) => {
        return; // receiver gone, stop producing
    }
}
```

**Use this when:** high-volume telemetry where occasional loss is acceptable.

Sampling every 10th event is better than OOMing on all of them.

---

## Backpressure — Timeout

📌 SLA-bound systems

```go
use std::time::Duration;

match tx.send_timeout(event, Duration::from_millis(100)).await {
    Ok(())                            => {}

    Err(SendTimeoutError::Timeout(v)) => {
        warn!("consumer too slow, dropped event after 100ms: {v:?}");
    }

    Err(SendTimeoutError::Closed(_))  => {
        return; // receiver gone, stop producing
    }
}
```

**Use this when:** you have a latency SLA.

Events older than N milliseconds are no longer useful — better to drop than deliver stale data.

---

## Backpressure — The Numbers Matter

| Buffer     | Consequence                                                     |
| ---------- | --------------------------------------------------------------- |
| Too small  | Backpressure too aggressive — producer stalls, throughput drops |
| ---------  | ------------------------------                                  |
| Too large  | OOM risk if producer is consistently faster than consumer       |
| ---------  | ------------------------------                                  |
| Just right | Absorbs bursts, applies pressure on sustained overload          |
| ---------  | ------------------------------                                  |

```

```

```go
let (tx, rx) = mpsc::channel::<ReconcileEvent>(10);
//                                              ^^
// 10 pending events
// each event takes ~50ms to process
// burst capacity: 10 × 50ms = 500ms
// beyond that: producer pauses → consumer catches up
```

---

## Graceful Shutdown

📌 `watch` channel as a shutdown signal everyone can observe

```go
let (stop_tx, _) = tokio::sync::watch::channel(false);

// signal shutdown from anywhere:
stop_tx.send_replace(true);

// react in any task — two ways to stop:
let mut stop_rx = stop_tx.subscribe();

tokio::select! {
    _ = stop_rx.changed() => return, // explicit: send_replace(true) called
    _ = tx.closed()       => return, // implicit: all receivers dropped
    Some(msg) = rx.recv() => process(msg).await,
}
```

- `watch::Sender` is internally `Arc` — clone and share freely, no `Mutex` needed
- Ownership gives you shutdown signalling for free

---

## The Modern Answer: CancellationToken

```go
// Hierarchical: cancelling parent automatically cancels all children
let parent = tokio_util::sync::CancellationToken::new();
let child  = parent.child_token();

tokio::spawn(async move {
    tokio::select! {
        _ = child.cancelled() => { /* stop immediately */ }
        _ = do_work()         => {}
    }
});

// Shutdown from the top — propagates to all child tokens automatically
parent.cancel();
```

📌 `CancellationToken` is the modern answer to coordinated task shutdown in Kubernetes controllers

---

## Channel Quick Reference

| Question                                 | Use this                                |
| ---------------------------------------- | --------------------------------------- |
| One receiver, multiple producers?        | `std::sync::mpsc` / `tokio::sync::mpsc` |
| ---------------------------------------- | --------------------------------------- |
| Multiple workers compete for work?       | `crossbeam` MPMC                        |
| ---------------------------------------- | --------------------------------------- |
| Every receiver gets every message?       | `tokio::sync::broadcast`                |
| ---------------------------------------- | --------------------------------------- |
| Only latest value matters?               | `tokio::sync::watch`                    |
| ---------------------------------------- | --------------------------------------- |
| One-time reply to a specific caller?     | `tokio::sync::oneshot`                  |
| ---------------------------------------- | --------------------------------------- |
| Hierarchical task cancellation?          | `tokio_util::sync::CancellationToken`   |
| ---------------------------------------- | --------------------------------------- |
| Fan-out, buffer overflow must not drop?  | `async_broadcast`                       |
| ---------------------------------------- | --------------------------------------- |
| React to whichever input is ready first? | `tokio::select!`                        |
| ---------------------------------------- | --------------------------------------- |

---

# Thank you! 🦀

📌 Slides: https://gauravgahlot.in/talks/

📌 References:

- 🎬 Crust of Rust (https://youtube.com/@jonhoo)
- 📚 Rust Atomics and Locks
- 📚 Rust for Rustaceans
