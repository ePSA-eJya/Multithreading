# Logical Memory Unit — Concurrency Interview Notes

Oct 24, 2026 · @Epsa Sharma

## The problem

Design a thread-safe logical memory unit: an LRU cache of pages in front of slow physical memory, where each page keeps a full history of entries.

**Original ask**

- The CPU requests pages by `pageId`.
- If the page is in logical memory, return it. Otherwise, fetch it from physical memory.
- Keep all entries for each page.
- Support requests for the most recent entry, or the last k entries, of a page.
- Many CPU threads and user threads call this at the same time.

**What the clarifying questions pinned down**

| Question | Answer |
| --- | --- |
| Is k a cap on stored entries? | No. Every entry is kept forever. k is a per-request parameter. |
| What is an entry? | A versioned record (`data`, `version`) appended to a page. Never overwritten. |
| Do users ask for one specific entry? | No. Only latest, or last k (newest first). |
| Is logical memory bounded? | Yes, N pages, LRU eviction. |
| Does a read count as a use? | Yes. Every call moves the page to the front of the LRU list. |
| Read vs write mix? | Read-heavy, about 90% reads. |
| Reader during a write? | Must see a consistent snapshot: all entries before the write or all including it, never a torn list. |
| Contention scope? | Per page. A write to page A must not slow a read of page B. |
| Physical fetch cost? | Slow, about 10 ms, and it can fail. |

**API**

```cpp
Page                 getPage(int pageId);            // CPU path
void                 addEntry(int pageId, string data);
optional<Entry>      getLatest(int pageId);
vector<Entry>        getLastK(int pageId, size_t k); // newest first
```

**Tip:** spend the first few minutes on clarifying questions that change the *threading* design: read/write ratio, capacity, concurrent misses, and consistency.

## Step 1: First instinct

The first design had the right shape: an LRU list of pages, a history per page, and two levels of locking.

**What I proposed**

- A stack of entries for each page.
- A linked list of pages, so an accessed page can be moved to the front.
- On a miss, fetch from physical memory and add the page to the list.
- One global lock protecting the list.
- One lock per page, for threads reading and writing entries.
- For concurrent misses: check a set of pageIds, and fetch if the ID isn't there. (Unsure about this part.)

**What held up**

- Doubly linked list for LRU order.
- Separating the global lock (cache structure) from per-page locks (entries). This is two-level locking.
- Keeping a history of entries per page.

**Gaps the interviewer raised**

1. **Lookup:** a set says a page exists, but not where its node is. How do you get pageId → node in O(1)?
2. **Stack:** only exposes the top. Reading the last k means popping and pushing back, which is ugly under concurrency.
3. **Global lock on reads:** every read moves the page in the LRU list, so every read takes the global lock.
4. **Concurrent miss:** "check the set, then fetch" is a race (see Step 3).
5. **Lock ordering:** with a global lock and page locks, taking them in different orders can deadlock.

## Step 2: Fixing the data structures and locks

Three quick fixes gave a classic LRU cache with a read-write lock per page, while the global lock had to stay a plain mutex.

| Gap | Fix | Why it works |
| --- | --- | --- |
| pageId → node lookup | `unordered_map<pageId, LLNode*>` | HashMap plus doubly linked list gives O(1) lookup, move-to-front, and eviction. |
| Stack of entries | `vector<Entry>`, newest at the back | O(1) amortized append; last k = read the last k indexes backwards. |
| Readers vs writers on a page | Per-page read-write lock (`shared_mutex`) | Many readers together; a writer gets exclusive access. Read-heavy workload benefits. |
| Global LRU structure | Plain `mutex`, not a read-write lock | Every op (even `getLatest`) moves a node to the front, so every op is a write to the list. A read-write lock gives nothing here. |

**Key insight:** a "read" of entries is still a *write* to the LRU list. Look at what each operation mutates, not what it is called.

**Vector resize trap:** when a vector grows it reallocates and copies. A reader iterating during a resize would read freed memory. The per-page read-write lock prevents this, because the writer holds it exclusively during `push_back`.

**Rule:** keep the global critical section tiny. Lock, move the node, unlock. Never do slow work while holding it.

## Step 3: The concurrent-miss race

The core concurrency problem: two threads miss on the same page, and without care both fetch it. The fix is to reserve the slot with a future before fetching.

**The race (check-then-act)**

```text
T1: checks set → 42 not present
T2: checks set → 42 not present
T1: fetches page 42 from physical memory (10 ms)
T2: fetches page 42 from physical memory (10 ms)   ← duplicate work, two copies
```

The check and the act are two separate steps, so another thread can slip in between. Holding the global lock through the whole 10 ms fetch fixes the race but blocks *every* thread, including those reading pages already in memory.

**My idea: reserve the slot first.** On a miss, put the pageId in the map *before* fetching. That's the right half of the answer. Two things were still open:

1. **Atomicity:** "check, then insert" must be one step. Do both under the global lock.
2. **What does T2 do?** It sees 42 in the map, but there's no data yet. It needs something to *wait on*.

**The missing piece: a Future (or Promise)**

A future is a placeholder for a value that will exist later. Store `pageId → Future<Page>` instead of `pageId → Page`.

- The **first** thread to miss creates an empty future, puts it in the map, and fetches.
- **Later** threads find the future and block on `future.get()`. No spinning.
- When the fetch finishes, the first thread completes the future, and all waiters wake with the same page.

| Language | Type |
| --- | --- |
| C++ | `std::promise` + `std::shared_future` (many threads can `.get()` a shared\_future; a plain `future` only once) |
| Java | `CompletableFuture<Page>` |
| JS/TS | `Promise<Page>` |
| Go | a channel, or `sync.Once` per entry |

**Habit to build:** for every step, ask out loud, "What if two threads do this at the same time?"

## Step 4: Putting getPage together

The final shape is a two-phase `getPage`: decide under one short lock, release it, then fetch outside the lock.

**My walkthrough:** the first thread grabs a lock on the set, inserts the pageId with a future, and fetches. Other threads wait on the future. After the fetch, lock the linked list and add the node, then lock the map and add the entry.

**Three problems with that version**

1. **Checked before locking.** The same check-then-act race. The check must happen *inside* the lock.
2. **Too many structures and locks.** The set is redundant, because the map of futures already says "present or in flight." The map and list must change *together*: with separate locks, eviction can remove a node from the list while another thread still finds it in the map, which corrupts the structure. They are one logical structure, so they get **one lock**.
3. **Unanswered questions:** when the lock is released, what happens on failure, where eviction runs, and lock ordering.

**The resolved flow**

1. Lock the global mutex.
2. Look up pageId in the map.
   - **Found:** move the node to the front and copy its future.
   - **Not found:** create a promise and future, insert the node at the front, mark "I am the fetcher", and evict from the back while size > N.
3. Unlock. **Never hold the global lock during the 10 ms fetch.**
4. If I'm the fetcher, fetch from physical memory.
   - **Success:** `set_value(page)` wakes all waiters.
   - **Failure:** under the lock, remove the slot (so the next caller retries), then `set_exception`. Otherwise waiters hang forever.
5. Everyone returns `future.get()`.

**Lock ordering (#5)**

`addEntry` needs the global lock (LRU move) and the page lock (append). The clean answer is to **never hold both at once**: `getPage` takes and releases the global lock, then the caller takes the page lock. No nesting means no ordering problem, so no deadlock. If you must nest, always take them in the same order everywhere.

**Bugs from my first code draft (common C++ traps)**

- `void getPage` should return the page.
- The same name `m` was used for both the mutex and the map.
- `unique_lock mlock` was declared twice in one scope; reuse it with `mlock.lock()`.
- `mp[pid]` silently **inserts** a default value when the key is missing. Use `find()`.
- The map should hold a future, not a `Page`.
- The second lock/unlock block isn't needed: the node is inserted in phase 1, and the future carries the result.

## Step 5: Final code (C++17)

This version passed ThreadSanitizer with no data races. In testing, 8 concurrent misses on one page caused exactly 1 fetch, 400 concurrent writes interleaved safely with `getLastK` reads, and a failed fetch reached waiters as an exception and could be retried.

```
#include <algorithm>
#include <future>
#include <list>
#include <memory>
#include <mutex>
#include <optional>
#include <shared_mutex>
#include <string>
#include <unordered_map>
#include <vector>

struct Entry {
    std::string data;
    uint64_t version;
};

struct Page {
    Page(int id) : id(id) {}
    const int id;
    mutable std::shared_mutex rw;     // per-page RW lock: guards entries
    std::vector<Entry> entries;       // full history, newest at back
    uint64_t nextVersion = 1;
};
using PagePtr = std::shared_ptr<Page>;

PagePtr fetchFromPhysical(int pid);   // slow (~10ms), may throw

class LogicalMemory {
public:
    LogicalMemory(size_t capacity) : capacity_(capacity) {}

    // ---------- CPU path ----------
    PagePtr getPage(int pid) {
        std::shared_future<PagePtr> fut;
        std::promise<PagePtr> prom;
        uint64_t myToken = 0;
        bool iAmFetcher = false;

        {   // Phase 1: short critical section, check + insert atomically
            std::lock_guard<std::mutex> lk(mtx_);
            auto it = index_.find(pid);
            if (it != index_.end()) {
                // Hit (or someone else is fetching): move to front, share their future
                lru_.splice(lru_.begin(), lru_, it->second);
                fut = it->second->fut;
            } else {
                // Miss, and I'm first: reserve the slot with a placeholder future
                fut = prom.get_future().share();
                myToken = ++tokenCounter_;
                lru_.push_front(Slot{pid, fut, myToken});
                index_[pid] = lru_.begin();
                iAmFetcher = true;
                evictIfNeededLocked();
            }
        }   // lock released BEFORE the slow fetch

        // Phase 2: only one thread per pid does the slow I/O
        if (iAmFetcher) {
            try {
                prom.set_value(fetchFromPhysical(pid));
            } catch (...) {
                {   // remove the failed slot so the next caller can retry,
                    // but only if it's still OUR slot
                    std::lock_guard<std::mutex> lk(mtx_);
                    auto it = index_.find(pid);
                    if (it != index_.end() && it->second->token == myToken) {
                        lru_.erase(it->second);
                        index_.erase(it);
                    }
                }
                prom.set_exception(std::current_exception());  // wake waiters with the error
            }
        }

        return fut.get();   // everyone blocks here; no spinning
    }

    // ---------- Entry APIs ----------
    void addEntry(int pid, std::string data) {
        PagePtr p = getPage(pid);                          // global lock taken & released inside
        std::unique_lock<std::shared_mutex> wl(p->rw);     // exclusive: one writer
        p->entries.push_back(Entry{std::move(data), p->nextVersion++});
    }

    std::optional<Entry> getLatest(int pid) {
        PagePtr p = getPage(pid);
        std::shared_lock<std::shared_mutex> rl(p->rw);     // shared: many readers
        if (p->entries.empty()) return std::nullopt;
        return p->entries.back();
    }

    std::vector<Entry> getLastK(int pid, size_t k) {
        PagePtr p = getPage(pid);
        std::shared_lock<std::shared_mutex> rl(p->rw);
        const auto& e = p->entries;
        size_t n = std::min(k, e.size());
        return std::vector<Entry>(e.rbegin(), e.rbegin() + n);  // copy under lock = consistent snapshot
    }

private:
    struct Slot {
        int pid;
        std::shared_future<PagePtr> fut;
        uint64_t token;   // identifies which fetch created this slot
    };

    void evictIfNeededLocked() {               // caller holds mtx_
        while (lru_.size() > capacity_) {
            index_.erase(lru_.back().pid);
            lru_.pop_back();
        }
    }

    const size_t capacity_;
    std::mutex mtx_;                                            // guards lru_ + index_ together
    std::list<Slot> lru_;                                       // front = MRU, back = LRU
    std::unordered_map<int, std::list<Slot>::iterator> index_;  // pid -> list node, O(1)
    uint64_t tokenCounter_ = 0;
};
```

**How the code answers each question**

| Question | Answer in the code |
| --- | --- |
| Atomic check-and-insert | `find` and insert both happen under `mtx_`; exactly one thread becomes the fetcher. |
| How waiters wait | They copy the `shared_future` and block in `fut.get()`, woken by `set_value`. |
| Lock during fetch | Never held. `mtx_` is released before `fetchFromPhysical`. |
| Fetch failure | Slot removed, then `set_exception`. Waiters get the error; the next call retries. |
| Map + list consistency | One mutex guards both. `std::list::splice` moves to front in O(1). |
| Torn reads / vector resize | Readers `shared_lock`, writer `unique_lock`; `getLastK` copies under the lock. |
| Lock ordering | No function holds both locks, so no deadlock. |
| Page evicted while in use | Callers hold a `shared_ptr<Page>`, so memory stays valid. |
| Why the token | Stops a failing fetcher from deleting a newer slot inserted after eviction. |

## Step 6: My version with condition variables

The same design works with a mutex and condition variable per page instead of a future. This is the version I'm most comfortable writing in an interview. It passed ThreadSanitizer: 8 concurrent misses caused 1 fetch, and a failed fetch did not hang waiters.

**How it maps to the future version**

- `Page* page == nullptr` means "still loading", which plays the role of the incomplete future.
- Waiters sleep on `node->cv` instead of `future.get()`.
- The fetcher sets `page` (or `failed`), then calls `notify_all`, which plays the role of `set_value` / `set_exception`.

**Bugs I made along the way (and the fixes)**

| Bug | What goes wrong | Fix |
| --- | --- | --- |
| Separate `map<int,mutex>` and `map<int,condition_variable>` accessed outside the global lock | Data race: concurrent inserts into a `std::map` are undefined behavior | Put `mutex` + `cv` inside `Node`; touch `mp` only under `m` |
| Re-checking `mp[pid] == nullptr` after unlocking to decide who fetches | Racy, unprotected read | Decide once under the lock: `bool fetch` |
| `catch` only logs the error | `page` stays null; nobody notifies; every waiter (and the fetcher) waits forever; future calls hang too | `failed` flag, always `notify_all`, remove our node from `mp`, throw after waking |
| `Page* p;` uninitialized | On exception, `p` is garbage and gets stored | `Page* p = nullptr;` |
| `getLastK` pushes `e[n]` | Out of bounds when k ≥ size; wrong entries otherwise | Walk `i` from `size-1` down: `e[i]` |
| `auto e = node->page->entries;` | Copies the whole vector on every read | `auto& e` (safe, the page lock is held) |
| `min(k, e.size())` | `int` vs `size_t` compile error | `min(k, (int)e.size())` |
| Calls to `add()` / `removeNode()` vs definitions `addFront()` / `remove()` | Compile error | One consistent set of names |
| No `addEntry` | Entries can never be written | Add it, with the page lock |

**Final code**

```cpp
struct Page{
    int pid;
    vector<string>entries;
};

struct Node{
    int pid;
    Page* page=nullptr;              // nullptr = still loading
    bool failed=false;
    Node* next = nullptr;
    Node* prev=nullptr;
    mutex m;                         // per-page lock
    condition_variable cv;           // waiters sleep here
    Node(int id){
        pid=id;
    }
};

class LRUCache{
    mutex m;                         // guards mp + linked list
    int cap;
    Node* head;
    Node* tail;
    map<int,Node*>mp;

    void removeNode(Node* node){
        node->prev->next=node->next;
        node->next->prev=node->prev;
    }

    void addFront(Node* n) {
        n->next = head->next;
        n->prev = head;
        head->next->prev = n;
        head->next = n;
    }

    Node* getNode(int pid){
        Node* node;
        bool fetch = false;

        // Step 1: global lock, only for map + list
        unique_lock<mutex> lock(m);
        if(mp.count(pid)==0){
            node=new Node(pid);
            mp[pid] = node;
            addFront(node);
            fetch=true;                              // decided under lock

            if ((int)mp.size() > cap) {
                Node* lru = tail->prev;
                removeNode(lru);
                mp.erase(lru->pid);                  // not deleted: may still be in use
            }
        }
        else{
            node=mp[pid];
            removeNode(node);
            addFront(node);
        }
        lock.unlock();                               // never hold during fetch

        // Step 2: only the first thread fetches
        if(fetch){
            Page* p = nullptr;
            try{
                p=fetchPage(pid);
            }
            catch(const exception &e){
                cerr<<"error in fetch page "<<pid<<": "<<e.what()<<endl;
            }

            if (!p) {                                // failed: remove OUR node so next call retries
                lock.lock();
                if (mp.count(pid) && mp[pid] == node) {
                    removeNode(node);
                    mp.erase(pid);
                }
                lock.unlock();
            }

            unique_lock<mutex> plock(node->m);
            node->page=p;
            node->failed=(p==nullptr);
            plock.unlock();

            node->cv.notify_all();                   // always notify
        }

        // Step 3: wait until loaded OR failed
        unique_lock<mutex> plock(node->m);
        while(node->page==nullptr && !node->failed){  // loop handles spurious wakeups
            node->cv.wait(plock);
        }
        if(node->failed) throw runtime_error("page fetch failed");

        return node;
    }

    public:

    LRUCache(int c){
        cap=c;
        head=new Node(-1);
        tail=new Node(-1);
        head->next=tail;
        tail->prev=head;
    }

    Page* getPage(int pid){
        return getNode(pid)->page;
    }

    void addEntry(int pid, const string& data){
        Node* node=getNode(pid);
        unique_lock<mutex> plock(node->m);
        node->page->entries.push_back(data);
    }

    string getLatest(int pid){
        Node* node=getNode(pid);
        unique_lock<mutex> plock(node->m);
        auto& e=node->page->entries;
        if(e.empty()) return "";
        return e.back();
    }

    vector<string> getLastK(int pid,int k){
        Node* node=getNode(pid);
        unique_lock<mutex> plock(node->m);
        auto& e=node->page->entries;

        vector<string>v;
        int n=min(k,(int)e.size());
        int i=(int)e.size()-1;
        while(n>0){
            v.push_back(e[i]);
            i--;
            n--;
        }
        return v;
    }
};
```

**Say out loud in the interview:** evicted nodes aren't deleted because another thread may still hold the pointer, and `shared_ptr<Node>` would fix that. Also, entries use a plain mutex for simplicity; a `shared_mutex` would let readers run in parallel.

|  | Future version (Step 5) | Condition variable version |
| --- | --- | --- |
| Code size | Shorter: the future handles waiting and errors | More code: state, mutex and notify are manual |
| Error propagation | Built in (`set_exception`) | Manual (`failed` flag) |
| What it shows | You know the library | You understand the mechanism underneath |

## Interview bar, follow-ups and cheat sheet

For a fresher, the data structures, the two-level locking, and spotting the concurrent-miss race are the bar. Futures and the failure path are a bonus.

| Level | What's expected | Where I was |
| --- | --- | --- |
| Fresher | HashMap + doubly linked list, vector of entries, one cache lock + per-page RW lock, spot the double-fetch race, mostly correct `getPage` | Hit most of this, with hints on the race |
| 2–4 years | Futures to block waiters, release the lock before the fetch, `set_exception` on failure | Didn't know the term "future" |
| Senior | Token check for evict-then-replace, `shared_ptr` lifetime through eviction, the follow-ups below | Beyond scope |

**Fresher-safe fallback:** hold one mutex for the whole `getPage`, *including* the fetch, then say: "This blocks everyone during the fetch. I'd improve it by releasing the lock and using a placeholder." Naming the flaw scores almost as well as fixing it.

**Follow-up questions to practise**

1. **Lost writes on eviction:** an evicted page loses its new entries. How would you write back to physical memory without holding `mtx_` during I/O?
2. **Global mutex bottleneck:** at very high read QPS, `mtx_` is contended. Options: shard the cache by pageId hash, or use approximate LRU and buffer access records (as Caffeine does).
3. **Unbounded history:** a hot page's vector grows forever. What changes if memory per page is capped?
4. **Lock-free `getLatest`:** could an atomic pointer to the latest entry avoid the page lock?

**Cheat sheet**

- LRU = `unordered_map<key, list::iterator>` + `std::list`; `splice` to move to front.
- If a "read" mutates shared state (LRU order), it's a write. Use a mutex, not a read-write lock.
- Check-then-act is a race. Do the check and the act under the same lock.
- Never do slow I/O while holding a shared lock.
- Placeholder future in the map: one fetcher, many waiters, no spinning.
- On failure, remove the placeholder and `set_exception`, or waiters hang forever.
- Avoid deadlock by never holding two locks at once, or by always taking them in one fixed order.
- `map[key]` in C++ inserts on miss. Use `find()`.
- Copy results out under the lock for a consistent snapshot.
