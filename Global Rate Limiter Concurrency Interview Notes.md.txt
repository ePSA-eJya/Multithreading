# Global Rate Limiter: Concurrency Interview Notes

Oct 24, 2026 · @Epsa Sharma

## Problem and requirements

Build a `RateLimiter` shared by many threads in one process so that the whole process never sends more than N requests in any rolling T-second window.

```cpp
class RateLimiter {
public:
    RateLimiter(size_t maxRequests, chrono::steady_clock::duration window);
    void acquire();     // blocks until a slot is free, then takes it
    bool tryAcquire();  // takes a slot and returns true, or returns false immediately
};
```

**Clarified requirements**

- **Rolling window, not fixed.** For any time t, requests in (t − T, t\] must be ≤ N. A fixed window that resets every T lets 2N requests through around a boundary (5 at 0.99s + 5 at 1.01s).
- **One shared budget.** N slots per T cover every thread, whichever method it calls.
- **You implement both methods.** Callers pick one per request: `acquire()` means "I'll wait"; `tryAcquire()` means "only if a slot is free now".
- **No `release()`.** The limiter controls when a request may start, not how many run at once. That's the difference from a semaphore.

|  | `acquire()` | `tryAcquire()` |
| --- | --- | --- |
| Slot free | Takes it, returns | Takes it, returns `true` |
| No slot | Sleeps until one frees up | Returns `false` immediately |

## Algorithm: sliding window log

Keep a deque of timestamps for granted requests. It enforces the rolling window exactly in O(1) amortized time and O(N) memory.

1. Read `now`.
2. Evict from the front every timestamp with `now - ts >= T`.
3. If `size < N`, push `now` to the back and grant. Otherwise refuse.

**Notes**

- **Use a monotonic clock** (`steady_clock`). Wall-clock time can jump backward with NTP and corrupt the window.
- **Read `now` inside the lock.** That keeps the deque sorted, so evicting only from the front is safe.
- **Amortized O(1):** each timestamp is pushed once and popped once.
- Store the window as a `duration`, not `int` seconds, so 500ms windows work. Make the max a `size_t` to avoid signed/unsigned comparisons.

## Thread safety

Evict, check and push must run in one critical section under one mutex. Otherwise two threads can both see a free slot.

**The bug: a check-then-act race** (N = 5, deque holds 4)

```text
Thread A: evict → size 4 < 5 ✓
Thread B: evict → size 4 < 5 ✓
Thread A: push → size 5
Thread B: push → size 6   ← limit broken
```

**Rules**

- **Don't release the lock between steps.** Locking for eviction, unlocking, then locking again for the push lets another thread take the last slot in between.
- **Don't split head and tail locks.** The decision depends on `size()`, which both ends change.
- **A lock protects data, not a function.** Separate locks for `tryAcquire()` and `acquire()` give no exclusion between them, since both touch the same deque. Rule: one lock per piece of shared state.
- **Split into a `...Locked` helper.** `std::mutex` isn't recursive, so `acquire()` can't call a `tryAcquire()` that locks again. Put the logic in a private helper that assumes the caller holds the lock.
- **Validate `maxRequests > 0`** in the constructor. With 0, the deque is empty when a request fails and `dq.front()` is undefined behavior.

```cpp
private:
    // Caller MUST hold m
    bool tryAcquireLocked(chrono::steady_clock::time_point now) {
        while (!dq.empty() && now - dq.front() >= window) dq.pop_front();
        if (dq.size() < maxR) { dq.push_back(now); return true; }
        return false;
    }

public:
    bool tryAcquire() {
        lock_guard<mutex> lock(m);
        return tryAcquireLocked(chrono::steady_clock::now());
    }
```

## Blocking acquire()

Sleep with `cv.wait_until(lock, dq.front() + window)`. That's the exact moment the next slot frees up, and no `notify` is ever needed.

**Why not busy-wait** (`while (!tryAcquire()) {}`)

- It burns CPU.
- Every spinning thread keeps grabbing the mutex, which slows down everyone else.

**What the thread is actually waiting for.** It isn't waiting for the lock, which is held for microseconds. It's waiting for time to pass until the oldest timestamp leaves the window.

```text
N = 3, T = 1s, now = 10.95s
[ 10.2s, 10.5s, 10.9s ]  → next slot at 10.2 + 1 = 11.2s
```

**Why no notify.** A semaphore frees a slot when someone calls `release()`. Here slots free up because time passes, and every waiter already knows its deadline.

```cpp
void acquire() {
    unique_lock<mutex> lock(m);   // unique_lock: wait_until must unlock and relock it
    while (!tryAcquireLocked(chrono::steady_clock::now())) {
        cv.wait_until(lock, dq.front() + window);
    }
}
```

- **Loop:** re-check after every wakeup. It may be spurious, or another thread may have taken the slot.
- **Recompute the deadline** every pass, since `dq.front()` may have changed.

## Thundering herd, leader/follower and fairness

With one leader doing the timed wait, only one thread wakes per slot instead of all of them.

**Thundering herd.** If 50 threads block in `acquire()`, all compute the same deadline and wake together at 11.2s. One wins and 49 go back to sleep, wasting wakeups and mutex round-trips.

**Leader/follower fix**

- One **leader** does `wait_until(front + window)`.
- **Followers** do a plain `wait()` with no deadline. Sleeping threads use no CPU.
- When the leader gets its slot, it steps down and calls `notify_one()` to promote the next follower.

```cpp
class RateLimiter {
    mutex m;
    condition_variable cv;   // leader's timed wait
    condition_variable fv;   // followers wait for leadership
    bool hasleader = false;
    // dq, maxR, wind, trywithlock(...) as before

public:
    void acq(){
        unique_lock<mutex> lock(m);
        while(hasleader){
            fv.wait(lock);
        }
        hasleader=true;
        while(!trywithlock(chrono::steady_clock::now())){
            cv.wait_until(lock,dq.front()+wind);
        }
        hasleader=false;
        fv.notify_one();
    }
};
```

- **Use a `bool` member**, not a global `int` shared by every limiter instance.
- **RAII step-down:** without it, an exception leaves `hasLeader` true and every future `acquire()` deadlocks.
- **Barging is possible:** a new thread can become leader before the notified follower wakes. That's correct, just not FIFO.

**Fairness (strict FIFO) with tickets**

```cpp
uint64_t nextTicket = 0, nowServing = 0;

void acquire() {
    unique_lock<mutex> lock(m);
    uint64_t my = nextTicket++;
    cv.wait(lock, [&]{ return my == nowServing; });
    while (!tryAcquireLocked(steady_clock::now()))
        cv.wait_until(lock, dq.front() + window);
    ++nowServing;
    cv.notify_all();   // small herd at handoff; per-waiter CVs remove it
}
```

**`tryAcquire()` can starve waiters.** It skips the leader/follower handoff. If it takes the slot the leader was waiting for, the leader wakes, fails, recomputes the next `front + window`, and sleeps again. Under heavy load this can repeat forever.

| Option | How | Trade-off |
| --- | --- | --- |
| Allow it | Leave `tryAcquire()` as is | Max throughput; blocked threads can starve |
| Waiters get priority | `if (hasLeader) return false;` in `tryAcquire()` | No starvation; `tryAcquire()` refuses slightly more often |

## Condition variable and mutex mechanics

A condition variable doesn't watch your variables, and every deque access happens with the mutex held.

**1. Changing a flag doesn't wake anyone.** `dv.wait(lock)` sleeps until someone calls `notify` on `dv`. Setting `tot = 0` does nothing by itself, so `dv.notify_one()` must follow it. The `while` loop is still needed to re-check after spurious wakeups or barging.

**2. Change the flag under the mutex, or risk a lost wakeup.** Without the lock: a follower checks `tot == 1`, you set `tot = 0` and notify, and only then does the follower go to sleep. It misses the notify and sleeps forever.

**3. `wait_until` releases and reacquires the lock for you.**

1. Before sleeping, the leader holds the lock while `dq.front() + window` is evaluated.
2. While sleeping, `wait_until` atomically releases the mutex. That's how a `tryAcquire()` thread can get in.
3. On waking (timeout, notify or spurious), it reacquires the mutex before returning.

```text
Leader                                    tryAcquire() thread
──────                                    ───────────────────
holds lock
reads dq.front() → deadline 11.2s
wait_until: releases lock, sleeps
                                          takes lock
                                          evicts, pushes 11.19s → slot taken
                                          releases lock
wakes at 11.2s, reacquires lock
tryAcquireLocked() fails (under lock)
reads new dq.front() (under lock)
wait_until: releases lock, sleeps again
```

That's why `wait`/`wait_until` take a `unique_lock`: a `lock_guard` can't be unlocked and relocked.

**4. Mutual exclusion covers code holding the lock, not one function.** Only one thread at a time can be anywhere that holds `m`: the leader checking for a slot, a direct `tryAcquire()` caller, a new thread entering `acquire()`, or a follower re-checking its flag.

**5. Waking up isn't the same as continuing.** If the leader's deadline passes while another thread holds `m`, the OS wakes the leader, but it blocks inside `wait_until` until the lock is released. It runs nothing past `wait_until` until then, which is only a few microseconds.

```text
11.2s: leader wakes → tries to lock m → BLOCKED (tryAcquire thread holds it)
       tryAcquire thread releases m
       leader gets m → wait_until returns → tryAcquireLocked()
```

## Distributed extension (10 servers)

The usual answer is to move the sliding-window log into Redis and do evict + count + add atomically in a Lua script, the same idea as the mutex.

**Limiting at a load balancer** is a valid pattern (API gateways do it), but it has gaps:

- The limit is on **outbound** calls. One inbound request may make 0, 1 or 5 calls, and background jobs never pass through the load balancer.
- It's a single point of failure. Running several load balancers brings back the shared-counter problem.

| Approach | How | Trade-off |
| --- | --- | --- |
| Shared store (Redis) | Sorted set of timestamps per key; atomic Lua script | Extra network hop per call; decide fail-open vs fail-closed |
| Static partitioning | Each server gets N/10, local limiter | Simple, no network; wastes capacity when traffic is uneven |
| Token leasing | Servers fetch token batches from a coordinator | Fewer round-trips; more complex |

Avoid clock skew between servers by using Redis's own clock (`TIME`) inside the script.

## Feedback and key terms

Overall: lean hire for a mid-level role. The core concurrency reasoning was solid, and the gaps can be fixed with practice.

**Strengths**

- Asked rolling vs. fixed window up front.
- Chose a correct sliding-window log with a deque.
- Spotted the race and justified one lock over split locks.
- Found `wait_until(front + window)` and reasoned that no `notify` is needed.
- Came up with leader/follower without knowing the name.

**To work on**

- [ ] Lock scope: keep the lock in public methods; locks guard data, not functions.
- [ ] Waiting for a lock vs. waiting for a condition; sleeping threads use no CPU.
- [ ] Learn the pattern names so answers land faster.
- [ ] Practice the Redis + atomic script answer for the distributed follow-up.

| Term | Meaning |
| --- | --- |
| Check-then-act race | A check and the action based on it aren't atomic, so another thread acts in between |
| Lost wakeup | A notify fires before the waiter sleeps, so it sleeps forever |
| Spurious wakeup | A condition variable wait returns without a notify; always re-check in a loop |
| Thundering herd | Many threads wake for one resource; one wins, the rest waste the wakeup |
| Leader/follower | One thread does the timed wait; others wait untimed and are promoted one at a time |
| Barging | A newly arrived thread takes a resource ahead of a notified waiter |
| Starvation | A thread waits indefinitely because others keep winning |
