# Playground Access Control: Interview Write-up

## Problem

- Only **one team** may use the playground at a time.
- At most **10 players** from that team may be inside at once.
- Players from many teams call `enter(team)` / `leave(team)` concurrently.
- Follow-up: **no team and no player may starve**.

Close relatives: the *unisex bathroom* problem and *readers-writers* (where "a team" plays the role of "readers of one kind").

---

## The progression (how to talk through it in an interview)

### V1: Baseline (correct, but can starve)

State: `curr` (players inside) and `t` (team inside, `-1` = empty), guarded by **one** mutex.

```cpp
void enter(int id) {
    unique_lock<mutex> lock(p);
    while ((t != -1 && t != id) || curr >= 10)
        cv.wait(lock);
    if (curr++ == 0) t = id;
}
void leave() {
    lock_guard<mutex> lock(p);
    if (--curr == 0) t = -1;
    cv.notify_all();
}
```

Key rules this gets right:

| Rule | Why |
|---|---|
| Check **and** update under the same lock | Otherwise two players both see `curr == 9` and you end up with 11 inside (check-then-act race). |
| Hold the lock briefly, not while playing | Holding it while playing would allow only one player at a time. |
| `while`, not `if`, around `wait` | State may change before the woken thread gets the lock back, and spurious wakeups are allowed. |
| `notify_all`, not `notify_one` | When the playground empties, up to 10 players of the next team may need to wake. One `notify_one` could wake the wrong thread. |
| **Whoever changes state a waiter depends on must notify** | A waiting thread is asleep and only re-checks its condition when notified. |

**Starvation:** if Team A players keep arriving, `curr` never reaches 0, `t` never resets, and Team B waits forever. Individual players can also starve, because neither `notify_all` nor `std::mutex` preserves arrival order.

### V2: Turnstile (team starvation fixed)

Every player locks a `turnstile` mutex **first** and holds it **until admitted**. A B1 waiting at the front of the line blocks new Team A arrivals at the turnstile, so Team A drains and B gets its turn.

```cpp
class Playground {
    int curr = 0, t = -1;
    mutex turnstile, p;
    condition_variable cv;
public:
    void enter(int id) {
        lock_guard<mutex> door(turnstile);      // everyone lines up here first
        unique_lock<mutex> lock(p);
        while ((t != -1 && t != id) || curr >= 10)
            cv.wait(lock);                      // releases p, KEEPS the turnstile
        if (curr++ == 0) t = id;
    }                                           // admitted: door + lock released here
    void leave() {                              // never touches the turnstile
        lock_guard<mutex> lock(p);
        if (--curr == 0) t = -1;
        cv.notify_all();
    }
};
```

- Lock order is always `turnstile` → `p`, and `leave()` only takes `p`, so there's no circular wait and no deadlock.
- The turnstile is held until **admitted**, not until the player leaves, so teammates queued behind pass through in microseconds once the holder gets in.
- The flaw: `std::mutex` isn't FIFO, so an individual player can still keep losing the race.

### V3: Ticket lock (strict FIFO)

`my = nextTicket++` on arrival. The player waits until `my == nowServing` *and* the team/capacity checks pass. Advance `nowServing` **on admission** and notify then too, because the next ticket holder may be able to enter immediately.

```cpp
class Playground {
    int curr = 0, t = -1, nowserv = 0;
    atomic<int> nexttick{0};
    mutex p;
    condition_variable cv;
public:
    void enter(int id) {
        int my = nexttick.fetch_add(1);         // ticket assigned inside, atomically
        unique_lock<mutex> lock(p);
        while ((t != -1 && t != id) || curr >= 10 || my != nowserv)
            cv.wait(lock);
        if (curr++ == 0) t = id;
        nowserv++;                              // advance on ADMISSION
        cv.notify_all();                        // next ticket may enter right away
    }
    void leave() {
        lock_guard<mutex> lock(p);
        if (--curr == 0) t = -1;
        cv.notify_all();                        // head of line may be waiting for space
    }
};
```

- Bugs fixed during the interview: the caller passed in the ticket (duplicates or gaps cause deadlock); `nowServing` advanced in `leave()` (only one player inside at a time); the notify was missing in `leave()` (the head of the line slept forever).
- The flaw: **head-of-line blocking.** With 3 Team A players inside, then B1, then A4–A10 in line, A4–A10 can't use the 7 free slots. Then B1 plays alone while 9 slots sit empty. No starvation, but poor utilization.

### V4: Round-robin over per-team queues with a quota (final)

This keeps V1's throughput and V3's fairness.

---

## Final design

**State (all guarded by one mutex):**

| Field | Meaning |
|---|---|
| `teams_[team].queue` | FIFO of waiting players' tickets for that team |
| `teams_[team].cv` | per-team condition variable, so we wake only the team that can act |
| `rotation_` | round-robin order of teams **that have waiters** (never contains the current team while its turn is open) |
| `current_`, `inside_` | team holding the playground and how many are inside |
| `admittedThisTurn_` | admissions in the current turn |
| `closing_` | the current turn has ended: admit nobody new, just drain |

**A player can enter if:**

1. it is at the **head of its team's queue** (FIFO within team), **and**
2. `inside_ < capacity`, **and**
3. either the playground is **empty and its team is at the front of `rotation_`**, or its team is **current and the turn is not closing**.

**A turn closes when** another team is waiting **and** either:

- the current team has admitted **K** (quota) players this turn, or
- the current team has **no one left waiting**.

If no other team is waiting, the current team keeps admitting past K, so the playground never sits idle while someone wants it (**work-conserving**). On close, any remaining waiters from the current team go to the **back** of the rotation.

**Wake-ups (targeted, no thundering herd):**

- After admission: notify own team (the next teammate may fit).
- On leave, turn still open: notify own team (a slot opened up).
- On leave, playground now empty and turn over: notify the team at `rotation_.front()`.

Full code with a stress test: [`playground.cpp`](playground.cpp). It was verified with 60 threads × 300 entries across 5 teams: 0 invariant violations, clean under ThreadSanitizer, worst observed wait about 10 ms.

---

## Correctness argument

**Safety**

- `current_` changes only when `inside_ == 0`, and admission requires `team == current_` or an empty playground, so **only one team is ever inside**.
- The capacity check and `++inside_` happen in the same critical section, so there are **never more than 10 inside**.

**Liveness (starvation-freedom)**

- Every turn admits at most K players once another team is waiting, and every player inside eventually leaves, so **every turn ends in finite time**.
- Teams are served in rotation order and a team joins at the back, so a waiting team gets the playground after at most **N − 1 other turns** (N = number of teams).
- Within a team, players are served FIFO by ticket. A player at position *p* in their team's queue waits at most about ⌈p / K⌉ of their team's turns.
- So every player's wait is **bounded**, which means the design is starvation-free.

---

## Trade-offs

| Design | Utilization | Team fairness | Player fairness | Complexity |
|---|---|---|---|---|
| V1 baseline | High | Can starve | Can starve | Low |
| V2 turnstile | Medium | Good | Not guaranteed (mutex isn't FIFO) | Low |
| V3 ticket lock | Low (head-of-line blocking) | Strict FIFO | Strict FIFO | Low |
| **V4 round-robin + quota** | **High** | **Bounded wait** | **FIFO within team** | Medium |

Choosing **K** trades **throughput** against **latency**. A larger K means fewer team switches (less draining overhead) but longer waits for other teams. A smaller K is the reverse. A time-based limit (each turn gets at most T seconds of admissions) is a reasonable alternative when play durations vary a lot.

## Extensions to mention if time permits

- **Timeouts and cancellation** (`wait_for`): a player who gives up must remove its ticket and, if it was at the head of its team's queue, wake the next teammate.
- **Priority teams:** weighted round-robin, e.g. a quota proportional to team weight (as in Deficit Round Robin).
- **Distributed version:** keep the state in a single coordinator (or Redis with Lua/transactions, or ZooKeeper/etcd with leases) so crashed players' slots are reclaimed when their leases expire.
- **Memory:** `teams_` grows with every team ever seen. Erase entries with empty queues if team IDs are unbounded.

## Interview habits from this session

1. Trace 2–3 players through your code out loud before saying you're done.
2. For every state change, ask who might be waiting on it and make sure they get notified.
3. Say lock ordering and the starvation bound out loud without being asked.
