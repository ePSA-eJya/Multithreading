# Playground Access Control: My Thought Process, Step by Step

> A revision journal of how I solved this in the mock interview: what I thought at each stage, where it broke, and what fixed it. Read it top to bottom to replay the reasoning.

## Problem

- Only **one team** in the playground at a time.
- At most **10 players** of that team inside at once.
- Players from different teams arrive **concurrently**.
- Follow-up: make sure **no team and no player starves**.

## 30-second recall (read this first when revisiting)

| Stage | My idea | What broke | Fix / lesson |
|---|---|---|---|
| 1 | "Lock the playground" | Holding a lock while playing allows only one player | Hold the lock briefly to update state, not while playing |
| 2 | Track `count` + `currentTeam` | (none, this is correct) | That's the whole state for the baseline |
| 3 | Check under lock 1, increment under lock 2 | Two players both see 9 and you get 11 inside | **Check + update in one critical section, one lock** |
| 4 | `while (t != id \|\| curr >= 10) wait` | First player waits forever (`t == -1 != id`) | Allow entry when empty: `(t != -1 && t != id)` |
| 5 | Spotted starvation | Team A keeps arriving, `curr` never hits 0 | Need to track **who's waiting**, not just who's inside |
| 6 | "Block the same team from going twice in a row" | Doesn't stop the never-empties case; idles the playground; C can starve | A "not last time" rule is not the same as "everyone eventually gets a turn" |
| 7 | Turnstile mutex | Released it immediately, so it did nothing | **Hold the turnstile until admitted** |
| 8 | Ticket system (FIFO) | Caller passed the ticket; `nowserv++` in leave; dropped notify in leave | Assign tickets inside; advance on admission; **every state change must notify** |
| 9 | Noticed head-of-line blocking | Strict FIFO wastes free slots | Fairness vs. utilization trade-off |
| 10 | Round-robin across per-team queues | When does a turn end? What if the next team is empty? | **Quota K** + rotation of only teams that have waiters |

---

## Stage 0: Clarifying question

**What I asked:** once players of the current team start leaving, can more teammates still enter, or must the playground empty first?

**Answer:** teammates can keep entering while there's space. The playground switches teams only when it's empty.

**Why it mattered:** this rule is exactly what later causes **team starvation**. Good clarifying questions find the rule that the hard part depends on.

---

## Stage 1: "Lock the playground"

**What I thought:** the playground is the shared resource. Lock it when a team enters so no other team can, and also when the count reaches 10.

**What was wrong:** if a player takes a lock and **holds it while playing**, a teammate can't get in either, so only 1 player is inside, not 10.

**Lesson:** the lock protects **the bookkeeping variables**, not the playground itself. Take it, update state, release it, then go play.

---

## Stage 2: Pick the state

**What I thought:**

- `count`: how many players are inside
- `currentTeam`: which team is inside (`-1` when empty)

**Verdict:** correct. This is all the baseline needs.

---

## Stage 3: Check-then-act across two locks

**What I thought:** grab the playground lock and check the condition. Then release it, grab a *counter* lock, and increment.

**What was wrong:** 9 Team A players are inside.

1. A10 locks, sees 9, unlocks.
2. A11 locks, sees 9, unlocks.
3. Both increment: **11 inside**.

Splitting the check and the update across two locks reopens the race.

**Also settled here:**

- Set `currentTeam = myTeam` when the playground is empty, and reset it to `-1` when the last player leaves.
- Use `notify_all`: when the playground empties, up to 10 players of the next team may need to wake.
- After waking, **re-check the condition** in a `while` loop, because state may have changed and spurious wakeups exist.

**Lesson:** **one lock, one critical section for check + increment + team assignment.**

---

## Stage 4: First code (and its deadlock)

### What I wrote

```cpp
void enter(int id){
    unique_lock<mutex> lock(p);
    while (t != id or curr >= 10)      // BUG
        cv.wait(lock);
    curr++;
    if (curr == 1) t = id;
}
```

### What was wrong

Trace the first player on an empty playground: `t = -1`, `id = 5`, so `t != id` is **true**, and the player waits forever. Nobody can ever enter.

### Fix

```cpp
while ((t != -1 && t != id) || curr >= 10)
    cv.wait(lock);
```

Wait only if **another** team holds the playground or it's full.

### Baseline V1 (correct, but can starve)

```cpp
class Playground {
    int curr = 0, t = -1;
    mutex p;
    condition_variable cv;
public:
    void enter(int id) {
        unique_lock<mutex> lock(p);
        while ((t != -1 && t != id) || curr >= 10) cv.wait(lock);
        if (curr++ == 0) t = id;
    }
    void leave() {
        lock_guard<mutex> lock(p);
        if (--curr == 0) t = -1;
        cv.notify_all();
    }
};
```

**Invariant:** `t == -1` exactly when `curr == 0`, because both are always updated together under `p`.

**Small C++ points I missed:** `class` members are private by default (add `public:`), the class needs its trailing `;`, and an explicit `unlock()` isn't needed because RAII releases the lock.

**Lesson:** **trace the very first player and the empty case before saying you're done.**

---

## Stage 5: Finding starvation

**What I found:**

- **Team starvation:** when the last A player leaves and everyone is notified, an A player can win the race and claim the playground again.
- **Player starvation:** wake-up order is random, so a later arrival can enter before an earlier one.

**The bigger case (the interviewer added this):** if A players keep arriving, `curr` stays between 1 and 10 and **never reaches 0**. `t` never resets, and Team B waits forever even though the playground is always busy.

**Lesson:** V1 only tracks **who's inside**. To be fair, the system has to know **who's waiting and in what order**.

---

## Stage 6: Idea 1, "Don't let the same team go twice in a row"

**What I thought:** remember the previous team and refuse it if it tries to claim again.

**Why it's not enough:**

1. The rule is checked only when the playground switches teams, and in the bad case **it never empties**.
2. If A is the **only** team waiting, refusing A leaves the playground idle for no reason.
3. With 3+ teams, A and B can alternate forever and **C starves**.

**Lesson:** "not the same as last time" is not the same as **"everyone eventually gets a turn"**. A good scheduler should also be **work-conserving**: if someone is waiting and there's space, let them in.

---

## Stage 7: Idea 2, the turnstile

### My first version (didn't work)

B1 grabs the turnstile and **releases it immediately**, then waits.

**Problem:** A12 arrives, walks through the free turnstile, sees `t == A`, and enters. Team A never drains, so the turnstile did nothing.

### What confused me, and the answers

**Q: When do we lock the turnstile? Only when the playground is full or taken?**

**A:** No. **Every player locks it first, unconditionally.** To know the playground is full you'd have to read the state, and doing that before the turnstile means you've already skipped the line.

**Q: After B1 enters, A12 also tries to grab the turnstile, right?**

**A:** Yes. A12, B2, B3, ... all compete. If B2 wins, it enters. If A12 wins, it waits while holding the turnstile, and now **A12 is the barrier**, so Team B drains. Teams naturally alternate.

### The key rule

> **Hold the turnstile until you are admitted.**

| Step | What happens |
|---|---|
| 1 | B1 locks turnstile → locks `p` → sees `t == A` → `cv.wait()` (releases `p`, **keeps the turnstile**) |
| 2 | A12 blocks at the turnstile and never reaches the check |
| 3 | A players inside call `leave()` (which doesn't need the turnstile), and the playground empties |
| 4 | B1 wakes, claims `t = B`, enters, releases the turnstile |

### My code

```cpp
void enter(int id){
    lock_guard<mutex> door(turnstile);
    unique_lock<mutex> lock(p);
    while ((t != -1 && t != id) || curr >= 10) cv.wait(lock);
    curr++;
    if (curr == 1) { t = id; door.unlock(); }   // BUG: lock_guard has no unlock()
}
```

**Fix:** delete `door.unlock()`. The `lock_guard` releases the turnstile when `enter()` returns, which is exactly when the player is admitted.

### Fixed V2 (turnstile, complete)

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

**Q: If B1 holds the turnstile, B2 (same team) can't enter either until B1 releases it, right?**

**A:** Right, but B1 releases it as soon as it's **admitted**, not when it leaves the playground. B2 is only held up while B1 waits for Team A to drain, and B2 couldn't have entered then anyway. Once B1 is in, B2, B3, ... pass through the turnstile one after another in microseconds. Players go through the door **one at a time**, but quickly.

**Deadlock check:** the order is always `turnstile → p`, and `leave()` only takes `p`, so there's no circular wait.

**Remaining flaw:** `std::mutex` isn't FIFO, so a player can keep losing the turnstile race.

---

## Stage 8: Idea 3, a FIFO queue, built as a ticket system

My idea was "keep a queue." The cheapest version is a deli-counter ticket system: `nextTicket` and `nowServing`.

### Attempt 1

```cpp
void enter(int id, int nexttick) {          // BUG 1: caller supplies the ticket
    unique_lock<mutex> lock(p);
    while ((t != -1 && t != id) || curr >= 10 || nexttick != nowserv)
        cv.wait(lock);
    ...
}
void leave() {
    ...
    nowserv++;                               // BUG 2: advances on leave
    cv.notify_all();
}
```

- **Bug 1:** duplicate or skipped tickets mean everyone waits for a number that never comes, so the system **deadlocks**.
- **Bug 2:** A1 (ticket 1) enters. A2 (ticket 2) can't enter until A1 **leaves**, so only **1 player is ever inside**.

### Attempt 2

```cpp
void enter(int id){
    unique_lock<mutex> tick(tt);
    int my = nexttick++;                     // fixed: ticket assigned inside
    tick.unlock();

    unique_lock<mutex> lock(p);
    while ((t != -1 && t != id) || curr >= 10 || my != nowserv) cv.wait(lock);
    if (curr++ == 0) t = id;
    nowserv++;                               // fixed: advance on admission
    cv.notify_all();                         // next ticket may enter right away
}
void leave(){
    unique_lock<mutex> lock(p);
    if (--curr == 0) t = -1;
                                             // BUG 3: no notify
}
```

**Bug 3:** 10 A players are inside and A11 holds the next ticket, waiting for space. A1 leaves, but nobody is notified, so A11 sleeps forever and every later ticket is stuck behind it.

### My question: "When `curr` becomes 9, the while condition is false, so won't the waiting thread just continue?"

**No.** A thread inside `cv.wait()` is **asleep**. It doesn't poll or watch variables. It re-checks the `while` condition **only after someone notifies it**.

```
check → true → wait() → [ASLEEP until notified] → wake → check again
```

Analogy: napping in a waiting room. A table freeing up doesn't wake you. The host has to tap your shoulder.

> **Rule: whoever changes state that a waiter depends on must notify.**

### Fixed V3 (strict FIFO, starvation-free)

```cpp
class Playground {
    int curr = 0, t = -1, nowserv = 0;
    atomic<int> nexttick{0};
    mutex p;
    condition_variable cv;
public:
    void enter(int id) {
        int my = nexttick.fetch_add(1);
        unique_lock<mutex> lock(p);
        while ((t != -1 && t != id) || curr >= 10 || my != nowserv) cv.wait(lock);
        if (curr++ == 0) t = id;
        nowserv++;
        cv.notify_all();
    }
    void leave() {
        lock_guard<mutex> lock(p);
        if (--curr == 0) t = -1;
        cv.notify_all();
    }
};
```

---

## Stage 9: The cost of strict FIFO

**Scenario:** 3 A players inside, then B1 in line, then A4–A10 behind B1.

**What I said:** A4–A10 could all have played together, but they wait for B1, so the playground is underused.

**Two periods of waste:**

1. **Draining:** only 3/10 slots are used while B1 waits, and A4–A10 can't get past B1.
2. **B1 alone:** 9/10 slots sit empty while B1 plays by itself.

This is **head-of-line blocking**: strict FIFO gives the most fairness but costs throughput.

| V1 baseline | V3 strict FIFO |
|---|---|
| High utilization, can starve | No starvation, poor utilization |

The goal is a middle ground.

---

## Stage 10: Final idea, round-robin over per-team queues

### Step 10a: what I proposed

Keep **one queue per team** and give teams turns **round-robin**. When a team finishes, the turn moves to the next team that has players waiting.

**Why it's good:**

- **Team fairness:** every team gets a turn in rotation.
- **Player fairness:** FIFO within each team's queue.
- **Bonus:** use one condition variable per team and wake only the team whose turn it is, which avoids waking every thread on every exit (the "thundering herd").

**Gap:** when is a team's turn over? If A's queue keeps refilling, it never "completes," and we're back to V1's starvation.

### Step 10b: add a quota K

A team may admit **up to K players** per turn. Then it stops admitting, drains, and the next team goes.

### Step 10c: the edge case I raised, "what if the next team is empty?"

We shouldn't wait on a team that has nobody waiting. My fix was **"a FIFO of queues"**: keep a rotation list containing **only teams that have waiters**.

- A team joins the back of the rotation when its first player starts waiting.
- It leaves the rotation when it gets the playground (and rejoins at the back if players remain when its turn ends).

### Final rules

**A player may enter if** it's at the head of its team's queue, **and** there's space, **and** either:

- the playground is empty and its team is at the front of the rotation, or
- its team is current and the turn hasn't closed.

**A turn closes when** another team is waiting **and** either the team has admitted K players or its own queue is empty. If nobody else is waiting, the team keeps going, so the playground never sits idle while someone wants it (**work-conserving**).

**Wake-ups:**

- After admission: wake own team (the next teammate may fit).
- On leave, turn still open: wake own team (a slot opened).
- On leave, playground now empty and turn over: wake the team at the front of the rotation.

### Why it's starvation-free (say this out loud)

- Each turn admits at most K players once others are waiting, and players inside eventually leave, so **every turn ends**.
- A waiting team gets the playground after at most **N − 1 other turns** (N = number of teams).
- Within a team it's FIFO, so a player at position *p* waits about ⌈p / K⌉ of their team's turns.
- Every wait is **bounded**, so nothing starves.

**Choosing K:** a bigger K means fewer team switches and better throughput but longer waits for other teams. A smaller K is the reverse.

Full tested implementation: [`playground.cpp`](playground.cpp)

---

## Lessons to carry into the next interview

1. **The lock guards the state, not the resource.** Hold it briefly.
2. **Check and update in one critical section.** Never split them across locks.
3. **`while` around `wait`, always.**
4. **A waiting thread is asleep.** Every state change a waiter depends on must notify.
5. **Trace 2–3 players**, including the very first one on an empty system, before saying "done."
6. **Starvation means tracking who's waiting**, not just who's inside.
7. **A turnstile only works if you hold it until admitted.**
8. **Tickets:** assign them inside, advance on admission, notify after advancing.
9. **Fairness costs throughput.** Know the trade-off and propose a middle ground (quota + round-robin).
10. **Say these out loud:** lock ordering ("no circular wait"), the starvation bound ("at most N − 1 turns"), and work-conservation.

**Related problems to practice:** unisex bathroom, readers-writers (all 3 variants), H2O, and barriers, from *The Little Book of Semaphores*.
