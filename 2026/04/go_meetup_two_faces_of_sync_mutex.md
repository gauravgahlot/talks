---
author: ""
paging: "@GDG Golang, Berlin"
date: Apr 15, 2026
---

# The Two Faces of sync.Mutex

---

# The Two Faces of sync.Mutex

❯ $whoami

```yaml
Name: Gaurav Gahlot
Role: Staff Software Engineer, IONOS Cloud
X: _gauravgahlot
Web: https://gauravgahlot.in/
GitHub: gauravgahlot
OSS: Akri (maintainer, CNCF Sandbox)
Community: Cloud Native Berlin (organizer)
```

---

## A Simple Question

```go
var mu sync.Mutex

mu.Lock()
// critical section
mu.Unlock()
```

```

```

What happens when you call `Lock()`?

---

## What Most of Us Know

- `Lock()` blocks until the mutex is available
- `Unlock()` releases it
- Don't copy a Mutex after first use
- Don't forget to `defer mu.Unlock()`

```

```

That's enough to use it. But not enough to reason about performance.

---

## The Question Behind the Question

- Does it get slower with more goroutines?
- Does the number of cores matter?
- Is it always the same kind of "slow"?

```

```

Let's find out.

---

## Uncontended — The Baseline

```go
func BenchmarkMutexUncontended(b *testing.B) {
    var mu sync.Mutex
    for b.Loop() {
        mu.Lock()
        mu.Unlock()
    }
}
```

📌 One goroutine, no contention. Just the raw cost of Lock + Unlock.

---

## Uncontended — Results

```
BenchmarkMutexUncontended-12    ~14 ns/op
```

```

```

That's an atomic compare-and-swap. Fast path — no parking, no scheduling.

When nobody else wants the lock, the cost is almost free.

---

## The Fast Path

```go
func (m *Mutex) Lock() {
    // Fast path: grab uncontested lock
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        return
    }
    // Slow path...
    m.lockSlow()
}
```

📌 If `state == 0` (nobody holds it, nobody waiting), flip the locked bit and return.

📌 One atomic CAS. That's your ~14 ns.

---

## One int32

Go's Mutex is two fields:

```go
type Mutex struct {
    state int32
    sema  uint32
}
```

📌 `state` is where all the magic lives.

📌 `sema` is the runtime semaphore for parking/waking goroutines.

---

## Unpacking state

```
 ┌─────────────────────────────────┬────────────┬───────┬────────┐
 │         Waiter Count            │ Starvation │ Woken │ Locked │
 │           (29 bits)             │  (1 bit)   │(1 bit)│(1 bit) │
 └─────────────────────────────────┴────────────┴───────┴────────┘
  31                              3       2         1       0
```

Four pieces of information in 32 bits:

- **Locked** — someone holds the lock
- **Woken** — a goroutine is spinning; tells Unlock not to wake a sleeper
- **Starvation** — mutex switched to fair, FIFO handoff mode
- **Waiter Count** — number of goroutines parked in the semaphore queue

---

## The Semaphore — sema

`sema` is not a counter. It's an **address** — a wait queue key.

```
Lock() fails    →  park in queue[&m.sema]
Unlock()        →  wake head of queue[&m.sema]
```

📌 Every mutex has its own queue — keyed by the address of its `sema` field.

📌 Parking and waking happen in the Go runtime — no OS calls.

---

## Face #1 — Fast but Unfair (Normal Mode)

When `Lock()` finds the mutex already held:

1. **Spin** — busy-wait for a few iterations, hoping the holder releases soon
2. **Park** — if spinning didn't work, join the wait queue via the semaphore

```

```

But here's the catch: when the lock is released, who gets it?

---

## Barging

In normal mode, a **new arrival can jump the queue**.

```
~~~graph-easy
[ A: holds lock ] - Unlock() -> [ Lock: free ]
[ C: spinning on CPU ] - CAS wins -> [ Lock: free ]
[ B: parked 500µs ] - stays parked -> [ Wait Queue ]
~~~
```

📌 This is intentional. C is already running, B needs to be woken up and rescheduled.

📌 Letting C barge in is faster for overall throughput.

---

## Contended — Normal Mode

```go
func BenchmarkMutexContended(b *testing.B) {
    var mu sync.Mutex

    b.RunParallel(func(pb *testing.PB) {

        for pb.Next() {
            mu.Lock()
            mu.Unlock()
        }
    })
}
```

📌 `b.RunParallel` — distributes work across GOMAXPROCS goroutines.

---

## Contended — Results

```
GOMAXPROCS=1   BenchmarkMutexContended    ~13 ns/op
GOMAXPROCS=4   BenchmarkMutexContended    ~40 ns/op
GOMAXPROCS=12  BenchmarkMutexContended    ~153 ns/op
```

```

```

As cores increase, contention increases. But there's more to the story.

---

## Why Cores Matter

**Single core (GOMAXPROCS=1):**

- Only one goroutine runs at a time
- Spinning is useless — the lock holder can't make progress while you spin
- Go's runtime **skips spinning** on single core

**Multiple cores:**

- Spinning makes sense — the holder might be running on another core
- But more cores = more goroutines arriving at the lock simultaneously

---

## The Fairness Problem

Barging is great for throughput. But what about Goroutine B?

```
B has been waiting...
  → C barges in
  → D barges in
  → E barges in
  → B is still parked
```

📌 In theory, B could wait forever.

---

## Face #2 — Fair but Slower (Starvation Mode)

If a goroutine has been waiting for more than **1 millisecond**, the mutex switches modes.

```go
starvationThresholdNs = 1e6 // 1ms
```

```

```

The rules change completely:

- ❌ No spinning
- ❌ No barging
- ✅ Strict FIFO — lock handed directly to the longest waiter
- ✅ New arrivals go to the back of the queue

---

## The Handoff

```
~~~graph-easy
[ Normal: Unlock() ] - state = 0 (free) -> [ Race! ]
[ C: spinning ] - CAS wins -> [ Race! ]
[ B: parked ] - still waiting -> [ Race! ]
~~~
```

📌 The mutex is now unlocked.

📌 Any goroutine — a spinner, a newly arrived goroutine, a woken waiter — can race to grab it.

---

## The Handoff

```
~~~graph-easy
[ Starvation: Unlock() ] - direct handoff -> [ B: head of queue ]
[ C: new arrival ] - goes to back -> [ Wait Queue ]
~~~
```

📌 The lock is never "free" in starvation mode. It passes hand-to-hand.

📌 This means no goroutine can barge — there's nothing to CAS on.

---

## When Does It Switch Back?

Starvation mode ends when:

- The waiter that got the lock is the **last one in the queue**, OR
- The waiter's wait time was **under 1ms**

```

```

The mutex flips back to normal mode to regain throughput.

📌 It's self-tuning. High contention → fair. Low contention → fast.

---

## Starvation in Action

```go
func BenchmarkMutexStarvation(b *testing.B) {
    var mu sync.Mutex

    b.RunParallel(func(pb *testing.PB) {

        for pb.Next() {
            mu.Lock()
            time.Sleep(time.Microsecond) // hold the lock longer
            mu.Unlock()
        }

    })
}
```

📌 Holding the lock longer forces waiters past the 1ms threshold.

---

## Starvation — Results

```
GOMAXPROCS=12  BenchmarkMutexStarvation    ~22,200 ns/op
```

```

```

~145× slower than contended. Every handoff pays the wake-up cost.

📌 Lower throughput. But no goroutine waits indefinitely.

---

## The Two Faces — What to Take Away

|                | Normal Mode      | Starvation Mode |
| -------------- | ---------------- | --------------- |
| **Throughput** | Higher           | Lower           |
| **Fairness**   | None (barging)   | Strict FIFO     |
| **Spinning**   | Yes (multi-core) | No              |
| **Handoff**    | Unlocked state   | Direct transfer |
| **Trigger**    | Default          | Waiter > 1ms    |

```

```

📌 One mutex. One int32. One bit flips the entire behavior.

📌 The mutex self-tunes — normal mode for speed, starvation mode for fairness.

---

## One More Thing

```

```

"Don't communicate by sharing memory; share memory by communicating."

```

```

But now you know what happens when you do share memory. 🙂

---

# Thank you!

```yaml
Slides: https://gauravgahlot.in/talks
```
