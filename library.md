# Library Management System — Interview Design Notes

## 1. Problem statement

Design a small library-management service using the **local filesystem as the only persistent storage**. In-memory data structures may be used for coordination or caching, but they must not be the only place where durable data lives.

### APIs

- `GetBook(bookId)` — returns a book's metadata and content.
- `CreateBook(metadata, content)` — creates a book and returns its integer ID.
- `ListBooks()` — returns IDs of books that exist.
- `UpdateMetadata(bookId, metadata)` — optionally replaces a book's metadata and returns success/failure.

Metadata is a small JSON document. Book content may be stored separately.

## 2. Storage layout

Chosen approach: **store metadata and content in separate files per book**.

```text
library/
├── next_id.txt
└── books/
    ├── 1/
    │   ├── metadata.json
    │   └── content.txt
    ├── 2/
    │   ├── metadata.json
    │   └── content.txt
    └── ...
```

- `metadata.json` stores the metadata JSON.
- `content.txt` stores the book content.
- `next_id.txt` stores the next candidate ID.
- A temporary directory/file may be used while creating or updating data.

## 3. Design decisions and reasoning

### 3.1 ID generation

**Decision:** persist a `next_id.txt` counter.

- Read the next ID, allocate it, and update the counter.
- Coordinate allocation and publication with a mutex so concurrent threads do not allocate the same ID.
- Never reuse IDs that may have belonged to previously created books.
- Gaps in IDs are acceptable.
- On startup, reconcile the counter against valid book directories. Ensure the next ID is greater than the largest existing ID.

A process-local mutex protects threads in one process only. If multiple server processes can write to the same directory, use a cross-process file lock or another cross-process coordination mechanism.

### 3.2 Crash-safe creation

**Decision:** write to a temporary location before publishing a new book.

Safer sequence:

1. Allocate an ID under the creation lock.
2. Create a temporary book directory on the same filesystem.
3. Write `metadata.json` and `content.txt` into that directory.
4. Check every write and close operation for errors.
5. Rename the temporary directory to its final location, such as `books/5/`.
6. Release the lock and return the ID.

Renaming a directory on the same filesystem is generally an atomic publication step: readers should see either no final directory or the published directory, rather than a half-published book. Platform and filesystem semantics must still be checked.

Clean up abandoned temporary directories during startup. Do not treat temporary directories as valid books.

**Durability caveat:** atomic rename does not by itself guarantee that data survives sudden power loss. Strong durability may require flushing file contents and relevant directory metadata to disk using OS-specific synchronization operations such as `fsync`.

### 3.3 Listing books

An index file was considered, but it can become inconsistent if the process publishes a book and crashes before updating the index.

**Simple initial design:** scan the `books/` directory and return IDs of valid final book directories.

- Ignore temporary directories and malformed directory names.
- Validate that a book directory has the required files.
- Decide explicitly how to handle corrupted books (for example, skip and log them, or return a reported error).
- For a very large library, a persistent index can be added later, with startup reconciliation/rebuild logic.

### 3.4 Getting a book

**Decision:** read both files from disk on each `GetBook` request.

- Disk remains the source of truth.
- This avoids cache invalidation complexity in the initial implementation.
- Return an appropriate error for a missing book, missing file, or unreadable/corrupt metadata.
- Add caching only if measured performance needs justify it.

### 3.5 Updating metadata

**Decision:** write the replacement metadata to a temporary file and then rename/replace the original file.

Suggested sequence:

1. Acquire the book's exclusive lock.
2. Verify the book exists.
3. If performing a field-level update, read and parse the existing JSON, modify it, and serialize valid JSON.
4. Write the complete new JSON to a temporary file in the same directory/filesystem.
5. Check write/close errors.
6. Rename the temporary file over `metadata.json`, checking for errors.
7. Release the lock.

This is a **replacement**, not an append. Appending another JSON object to the file would usually make it invalid JSON. Rename-over-existing behavior differs by operating system, so implementation details must be checked.

The content file is not changed by a metadata-only update.

### 3.6 RAM versus disk

- **RAM** is working memory. Global variables, maps, mutexes, caches, threads, and open file handles are lost when the process exits or crashes.
- **Disk/SSD/HDD storage** holds files that can survive a process restart if the data was successfully persisted.
- **ROM** is a different concept, commonly used for firmware; it is not the same thing as a normal writable disk.

For this design, files are the persistent state. In-memory locks and temporary structures coordinate work while the service is running, but cannot replace on-disk recovery information.

### 3.7 Concurrent reads and updates

**Decision:** use reader-writer locking.

- Multiple `GetBook` calls for the same book may run concurrently.
- `UpdateMetadata` for that book takes an exclusive lock.
- Hold the lock for the entire logical operation, not just while opening a file.

In C++, the relevant tools are:

- `std::shared_mutex`
- `std::shared_lock<std::shared_mutex>` for readers
- `std::unique_lock<std::shared_mutex>` for writers

### 3.8 Lock granularity

**Chosen option:** separate locks per book.

This allows a read of book 900 to proceed while book 5 is being updated, instead of blocking unrelated operations behind one global lock.

A possible lock manager is:

```cpp
std::mutex lockMapMutex;
std::unordered_map<int, std::shared_ptr<std::shared_mutex>> bookLocks;
```

The lock manager itself must be protected while locks are looked up or created. Keep a lock alive while any operation may be using it; removing locks carelessly can create races. Creation and listing also involve directory-wide or global state and need their own coordination rules.

A single global reader-writer lock is simpler and may be a reasonable first implementation. Per-book locks improve concurrency but add complexity.

**Why not rely only on atomic rename?** Rename can prevent readers from observing a partially replaced individual file, but it does not coordinate ID allocation, creation publication, listing, or a multi-file logical operation. Locks are still needed to define consistent behavior.

### 3.9 Retrying `CreateBook`

**Chosen option:** allow duplicates for the first version.

If creation succeeds but the response is lost, a client retry can create a second book with a different ID. This is acceptable if duplicate submissions are allowed by the requirements.

If retry safety becomes necessary, add an **idempotency key** supplied by the client and persist the mapping from that key to the created book ID. A retry with the same key can then return the original ID. The mapping must be made crash-safe along with the creation protocol.

### 3.10 Corrupted metadata — decision still open

Scenario: `metadata.json` contains invalid JSON, but `content.txt` is intact.

Options considered:

- **Return an error:** safest default when the API promises metadata plus content.
- **Return content only:** only if the API explicitly supports partial results and clearly marks metadata as invalid.
- **Return fabricated default metadata:** generally avoid silently doing this, because it can mislead callers and hide data corruption.

The final choice has not yet been made in the discussion. A reasonable default is to return a corruption/invalid-metadata error and log enough information for diagnosis without silently overwriting the file.

## 4. Crash recovery checklist

At startup, before accepting requests:

1. Read and validate `next_id.txt`.
2. Scan final book directories.
3. Validate each directory's ID and required files.
4. Find the largest valid existing ID.
5. Reconcile `next_id.txt` so it is greater than every existing ID; fail safely or repair according to a documented policy if the counter is malformed.
6. Remove or recover abandoned temporary directories according to the creation protocol.
7. If an index is later introduced, rebuild or reconcile it against the actual book directories.

Distinguish a process crash from sudden power loss. Process-crash recovery is easier; power-loss durability may require explicit filesystem synchronization.

## 5. Current design summary

| Concern | Current choice |
|---|---|
| Metadata/content storage | Separate files per book |
| ID generation | Persistent `next_id.txt` counter |
| Creation publication | Temporary book directory, then rename |
| Listing | Scan valid final book directories |
| GetBook | Read from disk each time |
| Metadata update | Temp file, then replace by rename |
| Read/write coordination | Reader-writer locks |
| Lock granularity | Per-book locks |
| CreateBook retry semantics | Duplicates allowed initially |
| Corrupt metadata behavior | Not finalized; error is the safest default |

## 6. Interview points to explain clearly

- **Persistent state versus runtime state:** files survive restarts; in-memory maps and locks do not.
- **Atomicity versus durability:** atomic rename helps prevent partial publication, but does not alone guarantee survival after power loss.
- **Thread safety versus process safety:** a mutex only coordinates threads sharing that mutex; separate processes need cross-process coordination.
- **Per-book locks versus global locks:** per-book locks increase concurrency but make lock lifecycle and global operations more complex.
- **Index consistency:** an index is a derived structure and needs a recovery/rebuild strategy.
- **Idempotency:** retrying a request is not automatically the same as executing it exactly once.
- **Error handling:** distinguish a missing book from corrupted or unreadable files; avoid silently inventing data.
- **Start simple:** begin with durable files, atomic publication, startup recovery, and explicit locking; optimize only when requirements or measurements justify it.


