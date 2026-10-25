# Concurrency Mock Interview: Bathroom & H2O

Oct 24, 2026 · @Epsa Sharma

## Overview

Two mock interview rounds on multithreading and concurrency: the shared bathroom (mutual exclusion plus fairness) and H2O (a reusable barrier). Each round lists the problem, the bugs found along the way, the final solution, and the reasoning an interviewer expects you to say aloud.

The main lesson from both rounds: trace every wait condition literally, hold one lock across each check-then-act, and wake every thread whose wait condition you just changed.

## Round 1: the shared bathroom

One bathroom is shared by Democrats (D) and Republicans (R). It holds at most 3 people, and the two groups are never inside together. People wait in a queue, and each one takes a variable time f(N) inside. Each person is a thread that calls `enter(group)` and `exit()`.

### Clarified requirements

| Question asked | Answer |
| --- | --- |
| When one of 3 leaves, can a same-group person refill the slot? | Yes. No batch draining, except when the fairness policy needs it. |
| Must groups take strict turns? | No. Prevent starvation, but if only one group is waiting it gets full use. |
| FIFO order within a group? | Not required. Waiting time must be bounded; FIFO is a plus. |

This gives three properties to check every design against:

- **Safety:** at most 3 inside, never mixed groups.
- **Liveness:** no starvation for either group.
- **Efficiency:** no idle capacity when only one group is waiting.

## Round 1 walkthrough

The state is `count`, `currentGroup`, one mutex and one condition variable. The person uses the bathroom outside the lock. Each step below is a bug found by tracing a scenario.

| Step | Draft | Scenario that broke it | Fix |
| --- | --- | --- | --- |
| 1 | Wait while `currentGroup != my AND count == 3` | A: 1 D inside, R arrives. `true AND false` is false, so R walks in and groups mix. | Use OR. Two independent reasons to wait. |
| 2 | Notify only when `count == 0` | B: 3 D inside, D waiting, one leaves. Nobody is woken, so a slot sits idle until the room empties. | Notify on every exit. |
| 3 | Wait while `currentGroup != my OR count == 3` | C: empty room, `currentGroup = NONE`, D arrives. `NONE != D` is true, so D waits forever. Nobody is inside to wake it. | Only compare groups when occupied: `currentGroup != NONE AND currentGroup != my`. |
| 4 | Safe code, no fairness | Rs keep refilling freed slots, the count never hits 0, and a waiting D starves. | Turnstile (below). |

### Turnstile

A second lock taken at the start of `enter()` and held while waiting on the condition variable. A waiting D blocks every later arrival at the gate, so the bathroom drains and D gets in. When only one group arrives, the gate passes quickly and capacity stays full.

- **Deadlock:** `exit()` must never take the turnstile. The waiter holds it while waiting for an exit, so an exit needing it is a circular wait.
- **Fairness:** `new ReentrantLock()` is non-fair, so a new arrival can barge past queued threads. Use `ReentrantLock(true)`, `Semaphore(1, true)` or a ticket counter. Fair locks cost throughput.
- **Cost:** during a group switch, same-group arrivals queue behind the other group at the gate while slots sit empty. That idle capacity is the price of fairness, and it beats starvation.

### notify vs notifyAll

With the turnstile, only the gate holder can be waiting on the condition variable, so `notify()` and `notifyAll()` behave the same. Without the turnstile, `notify()` can cause a lost wakeup: 3 D inside, an R and a D waiting, one D leaves, `notify()` wakes the R, which goes back to sleep. The D that could have entered is never woken. Fix with `notifyAll()` or one condition variable per group.

## Round 1 final solution

Safe, starvation-free and full capacity when only one group is waiting, assuming the turnstile is a fair lock.

```
enter(myGroup):
  turnstile.lock()            // fair lock; held while waiting
  lock
  while (count == 3 OR (currentGroup != NONE AND currentGroup != myGroup)):
      wait(cv)                // releases `lock`, still holds `turnstile`
  count++
  currentGroup = myGroup
  unlock
  turnstile.unlock()
  // use bathroom: f(N), outside both locks

exit():
  lock                        // never takes the turnstile
  count--
  if count == 0:
      currentGroup = NONE
  notifyAll(cv)
  unlock
```

## Round 2: building H2O

Hydrogen threads call `hydrogen(releaseHydrogen)` and oxygen threads call `oxygen(releaseOxygen)`. Each thread releases once, and threads must pass in groups of exactly 2 H and 1 O.

- A thread can't pass until the rest of its molecule has arrived.
- Molecules can't interleave: all 3 threads of one molecule release before any thread of the next.
- Order inside a molecule doesn't matter. Input `OOHHHH` can produce `HHOHHO` or `OHHHHO`, but not `HHHHOO`.
- Input has exactly 2N hydrogens and N oxygens.

Clarified: there are two separate functions, so each admission check uses only its own atom's count.

## Round 2 walkthrough

The design has two stages. **Admission** lets at most 2 H and 1 O into the current molecule. The **barrier** holds the 3 admitted threads until all have arrived, then releases them together.

State: `h`, `o` (admitted, the gate), `bh`, `bo` (at the barrier), a completion flag `com`, one mutex, and two condition variables: `cv` for admission, `gv` for the barrier.

| Step | Draft | Scenario that broke it | Fix |
| --- | --- | --- | --- |
| 1 | Barrier waits while `bh != 2 AND bo != 1` | 2 H waiting, no O: `false AND true` is false, so both H pass without an O. | `while not (bh == 2 and bo == 1)` |
| 2 | Barrier waits on the counts themselves | H1 leaves first and decrements `bh` to 1. H2 re-checks, sees 1 and sleeps forever. Admission never reopens: deadlock. | Record the event with a flag `com`. Departing threads don't undo it. |
| 3 | Each leaver resets `com` | The first leaver clears it, and H2 wakes to `false`: the same deadlock. | Only the last thread out (`bh == 0 && bo == 0`) resets `com`, `h` and `o`, then notifies `cv`. |
| 4 | No `gv.notify_all()` when `com` is set | Barrier waiters are never woken. | Notify in the same section that sets `com`. |
| 5 | Unlock between `bh--` and the last-thread check | H2 and O1 both see `bh == 0 && bo == 0` and both reset, so `h` goes to -2 and 4 H get admitted. | Decrement and check in one critical section. |

### Why the stale flag is safe

`com` stays true while molecule 1 drains, but the gate is still closed (`h == 2`, `o == 1`). It reopens only in the same critical section that clears `com`, so no molecule-2 thread can see a stale true.

### Why the release goes before the decrement

Suppose release moved after `bh--`, outside the lock. H1 and H2 decrement, then get preempted before releasing. O1 is last out, resets the gate and releases `O`. Molecule 2 forms, its two H get preempted the same way, and O2 releases. The output starts `OO`, which is not a valid molecule. The gate must reopen only after every thread has actually released.

## Round 2 final solution

This is your final version with the polish applied: the redundant unlock/lock pair is removed and `comp` is renamed to `com`. One lock is held for the whole function, and only the waits give it up.

```cpp
int h = 0, o = 0;       // admitted to the current molecule (the gate)
int bh = 0, bo = 0;     // currently at the barrier
bool com = false;       // current molecule is complete
mutex gm;
condition_variable cv;  // admission
condition_variable gv;  // barrier

void hydrogen(function<void()> releaseHydrogen) {
    unique_lock<mutex> gl(gm);
    while (h >= 2) cv.wait(gl);        // stage 1: admission, own count only
    h++;
    bh++;
    if (bh == 2 && bo == 1) {           // this thread completes the molecule
        com = true;
        gv.notify_all();
    }
    while (!com) gv.wait(gl);           // stage 2: barrier on the event, not the counts
    releaseHydrogen();                  // release BEFORE the decrement
    bh--;
    if (bh == 0 && bo == 0) {           // last thread out resets everything
        com = false;
        h -= 2;
        o -= 1;
        cv.notify_all();                // reopen admission
    }
}

// oxygen() mirrors hydrogen(): wait while (o >= 1), increment o and bo,
// call releaseOxygen(), decrement bo. Completion check and reset are identical.
```

### Two-lock alternative

A variant uses `m` for admission (`h`, `o`) and a separate `bm` for the barrier (`bh`, `bo`, `comp`), then takes `m` again for the exit section.

```cpp
void hydrogen(function<void()> releaseHydrogen) {
    unique_lock<mutex> lock(m);
    while (h >= 2) cv.wait(lock);
    h++;
    lock.unlock();

    unique_lock<mutex> block(bm);
    bh++;
    if (bh == 2 && bo == 1) {
        comp = true;
        bv.notify_all();
    }
    while (!comp) bv.wait(block);
    block.unlock();

    lock.lock();
    releaseHydrogen();
    bh--;
    if (bh == 0 && bo == 0) {
        comp = false;
        h -= 2;
        o -= 1;
        cv.notify_all();
    }
}
```

**It is correct**, because the gather and exit phases never overlap:

- No thread reaches the exit section until `comp` is true. The thread that set it had already finished its `bh++` or `bo++` under `bm`.
- The last thread out resets under `m`. A molecule-2 thread must take `m` to be admitted, so it sees the reset state.
- The two locks are never held together, so there is no lock-ordering deadlock.

**It does the same work as one lock.** A thread that reaches the barrier releases its lock while it waits, whether that lock is `m` or `bm`. What's left in the barrier section is a few increments and a flag check, and `releaseHydrogen()` still runs under `m`. So the second lock changes nothing about who can make progress.

**It is fragile.** `bh`, `bo` and `comp` are each guarded by different locks at different times. That is safe only because of the phase ordering above, so a later read of `bh` added under the wrong lock becomes a data race.

**Interview answer:** "I'd use one lock. `wait()` already releases it, so splitting gives no contention benefit. A second lock pays off only when the protected work is slow and independent, and here it is neither."

## Feedback and what to practice

The concurrency instincts are solid. The gaps are boolean precision and thinking about preemption, and both showed up in both rounds.

**Strengths**

- You asked sharp clarifying questions and raised starvation before the interviewer did.
- You found the turnstile on your own and reasoned correctly about barging and fair locks.
- You saw that the turnstile leaves at most one waiter, so `notify()` is safe there.
- You split H2O into admission plus a barrier, and moved from a hint to the completion flag.
- You fixed three critical bugs in one revision.

**Habits to drill**

- [ ] Say every wait condition aloud as "I wait if \_\_\_ or \_\_\_" before writing it. The AND/OR slip happened in both rounds.
- [ ] Trace the expression literally against a concrete state, not the way you intended it.
- [ ] Hold one lock across each check-then-act, and let only `wait()` release it.
- [ ] After every state change, ask "who is sleeping on this, and did I wake them?"
- [ ] At every gap between unlock and the next action, ask "what if I'm preempted here?"
- [ ] Write the code rather than describing it, and name the tradeoff before you're asked.

**Next rounds:** a bounded blocking queue, to drill notify discipline, and readers-writers with writer priority, to drill fairness.
