---
author: ""
paging: "@Go Meetup Berlin"
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
BenchmarkMutexUncontended-12    ~5 ns/op
```

```

```

That's an atomic compare-and-swap. Fast path — no parking, no scheduling.

When nobody else wants the lock, the cost is almost free.

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

- **Locked** — is someone holding the lock?
- **Woken** — has a waiting goroutine been woken up?
- **Starvation** — is the mutex in starvation mode?
- **Waiter Count** — how many goroutines are waiting?

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

📌 One atomic CAS. That's your ~5 ns.

---

## The Slow Path — Normal Mode

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
Goroutine A: holding the lock
Goroutine B: parked, waiting (has been waiting 500µs)
Goroutine C: just called Lock(), spinning

→ A calls Unlock()
→ C grabs the lock (it's already on the CPU!)
→ B stays parked
```

📌 This is intentional. C is already running, B needs to be woken up and rescheduled. Letting C barge in is faster for overall throughput.

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
GOMAXPROCS=1   BenchmarkMutexContended    ~X ns/op
GOMAXPROCS=4   BenchmarkMutexContended    ~Y ns/op
GOMAXPROCS=12  BenchmarkMutexContended    ~Z ns/op
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

## Face #2 — Starvation Mode

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

Normal mode:

```
Unlock() → set state to unlocked → someone grabs it (whoever is fastest)
```

Starvation mode:

```
Unlock() → hand the lock directly to the first waiter (no unlocked state)
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

## The Two Faces — Side by Side

|                | Normal Mode      | Starvation Mode |
| -------------- | ---------------- | --------------- |
| **Throughput** | Higher           | Lower           |
| **Fairness**   | None (barging)   | Strict FIFO     |
| **Spinning**   | Yes (multi-core) | No              |
| **Handoff**    | Unlocked state   | Direct transfer |
| **Trigger**    | Default          | Waiter > 1ms    |

---

## Spinning — A Closer Look

Spinning only happens when:

- ✅ GOMAXPROCS > 1 (multi-core)
- ✅ The mutex is in normal mode
- ✅ There are waiting goroutines
- ✅ A limited number of iterations (~4 spins)

```

```

📌 On a single core, spinning is disabled entirely.

📌 Each spin calls `runtime_doSpin()` — 30 PAUSE instructions on x86.

---

## Cores vs Goroutines — The Full Picture

```
            1 core      4 cores     12 cores
1 gor       ~5 ns       ~5 ns       ~5 ns        (uncontended)
4 gor       ~X ns       ~Y ns       ~Z ns        (contended)
12 gor      ~X ns       ~Y ns       ~Z ns        (contended)
```

```

```

📌 1 goroutine is always fast — no contention.

📌 More cores ≠ more speed under a single mutex. More cores = more contenders.

---

## What to Take Away

1. **Uncontended mutexes are fast** — a single CAS, ~5 ns
2. **Contention is the cost** — not the mutex itself
3. **Go's mutex self-tunes** — normal mode for speed, starvation mode for fairness
4. **Cores amplify contention** — more parallelism hitting one lock means more waiting
5. **Spinning is smart but limited** — only multi-core, only a few iterations

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
Gaurav Gahlot
X:      _gauravgahlot
Web:    https://gauravgahlot.in/
GitHub: gauravgahlot
```
