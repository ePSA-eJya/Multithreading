# Mock Interview: Task Scheduling & Load Rebalancing

Oct 25, 2026 · @Epsa Sharma

## Problem statement

Design a thread-safe scheduler for N worker nodes, each with its own task queue, exposing four APIs.

| API | Behavior |
| --- | --- |
| `addTask(nodeId, taskId)` | Add a task to the given node's queue. |
| `runTask(nodeId)` | Take the next task from that node's queue and run it. |
| `rebalanceTask()` | Move tasks from overloaded nodes to underloaded ones. |
| `printTask()` | Print the tasks queued on each node. |

Assumptions settled during the interview:

- All tasks cost the same, so load = number of pending tasks.
- Many producer threads call `addTask` concurrently; each node has one or more workers calling `runTask`.
- `rebalanceTask()` runs on a background timer while adds and runs continue.
- The set of nodes is fixed at startup, so the node map is read-only.
- Rebalance target: every node within ±1 of the average. Only pending tasks move.
- Approximate balance is acceptable. Hard guarantees: no task lost or duplicated, no deadlock.

## Interview walkthrough

The design converged after six rounds of probing; the biggest gaps were deadlock avoidance by hand and loop invariants.

| # | Topic | Your answer | Interviewer probe | Outcome |
| --- | --- | --- | --- | --- |
| 1 | Data structures and locking | Map of nodeId → Node {id, queue}; global lock for the map plus a lock per node | Does running a task under the node lock block adds? Does a global lock protect queues that writers modify under node locks only? | Right shape; two flaws to fix |
| 2 | Lock scope and empty queue | Separate locks for add and run; workers wait on the queue until signaled | Add and run mutate the same queue, so one lock is required | Fixed: pop under lock, execute outside it. Map is read-only, so no global lock |
| 3 | Spurious wakeups and rebalance strategy | Wait in a `while` loop; rebalance must also signal; lock the whole system for rebalance | Plain `int` counter races; locking all N nodes needs an order and stalls the cluster | Correct on `while` and signaling; deadlock question still open |
| 4 | First `moveTask` | Used `scoped_lock(dn->mtx, rn->mtx)` and a 50/50 split loop | Direction inverted, `or` condition loops forever, same-ID and missing-key cases, no notify | Strong lock choice; several logic bugs |
| 5 | Second `moveTask` | Fixed direction; switched `or` to `and`; guard for same ID | How do you avoid deadlock without `scoped_lock`? Why is locking the same mutex twice dangerous? | Could not give lock ordering by hand |
| 6 | `rebalanceTask` | Snapshot sizes, sort, two pointers, move toward average | Wrong mutex for size reads, unlocked live reads, termination on exact equality, pop from empty queue | Approach right; execution had races and an infinite-loop risk |

## Key concepts

Six ideas carry this problem; remember these even if you forget the code.

1. **Lock per node, not per system.** Each node's queue has its own mutex, so work on different nodes never contends. A fixed node map is read-only and needs no lock.
2. **The lock protects the queue, not the work.** Pop under the lock, release it, then execute. Holding the lock during execution blocks every `addTask` to that node.
3. **One lock per data structure.** Add and run both mutate the same queue, so they share one mutex. Reading `size()` needs that same mutex, not a different one.
4. **Condition variables wait in a loop.** Use `cv.wait(lk, predicate)` or `while (q.empty()) cv.wait(lk)` to survive spurious wakeups and lost races. Every path that adds tasks (including rebalance) must notify.
5. **Deadlock avoidance with two locks.** Use `std::scoped_lock(a, b)`, or lock by ascending node ID by hand so A→B and B→A take locks in the same order. Never lock the same non-recursive mutex twice: that is undefined behavior and usually self-deadlocks.
6. **Stale plan, safe execution.** Plan from a snapshot taken under brief per-node locks. Inside each move, re-check live sizes under both locks and never move more than the donor holds. Advance loop pointers from the plan, not from live reads, so the loop always terminates.

## Code review of the final submission

The final code had the right structure but nine bugs, two of them able to crash or hang the scheduler.

| Function | Issue | Why it matters | Fix |
| --- | --- | --- | --- |
| `moveTask` | `share` means a target, a count and a receiver size at once | Logic is impossible to reason about | Pass the average; compute the count inside |
| `moveTask` | Loop can pop an empty donor queue | Undefined behavior | Move exactly `k = min(donor − avg, avg − receiver)` from live sizes |
| `moveTask` | `<= share` fills the receiver to share + 1 | Off by one | Bound by `k` |
| `moveTask` | Donor can drop below average | Creates a new imbalance | Same `k` bound |
| `moveTask` | `cv.notify_all()` on an unspecified cv | Receiver's sleeping worker may never wake | `rn->cv.notify_all()` |
| `rebalance` | Sizes read under a shared `mm`, not the node's mutex | Data race with writers | Lock `node->mtx` for each read |
| `rebalance` | `m[nodef].size()` read live with no lock | Data race | Use snapshot values |
| `rebalance` | Pointers advance only when live size equals `share` exactly | Infinite loop with a remainder or concurrent pops | Update planned sizes by `k`; advance when at or past average |
| Both | `dn.pop()`, `Node*` in `vector<int>`, `mod.size()`, duplicate `lock` names | Does not compile | Read code once before saying done |

The earlier `moveTask` versions also had an inverted move direction and an `or` loop condition that oscillated forever; both were fixed during the interview.

## Reference solution (C++)

Per-node mutex and condition variable, a read-only node map, deadlock-free pairwise moves, and a rebalance loop that always terminates.

```cpp
struct Node {
    mutex mtx;
    condition_variable cv;
    deque<int> q;
};
unordered_map<int, Node*> m;   // built once at startup, read-only after

void addTask(int nodeId, int taskId) {
    auto it = m.find(nodeId);
    if (it == m.end()) return;
    Node* n = it->second;
    { lock_guard<mutex> lk(n->mtx); n->q.push_back(taskId); }
    n->cv.notify_one();
}

void runTask(int nodeId) {                 // called by that node's workers
    Node* n = m.at(nodeId);
    int task;
    {
        unique_lock<mutex> lk(n->mtx);
        n->cv.wait(lk, [&]{ return !n->q.empty(); });   // while-loop inside
        task = n->q.front(); n->q.pop_front();
    }
    execute(task);                          // no lock held
}

// Move tasks donor -> receiver using LIVE sizes, never past avg on either side
void moveTask(int did, int rid, int avg) {
    if (did == rid) return;                 // same mutex twice = UB
    auto di = m.find(did), ri = m.find(rid);
    if (di == m.end() || ri == m.end()) return;
    Node* dn = di->second; Node* rn = ri->second;
    int moved = 0;
    {
        scoped_lock lk(dn->mtx, rn->mtx);   // deadlock-free for any order
        int k = min((int)dn->q.size() - avg, avg - (int)rn->q.size());
        for (; k > 0; --k, ++moved) {
            rn->q.push_back(dn->q.back());  // steal newest from tail
            dn->q.pop_back();
        }
    }
    if (moved) rn->cv.notify_all();
}

void rebalanceTask() {
    vector<pair<int,int>> snap;             // (size, nodeId)
    int total = 0;
    for (auto& [id, n] : m) {
        lock_guard<mutex> lk(n->mtx);       // brief, per-node
        snap.push_back({(int)n->q.size(), id});
        total += n->q.size();
    }
    if (snap.empty()) return;
    int avg = total / snap.size();
    sort(snap.begin(), snap.end());

    int i = 0, j = snap.size() - 1;
    while (i < j) {
        auto& [lo, loId] = snap[i];
        auto& [hi, hiId] = snap[j];
        if (lo >= avg) { ++i; continue; }
        if (hi <= avg) { --j; continue; }
        int k = min(hi - avg, avg - lo);
        moveTask(hiId, loId, avg);
        lo += k; hi -= k;                   // update the PLAN, not live state
    }
}

void printTask() {
    for (auto& [id, n] : m) {
        lock_guard<mutex> lk(n->mtx);       // per-node consistent, not global
        cout << id << ": ";
        for (int t : n->q) cout << t << ' ';
        cout << '\n';
    }
}
```

Manual alternative to `scoped_lock`: lock the node with the smaller ID first, then the larger.

```cpp
Node* first  = (did < rid) ? dn : rn;
Node* second = (did < rid) ? rn : dn;
lock_guard<mutex> l1(first->mtx);
lock_guard<mutex> l2(second->mtx);
```

Known refinement: with `total % N` left over, one donor can keep several extra tasks. Letting donors drain to `avg + 1` while extras remain spreads the remainder.

## Feedback

Verdict: lean hire at mid-level, borderline for senior. Your concurrency instincts are solid; execution under pressure is the gap.

**Strengths**

- Picked the right structure early: a fixed node map, each node with its own lock.
- Changed the design when pushed, dropping the global lock once the map was fixed.
- Asked good clarifying questions, especially about tolerating stale sizes during rebalance.
- Used `while` with the condition variable and explained why.
- Chose `std::scoped_lock` for the two-node move, the strongest moment of the interview.

**Areas to improve**

- **Lock ordering by hand.** You spotted the deadlock but not the fix: lock by ascending ID.
- **State invariants before coding.** "Never pop an empty queue" and "every iteration makes progress" would have caught the empty-pop and infinite-loop bugs.
- **Shared state needs its own lock.** Reading `size()` under the wrong mutex, or none, happened twice.
- **Code precision.** Compile-level slips add up in a live round. Reread before saying done.

## Follow-up topics to practice

- [ ] Weighted load: tasks with different CPU costs, balancing by total cost instead of count
- [ ] Work stealing: idle workers steal from other nodes' tails instead of a periodic rebalancer
- [ ] Dynamic membership: nodes joining and leaving at runtime, which brings back a map lock (`shared_mutex`)
- [ ] Drill: write `moveTask` with manual lock ordering from memory, then trace A→B and B→A concurrently
