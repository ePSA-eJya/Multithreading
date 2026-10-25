# Multithreaded Web Crawler — Interview Revision Notes

Oct 24, 2026 · @Epsa Sharma

## The problem

Crawl every page reachable from `startUrl` on the same hostname, each exactly once, using multiple threads.

```java
interface HtmlParser {
    // Blocking HTTP call. Takes ~15ms on average and can be slow or fail.
    List<String> getUrls(String url);
}

List<String> crawl(String startUrl, HtmlParser htmlParser);
```

**Requirements**

1. Start from `startUrl` and call `getUrls(url)` to get the links on each page.
2. Only crawl URLs with the **same hostname** as `startUrl`.
3. Crawl each URL **at most once**.
4. Return every crawled URL, in any order.
5. The solution **must be multithreaded**; a single-threaded one times out.

**Assumptions:** every URL starts with `http://` and has no port. A trailing `/` makes a different URL. There are at most about 1,000 URLs.

**Example**

```
startUrl = "http://news.yahoo.com/news/topics/"

  news.yahoo.com/news/topics/  -> [news.yahoo.com/news, news.google.com]
  news.yahoo.com/news          -> [news.yahoo.com/us, news.yahoo.com/news/topics/]
  news.yahoo.com/us            -> []

Output: [ "http://news.yahoo.com/news/topics/",
          "http://news.yahoo.com/news",
          "http://news.yahoo.com/us" ]
```

`news.google.com` is excluded because its hostname is different.

**Hostname rule:** the hostname is everything after `http://` up to the next `/`, or to the end of the string. Don't assume it ends in `.com`. `news.yahoo.com` and `yahoo.com` are different hosts, so compare exactly, never by suffix.

## Step 1: High-level design

Treat the crawl as a BFS over a graph of URLs, with a pool of worker threads pulling from one shared queue.

**Shared state**

| State | Purpose | Type in C++ |
| --- | --- | --- |
| Queue | URLs waiting to be crawled | `queue<string> q` |
| Visited set | URLs already seen, so none is crawled twice | `set<string> s` |
| Working counter | Threads currently processing a URL | `int work` |
| Stop flag | Tells workers the crawl is finished | `bool stop` |
| Mutex + condition variable | Protect the state above; let idle workers sleep | `mutex m`, `condition_variable cv` |

**Basic flow**

1. Push `startUrl` into the queue and mark it visited.
2. A worker pops a URL and calls `getUrls(url)`.
3. For each returned link on the same host that isn't visited yet, mark it visited and push it.
4. Repeat until no work is left.

**Key point: mark a URL visited when you enqueue it, not when you crawl it.** Otherwise two workers can both enqueue the same URL before either crawls it.

## Step 2: The visited-set race

The check and the insert must be one atomic step, under one lock.

**The bug:** Thread A and Thread B each get a page linking to `/us`. Both check `visited.contains("/us")`, both see `false`, and both enqueue it. `/us` is crawled twice.

**The fix:** hold the lock across both the check and the insert.

```cpp
lock_guard<mutex> lock(m);
if (s.find(it) == s.end()) {   // check
    s.insert(it);               // insert, under the same lock
    q.push(it);
}
```

Shorter forms of the same idea:

- C++: `if (s.insert(it).second) q.push(it);` because `insert` returns whether it was new. You still need the lock, since `std::set` isn't thread-safe.
- Java: `ConcurrentHashMap.newKeySet().add(url)` returns `false` if the URL is already present, with no explicit lock.

## Step 3: Termination detection

The crawl is done only when **the queue is empty AND no thread is working**. An empty queue alone is not enough.

**Why an empty queue isn't enough**

```
t0: Queue = [startUrl]
t1: Thread A pops startUrl. Queue is now EMPTY.
t2: Thread A calls getUrls(startUrl), which blocks for ~15ms.
t3: Thread B sees an empty queue. Is the crawl done?
    No: Thread A is about to push new URLs.
```

**The fix: a `work` counter of threads currently processing a URL.** A thread stays "working" until its children are in the queue. Because it enqueues first and decrements second, `q.empty() && work == 0` can't happen while work remains.

**Lifecycle of one URL**

```
lock:   wait while (queue empty && !stop)
        if stop → exit
        pop url; work++               ← same critical section as the pop
unlock
        getUrls(url)                  ← slow call, NO lock held
lock:   for each same-host link: if not visited → mark visited, push
        work--                        ← only AFTER children are queued
        if queue.empty() && work == 0 → stop = true
unlock
        notify_all                    ← wake workers for new URLs, or to exit
```

**Rules that make it correct**

1. **Increment when you pop, in the same critical section.** If you pop, unlock, then increment, another thread can see `queue empty && work == 0` in that gap and exit early.
2. **Decrement only after the children are pushed.**
3. **The last thread out turns off the lights.** The thread that makes `work == 0` with an empty queue sets `stop = true` and calls `notify_all`. With `notify_one`, the other waiting threads never wake up and `join()` hangs.
4. **Wait in a `while` loop, not an `if`.** Condition variables can wake up spuriously, and another thread may have taken the item first. Re-check the condition every time you wake.
5. **Never spin or sleep for a fixed time.** Idle workers wait on the condition variable until notified.

## Step 4: Error handling

Fail the URL, not the thread: catch the exception, log it, treat the page as having no links, and keep going.

**Two mistakes to avoid**

- **Killing the worker thread.** If a thread returns on error, the pool shrinks by one per bad URL. With 4 threads and 4 broken links, nobody is left to process the queue.
- **Skipping the cleanup.** `work--`, the termination check and `notify_all` must run whether `getUrls` succeeds or fails. If the last active thread fails and skips them, `work` never reaches 0 and every thread waits forever.

```cpp
vector<string> nei;
try {
    nei = geturls(url);
}
catch (const exception& e) {        // standard exceptions: log e.what()
    cerr << "Failed to fetch " << url << ": " << e.what() << "\n";
}
catch (...) {                       // anything else
    cerr << "Failed to fetch " << url << ": unknown error\n";
}
// nei is empty on failure; the code below still runs work-- and notify_all
```

**C++ catch rules**

- `catch(e)` isn't valid; a catch parameter needs a type.
- Put the specific catch first and `catch(...)` last.
- `catch(...)` also catches things that don't inherit from `std::exception`, such as a thrown `int`.

**Possible follow-up: retries.** If you retry a failed URL, cap the attempts (for example, 3) and use backoff, so a permanently dead link can't loop forever.

## Step 5: Final code (self-feeding work queue)

This is the version written in the interview, with all fixes applied. Workers pull URLs from a queue and push the links they find back into it.

```cpp
#include <bits/stdc++.h>
using namespace std;

class Threadpool{
    private:
    int nt,work=0;                 // work = threads currently processing a URL

    queue<string>q;
    bool stop=false;
    vector<thread>threads;
    mutex m;
    condition_variable cv;

    public:
    set<string>s;                  // visited (public so main can read it after crawl())

    Threadpool(int n,string url){
        nt=n;
        q.push(url);
        s.insert(url);

        for(int i=0;i<nt;i++){
            threads.emplace_back([this]{
                while(task());     // keep working until task() says the crawl is done
            });
        }
    }

    ~Threadpool(){
        unique_lock<mutex> lock(m);
        stop=true;
        lock.unlock();
        cv.notify_all();

        for(auto& th: threads)
        {
            if(th.joinable()) th.join();
        }
    }

    string getHostname(const string &url){
        stringstream ss(url);
        string part;
        string host="";
        int cnt=0;
        while(getline(ss,part,'/')){
            host+=part+'/';
            cnt++;
            if(cnt==3) break;
        }

        return host;               // "http://news.yahoo.com/"
    }

    void crawl(){                  // wait until every thread has finished
        for(auto& th: threads){
            if(th.joinable()) th.join();
        }
    }

    bool task(){                   // returns false when the crawl is finished
        unique_lock<mutex> lock(m);

        while(q.empty() && !stop){
            cv.wait(lock);
        }

        if(stop){
            return false;
        }
        work++;                    // same critical section as the pop
        string url=q.front();
        q.pop();
        lock.unlock();

        vector<string>nei;

        try{
            nei=geturls(url);      // slow I/O, no lock held
        }
        catch(const exception& e){
            cerr << "Failed to fetch " << url << ": " << e.what() << "\n";
        }
        catch(...){
            cerr << "Failed to fetch " << url << ": unknown error\n";
        }

        lock.lock();
        for(auto it:nei){
            if(s.find(it)!=s.end()) continue;
            if(getHostname(url)==getHostname(it)){
                s.insert(it);
                q.push(it);
            }
        }
        work--;                    // only after children are queued
        if(q.empty() && work==0){
            stop=true;
        }

        lock.unlock();
        cv.notify_all();

        return true;
    }
};

int main(){

    Threadpool t(max(4u, thread::hardware_concurrency()), starturl);

    t.crawl();
    for(auto& u : t.s) cout << u << "\n";
}
```

`geturls` and `starturl` come from the problem harness.

**Bugs fixed along the way**

| Bug | Effect | Fix |
| --- | --- | --- |
| Workers waited on a `tasks` queue that nothing filled | Every worker slept forever | Wait on the URL queue `q` |
| Worker handled one item, then exited | Crawl stopped after a few URLs | `while(task());` |
| No `q.empty() && work == 0` check | Crawl never ended on its own | Set `stop` in `task()` |
| Unlocked between the wait and the pop | `q.front()` on an empty queue (undefined behavior) | Wait, pop and `work++` under one lock |
| `notify_one` | Threads left waiting; `join()` hangs | `notify_all` |
| `set<string> crawl()` returned nothing | Undefined behavior | `void crawl()` |
| `th.joinable` | Compile error | `th.joinable()` |
| `catch(e)` | Compile error | `catch(const exception& e)` + `catch(...)` |
| `hardware_concurrency()` may return 0 | No threads created | `max(4u, ...)` |

## Why `crawl()` when the destructor already joins?

`crawl()` joins at the right moment: after the work is finished but before you read the result. The destructor can only join at the closing `}` of `main`, which is too late.

**Without `crawl()`**

```cpp
int main(){
    Threadpool t(8, starturl);   // ① threads start crawling in the background
    for(auto& u : t.s)           // ② runs IMMEDIATELY: s holds only startUrl,
        cout << u << "\n";       //    and threads may be writing to it (data race)
}                                // ③ destructor joins HERE, after you've printed
```

**With `crawl()`**

```cpp
int main(){
    Threadpool t(8, starturl);   // ① threads start crawling
    t.crawl();                   // ② main WAITS until every thread finishes
    for(auto& u : t.s)           // ③ s is complete and safe to read
        cout << u << "\n";
}                                // ④ destructor: nothing left to join
```

**Also note:** this destructor sets `stop = true` before joining, so it would cut a running crawl short.

**Why reading `t.s` without the lock is safe after `crawl()`:** `join()` guarantees everything a thread did happens before `join()` returns. Once every thread is joined, nothing else touches `s`.

**Public `s` vs. returning it:** a public `s` lets callers read it while the crawl is still running, which is a data race. Returning it from `crawl()` or exposing a getter prevents that. Both work; encapsulation is the cleaner design.

**Analogy:** `crawl()` is waiting at the door for the food delivery before eating. Relying on the destructor is sitting down to an empty plate right after ordering.

## Step 6: Generic thread pool version

The same crawler can run on a reusable pool: each URL becomes a task, and a task submits more tasks for the links it finds. The pool knows nothing about crawling.

**Self-feeding queue vs. generic pool**

|  | Self-feeding queue (Step 5) | Generic thread pool |
| --- | --- | --- |
| Where work comes from | Workers create it | Outside code calls `submit(task)` |
| What's in the queue | Data (`string` URLs) | Functions (`function<void()>`) |
| When it stops | On its own, at `q.empty() && work == 0` | When the owner calls `wait()` or destroys it |
| Reusable for other problems | No | Yes |

```cpp
// ───── Generic thread pool: knows nothing about crawling ─────
class Threadpool{
    private:
    int nt,work=0;                        // work = tasks submitted but not yet finished
    queue<function<void()>>tasks;
    bool stop=false;
    vector<thread>threads;
    mutex m;
    condition_variable cv;                // workers wait here for tasks
    condition_variable doneCv;            // wait() waits here for work == 0

    public:
    Threadpool(int n){
        nt=n;
        for(int i=0;i<nt;i++){
            threads.emplace_back([this]{
                while(true){
                    function<void()>task;

                    unique_lock<mutex> lock(m);
                    while(tasks.empty() && !stop){
                        cv.wait(lock);
                    }
                    if(stop && tasks.empty()){
                        return;
                    }
                    task=move(tasks.front());
                    tasks.pop();
                    lock.unlock();

                    try{
                        task();                       // run outside the lock
                    }
                    catch(...){
                        //log
                    }

                    lock.lock();
                    work--;                           // finished only AFTER task() returns
                    if(work==0) doneCv.notify_all();
                }
            });
        }
    }

    void submit(function<void()> f){
        unique_lock<mutex> lock(m);
        tasks.push(move(f));
        work++;                                   // counted the moment it's submitted
        lock.unlock();
        cv.notify_one();
    }

    void wait(){                                  // block until every task is finished
        unique_lock<mutex> lock(m);
        while(work!=0){
            doneCv.wait(lock);
        }
    }

    ~Threadpool(){
        unique_lock<mutex> lock(m);
        stop=true;
        lock.unlock();
        cv.notify_all();

        for(auto& th: threads){
            if(th.joinable()) th.join();
        }
    }
};

// ───── Crawler: uses the pool ─────
class Crawler{
    private:
    mutex m;                                  // protects s
    string host;

    public:
    set<string>s;

    private:
    Threadpool pool;                          // declared LAST, so it's destroyed FIRST

    public:
    Crawler(int n): pool(n){}

    string getHostname(const string& url){
        stringstream ss(url);
        string part;
        string host="";
        int cnt=0;
        while(getline(ss,part,'/')){
            host+=part+'/';
            cnt++;
            if(cnt==3) break;
        }
        return host;
    }

    void task(string url){
        vector<string>nei;

        try{
            nei=geturls(url);                     // slow I/O, no lock held
        }
        catch(...){
            //log
        }

        for(auto& it:nei){
            if(getHostname(it)!=host) continue;

            unique_lock<mutex> lock(m);
            if(s.find(it)!=s.end()) continue;     // already visited
            s.insert(it);
            lock.unlock();

            pool.submit([this,it]{ task(it); });  // child submitted BEFORE parent finishes
        }
    }

    void crawl(string url){
        host=getHostname(url);
        s.insert(url);
        pool.submit([this,url]{ task(url); });
        pool.wait();                              // returns once work == 0
    }
};

int main(){
    Crawler c(max(4u, thread::hardware_concurrency()));
    c.crawl(starturl);
    for(auto& u : c.s) cout << u << "\n";
}
```

**Why it terminates correctly:** `submit()` increments `work` immediately, a parent submits its children inside `task()`, and the worker decrements `work` only after `task()` returns. So `work == 0` only when nothing is queued or running. This is the same rule as Step 3: count the child before the parent finishes.

**Why a second condition variable (`doneCv`)?** If `wait()` shared `cv` with the workers, `submit()`'s `notify_one` could wake the main thread instead of a worker. Main would go back to sleep and the task would sit in the queue with no worker woken. That's a lost wakeup. The alternative is `notify_all` everywhere, which works but wakes more threads than needed.

**Why is `pool` declared last?** C++ destroys members in reverse declaration order. Declaring `pool` last means its destructor joins the threads while `s` and `m` still exist.

**Which to use in an interview:** the Step 5 version is quicker to write. The generic version shows better design and is easier to extend, for example adding rate limiting in `Crawler::task()` without touching the pool.

## Choosing the thread count

The crawler is I/O-bound, so use many more threads than cores. Threads = cores is the rule for CPU-bound work.

| Workload | Threads busy doing | Good thread count |
| --- | --- | --- |
| CPU-bound (image resize, parsing) | Computing | About the number of cores |
| I/O-bound (network, disk) | Mostly waiting | Many times the number of cores |

**Rule of thumb**

```latex
\text{threads} \approx \text{cores} \times \left(1 + \frac{\text{wait time}}{\text{compute time}}\right)
```

Example: 8 cores, 15 ms waiting on the network and 0.5 ms parsing gives 8 × (1 + 30) ≈ 250 threads. With only 8 threads, throughput is about 8 ÷ 15 ms ≈ 530 pages per second while the CPU sits almost idle.

**Why not 1,000 threads?**

- **Memory:** each thread has its own stack, often 1–8 MB.
- **Context switching:** the scheduler spends more time switching than working.
- **Lock contention:** every thread competes for the single mutex `m`.
- **The target server:** 1,000 parallel requests to one host looks like a DoS attack, and you'll be rate-limited or blocked.
- **OS limits:** open sockets and file descriptors.
- **Problem size:** at most \~1,000 URLs exist, so most threads would sit idle.

At large scale, the better answer is async I/O: a few threads managing thousands of open connections.

## When to use a thread pool

Use a thread pool when you have many short, independent tasks: it creates N threads once and reuses them, instead of one expensive thread per task.

```cpp
for(auto& img : images) thread(process, img).detach();          // ❌ 10,000 images = 10,000 threads

Threadpool pool(8);
for(auto& img : images) pool.submit([img]{ process(img); });    // ✅ 8 threads, 10,000 tasks
pool.wait();
```

**It fits when all three hold:** there are many tasks, they're mostly independent, and each is short (milliseconds to seconds).

| Situation | Example | Pool size | Needs `wait()`? |
| --- | --- | --- | --- |
| Batch processing (CPU-bound) | Resize images, parse files | ≈ cores | Yes |
| Many I/O calls | Fetch 500 user profiles, download files | Many × cores | Yes |
| Server request handling | One task per incoming request | Tuned to load | No, runs forever |
| Tasks that create tasks | Crawler, parallel BFS, directory walk | Depends on work | Yes, with a `work` counter |
| Background jobs | Send email, write logs | Small | No, fire and forget |

**When it's the wrong tool**

- **A few long-running jobs** (a keyboard reader, a game loop): use plain `std::thread`s.
- **Tasks that wait on each other:** if every thread runs a task waiting for another queued task, nothing can run it. That's a deadlock.
- **Thousands of mostly idle connections** (chat servers): async I/O (`epoll`, coroutines) scales better.
- **Tiny tasks** (adding two numbers): locking the queue costs more than the work. Batch them into chunks.

**Interview signal:** "process these N things" or "handle each request", with no dependencies between items, usually means a thread pool.

**Thread pool checklist:** a `while` loop (or predicate) around `cv.wait`, run the task outside the lock, `notify_all` on shutdown, join every thread.

## Follow-up: rate limiting (in progress)

To limit requests to `news.yahoo.com` to 10 per second, wrap `geturls` in a rate limiter. Every network call then goes through one point, and the pool and crawl logic don't change.

```cpp
limiter.acquire();          // blocks until a request is allowed
nei = geturls(url);
```

**Algorithm options**

| Algorithm | How it works | Trade-off |
| --- | --- | --- |
| Fixed spacing | One request every 100 ms | Simple and smooth; no bursts |
| Token bucket | Up to 10 tokens, refilled at 10 per second; each request takes one | Allows short bursts, same average rate |

**Open questions to answer next**

- [ ] Which algorithm fits here, and why?
- [ ] How do you protect the shared state (`lastRequestTime` or `tokens`) when many workers call `acquire()` at once?
- [ ] What happens if a thread sleeps while holding the limiter's lock? (Hint: every other thread blocks behind it.)
- [ ] With 250 threads and a 10-per-second limit, most threads just wait in `acquire()`. What does that mean for the thread count?
- [ ] Write a \~20-line `RateLimiter` class in the same style as `Threadpool`: a mutex, the shared state, and `acquire()`.

## Quick revision checklist

Run through this list before submitting any multithreaded solution.

| # | Check | What goes wrong if you miss it |
| --- | --- | --- |
| 1 | Mark visited on enqueue, with check + insert under one lock | Same URL crawled twice |
| 2 | Pop and `work++` in the same critical section | Early exit while work remains |
| 3 | `work--` only after children are queued | Early exit while work remains |
| 4 | Done = `q.empty() && work == 0`, not just `q.empty()` | Crawl stops too soon |
| 5 | Slow I/O (`getUrls`) runs outside the lock | Threads run one at a time |
| 6 | `while` (or predicate) around `cv.wait` | Spurious wakeups; popping an empty queue |
| 7 | `notify_all` on completion or shutdown | Threads stuck waiting; `join()` hangs |
| 8 | Separate condition variables for separate conditions | Lost wakeup |
| 9 | Catch exceptions; fail the URL, not the thread | Pool shrinks; counter never reaches 0 |
| 10 | Cleanup (`work--`, notify) runs on success and failure | Hang when the last thread fails |
| 11 | Wait for completion before reading results | Incomplete results; data race |
| 12 | Join every thread; check `joinable()` | Crash on destruction or double join |
| 13 | Guard against `hardware_concurrency()` returning 0 | No threads created |
| 14 | I/O-bound means more threads than cores | Idle CPU, slow crawl |

**One-line summary:** a thread is "working" from the moment it pops a URL until its children are in the queue. The crawl ends when the queue is empty and nobody is working, and the last thread out wakes everyone to exit.
