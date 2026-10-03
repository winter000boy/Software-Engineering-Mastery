# #040 — Research Paper: *The Google File System* (Ghemawat, Gobioff, Leung)

**Date:** 2026-10-03  
**Roadmap row:** 40 — P1 Java Core · Collections & Generics · *Paper*  
**Paper:** Sanjay Ghemawat, Howard Gobioff, Shun-Tak Leung (Google), *The Google File System*, **SOSP '03**, October 2003  
**PDF:** <https://static.googleusercontent.com/media/research.google.com/en//archive/gfs-sosp2003.pdf>

> These are section-by-section study notes, following the paper's structure. Numbers are quoted as the paper reports them (2003 hardware). The distilled summary, 3 takeaways and next week's plan go in **#041** (row 41).

---

## At a Glance (copy to handwritten notes)

- **What:** a distributed file system for Google's **huge, append-mostly, batch-read** workloads, on **cheap commodity machines** where failures are the norm.
- **Architecture:** **one master** (all metadata, in RAM) + many **chunkservers** (data) + client libraries. **Control goes through the master; data never does.**
- **Files are split into 64 MB chunks**, each with a 64-bit handle and **3 replicas** by default, spread across **racks**.
- **Master metadata:** namespace, file→chunk mapping, chunk locations. The first two are persisted in an **operation log** (+ checkpoints); **locations are not persisted**, they're rebuilt from chunkserver reports.
- **Relaxed consistency:** regions can be *consistent*, *defined*, or neither. Applications cope by **appending, checkpointing, and writing self-validating records** (checksums + unique ids).
- **Record append:** atomic, **at-least-once** append at an offset GFS chooses. It allows duplicates and padding, so readers dedupe.
- **Leases:** the master gives one replica a 60 s lease, making it the **primary**, which picks the mutation order for all replicas.
- **Data flow ≠ control flow:** data is **pipelined along a chain** of chunkservers (nearest-first); control goes client → primary → secondaries.
- **Fault tolerance:** fast restart (no difference between "clean" and "crash" shutdown), chunk re-replication, a replicated op log, **shadow masters**, **checksums per 64 KB block**, **chunk version numbers** for stale-replica detection, **lazy garbage collection**.
- **Big lesson:** design for *your* workload; a single master works if you keep it **out of the data path**.

---

## 1. Why GFS Exists: The Four Observations (§1)

| # | Observation | Design consequence |
|---|-------------|-------------------|
| 1 | **Component failures are the norm.** Hundreds to thousands of cheap machines; disks, memory, connectors, networks, power supplies, OS and app bugs, human errors | Constant monitoring, error detection, fault tolerance and **automatic recovery** are built in, not bolted on (compare Hamilton, #015) |
| 2 | **Files are huge.** Multi-GB files are normal; managing billions of KB-sized files is unwieldy | Revisit block sizes: a **64 MB chunk** |
| 3 | **Most files are mutated by appending**, not overwriting; once written, they're read, often sequentially. Random writes are "practically non-existent" | Optimise **append**; make append **atomic**; **no client-side data caching** |
| 4 | **Co-design the file system and the applications** | A **relaxed consistency model** and a non-POSIX API (snapshot, record append) |

---

## 2. Design Overview (§2)

### 2.1 Assumptions

- Many inexpensive components that **often fail**.
- A **modest number of large files**: a few million files, typically ≥ 100 MB; multi-GB is common. Small files are supported but not optimised.
- Reads: **large streaming reads** (hundreds of KB to 1 MB+) and **small random reads** (a few KB).
- Writes: **large sequential appends**; small random writes are supported but needn't be efficient.
- **Many clients appending concurrently to the same file** (producer-consumer queues, many-way merges) need well-defined semantics with minimal synchronisation.
- **High sustained bandwidth matters more than low latency.**

### 2.2 Interface

A familiar create/delete/open/close/read/write interface, organised hierarchically by pathname, but **not POSIX**. Two extras:

- **Snapshot:** a cheap copy of a file or directory tree.
- **Record append:** many clients append concurrently, and each append is atomic.

### 2.3 Architecture

```
            ┌─────────────────────────── control (small) ───────────────────────────┐
            │                                                                        ▼
   ┌────────────────┐  (file name, chunk index)        ┌────────────────────────────────────┐
   │  Application   │ ───────────────────────────────▶ │            GFS master              │
   │  + GFS client  │ ◀─────────────────────────────── │  namespace · file→chunks ·         │
   │    library     │  (chunk handle, replica locs)    │  chunk→locations (in RAM)          │
   └───────┬────────┘                                  └────────────┬───────────────────────┘
           │ (chunk handle, byte range)                              │ HeartBeat: instructions ↓
           │                                                         │            state      ↑
           ▼  ═══════════════ data (large) ════════════════         ▼
   ┌───────────────────┐   ┌───────────────────┐   ┌───────────────────┐
   │ GFS chunkserver   │   │ GFS chunkserver   │   │ GFS chunkserver   │   chunks stored as plain
   │ (Linux files)     │   │ (Linux files)     │   │ (Linux files)     │   Linux files on local disk
   └───────────────────┘   └───────────────────┘   └───────────────────┘
```

- **Chunk:** fixed size (64 MB), identified by an immutable, globally unique **64-bit chunk handle** that the master assigns at creation. Replicated (3 by default; configurable per namespace region).
- **Master:** all metadata, plus system-wide activities (lease management, garbage collection, chunk migration), and periodic **HeartBeat** messages with each chunkserver.
- **No data caching** by clients (streaming workloads and huge working sets make it pointless, and it avoids cache-coherence problems) **or by chunkservers** (Linux's buffer cache already does it). Clients *do* cache **metadata**.

### 2.4 Single master

One master "vastly simplifies" the design and allows global placement and replication decisions. The risk is that it becomes a bottleneck, so its involvement in reads and writes is minimised.

**A read, step by step:**

1. The client turns (file name, byte offset) into a **chunk index** (offset ÷ 64 MB). That's just arithmetic, thanks to the fixed chunk size.
2. The client sends (file name, chunk index) to the master.
3. The master replies with the chunk handle and replica locations. The client **caches** this, keyed by (file name, chunk index).
4. The client reads directly from a replica, usually the **closest** one.
5. Further reads of that chunk need **no** master interaction until the cache entry expires or the file is reopened. The master often also returns info for the **following** chunks, which saves future round trips.

### 2.5 Chunk size: 64 MB

| Advantages | Disadvantages |
|-----------|---------------|
| Fewer client↔master interactions (one lookup covers a lot of data); a client can cache the locations of a multi-TB working set | A small file = one chunk; if many clients read it, its chunkservers become **hot spots** |
| Persistent TCP connections to a chunkserver for many operations | The paper's real incident: an **executable stored as a single-chunk file**, launched on hundreds of machines at once, overloaded its chunkservers. Fix: a higher replication factor + staggered start times |
| **Less metadata**, so it fits in the master's memory | |
| Lazy space allocation avoids internal fragmentation (each replica is a Linux file extended as needed) | |

### 2.6 Metadata

Three kinds, **all in master memory**:

| Metadata | Persisted? | How it's kept |
|----------|:---:|---------------|
| File and chunk **namespaces** | ✅ | Operation log + checkpoints |
| **File → chunks** mapping | ✅ | Operation log + checkpoints |
| **Chunk → replica locations** | ❌ | **Polled from chunkservers** at master startup and when a chunkserver joins; kept fresh via HeartBeats |

- **In-memory = fast**, and it allows cheap **periodic full scans** for garbage collection, re-replication and rebalancing.
- **Capacity concern:** < **64 bytes of metadata per 64 MB chunk**, and < 64 bytes per file name (prefix compression). Adding RAM to the master is "a small price to pay".
- **Why not persist locations?** The **chunkserver has the final word** on what's on its disks (disks fail, servers get renamed). Keeping a second copy on the master in sync would be harder than just asking.
- **Operation log:** the only persistent record of metadata, *and* the **logical timeline** that defines the order of concurrent operations. It's replicated to remote machines, and a client operation is acknowledged **only after** the log record is flushed locally **and** remotely. Records are **batched** before flushing to limit the throughput cost.
- **Checkpoints:** when the log grows past a threshold, the master writes a **compact B-tree-like checkpoint** (directly memory-mappable). Recovery = load the latest checkpoint + replay the log records after it. The checkpoint is built in a separate thread after switching to a new log file; it takes about a minute for a few million files. A failed checkpoint is detected and skipped.

### 2.7 Consistency model

**Namespace mutations** (e.g. file creation) are **atomic**: the master does them under namespace locks, and the op log gives a global total order.

**File data regions** after a mutation:

| | Write | Record append |
|---|---|---|
| **Serial success** | **defined** | **defined**, interspersed with **inconsistent** |
| **Concurrent successes** | **consistent but undefined** | defined, interspersed with inconsistent |
| **Failure** | **inconsistent** | inconsistent |

- **Consistent:** all clients see the same data, whichever replica they read.
- **Defined:** consistent **and** clients see exactly what the mutation wrote.
- **Concurrent writes** → consistent but *undefined*: the same bytes everywhere, but mingled fragments from several writers.
- **A failed mutation** → inconsistent: replicas may differ.

How GFS achieves "defined after successful mutations": (a) every replica applies mutations in the **same order** (leases, §3.1); (b) **chunk version numbers** detect replicas that missed mutations while down (§4.5). Stale replicas are never used and are garbage-collected.

**The stale-read window:** clients cache chunk locations, so they *may* read a stale replica until the cache entry times out or the file is reopened. Since files are mostly append-only, a stale replica usually returns a **premature end of chunk** rather than wrong data.

### 2.7.2 How applications cope

- **Append rather than overwrite.**
- **Checkpoint:** writers record how much has been successfully written (with application-level checksums); readers only process up to the last checkpoint. Alternatively, write the whole file and then **atomically rename** it.
- **Self-validating, self-identifying records:** checksums let readers skip padding and fragments; **unique record IDs** let readers drop duplicates (from at-least-once appends). This record-I/O logic lives in a **shared library**.

---

## 3. System Interactions (§3)

### 3.1 Leases and mutation order

- The master grants a **chunk lease** to one replica, the **primary**. The primary assigns **serial numbers** to mutations, and every replica applies them in that order.
- Global order = lease-grant order (chosen by the master), then serial numbers within a lease.
- A lease lasts **60 s initially** and is extended indefinitely while mutations continue (piggybacked on HeartBeats). If the master loses contact with a primary, it waits for the lease to **expire** before granting a new one, so there are never two primaries.

**A write, step by step (Figure 2):**

```
1. client → master:  who holds the lease? where are the replicas?
2. master → client:  primary + secondaries (the client caches this)
3. client → ALL replicas: push the DATA (any order; pipelined chain, see §3.2)
                          each stores it in an internal LRU buffer cache
4. client → primary: "write" request (refers to the pushed data)
                     the primary assigns a serial number and applies it locally
5. primary → secondaries: forward the request; they apply it in serial-number order
6. secondaries → primary: done
7. primary → client: success, or the errors from any replica
   (on error: the region may be inconsistent → the client retries steps 3–7, then from the start)
```

Large or chunk-straddling writes are split into several operations, so concurrent writers can interleave them. That's how "consistent but undefined" regions arise.

Note step 3: chunkservers buffer pushed data in an **LRU buffer cache** until it's used or aged out (rows 32–36's data structure, at work in GFS).

### 3.2 Data flow: decoupled from control flow

- Data is pushed **linearly along a chain** of chunkservers, not in a tree, so each machine's **full outbound bandwidth** goes to one recipient.
- Each machine forwards to the **closest** machine (in network topology) that hasn't received the data yet. Distances are estimated from IP addresses.
- **Pipelining:** a chunkserver starts forwarding as soon as it starts receiving.
- The ideal time to push B bytes to R replicas: **B/T + R·L** (T = throughput, L = per-hop latency). With 100 Mbps links and L ≪ 1 ms, **1 MB takes about 80 ms**.

### 3.3 Atomic record append

- The client supplies only the **data**; GFS picks the offset, appends it **atomically at least once**, and returns the offset. It's like `O_APPEND` without the races.
- Used heavily for **multiple-producer / single-consumer queues** and merged results.
- The primary checks whether the record fits in the current chunk. If not, it **pads the chunk** to 64 MB (and tells the secondaries to do the same), and the client retries on the next chunk. A record append is capped at **¼ of the chunk size** to bound fragmentation.
- On failure the client retries, so replicas may contain **duplicates** or partial records. GFS **doesn't** guarantee byte-identical replicas, only that the data is written at least once as an atomic unit at the same offset on all replicas of some chunk.

### 3.4 Snapshot

- **Copy-on-write**, like AFS. The master **revokes outstanding leases** on the affected chunks (so the next write must contact it), logs the operation, then duplicates the **metadata only**. The new files point to the same chunks, and each chunk's **reference count** goes above 1.
- On the first write to a shared chunk C, the master creates **C'** on the **same chunkservers** (a local disk copy, ~3× faster than the 100 Mb network back then) and hands out a lease on C'.

---

## 4. Master Operation (§4)

### 4.1 Namespace management & locking

- There are **no per-directory data structures** and no hard or symbolic links. The namespace is a **lookup table from full pathname → metadata**, compressed with prefix compression.
- Each path has a **read-write lock**. An operation on `/d1/d2/.../dn/leaf` takes **read locks** on `/d1`, `/d1/d2`, … and a **read or write lock** on the full path.
- Example: creating `/home/user/foo` (read lock on `/home/user`) is serialised against snapshotting `/home/user` (write lock on `/home/user`), because the two locks conflict.
- **Concurrent file creation in one directory** works: each takes only a **read** lock on the directory name, plus a **write** lock on its own file name.
- Locks are allocated lazily, and acquired in a **consistent total order** (by namespace-tree level, then lexicographically) **to prevent deadlock**. (The same technique is coming up in rows 42–66, Concurrency I.)

### 4.2 Replica placement

Goals: **maximise reliability/availability** and **network bandwidth utilisation**. Spreading across machines isn't enough; replicas must also be spread **across racks**, so a rack failure (a shared switch or power circuit) doesn't lose every copy, and reads can use several racks' bandwidth. The cost: writes must cross racks.

### 4.3 Creation, re-replication, rebalancing

- **Creation:** place on chunkservers with **below-average disk utilisation**, **limit "recent" creations** per chunkserver (a creation predicts heavy writes soon), and spread across racks.
- **Re-replication** when the replica count drops below its goal, **prioritised** by: how far it is from the goal (a chunk with 2 replicas lost first), live files over recently deleted ones, and chunks **blocking client progress**. Cloning traffic is **throttled** per cluster and per chunkserver.
- **Rebalancing:** a periodic redistribution for disk space and load; new chunkservers are filled **gradually**, rather than swamped with new chunks and the write traffic that comes with them.

### 4.4 Garbage collection

- **Deletion is lazy:** the master logs it and **renames the file to a hidden name** with a deletion timestamp. Regular scans remove hidden files **older than 3 days** (configurable). Until then, the file can be **undeleted** by renaming it back. This is Hamilton's "soft delete only" (#015 §7).
- **Orphaned chunks** (no longer reachable from any file) are found in the master's scan. In HeartBeats, chunkservers report the chunks they hold, and the master replies with the ones it no longer knows about, which the chunkserver can then delete.
- **Why GC instead of eager deletion:** it's simple and reliable when creations and deletion messages can fail or get lost; the work is batched into regular background scans (amortized, and done when the master is idle); and the delay is a **safety net against accidental deletion**. The downside is that storage can't be reused immediately. That's mitigated by expediting a file that's explicitly deleted *again*, and by per-namespace policies.

### 4.5 Stale replica detection

- Each chunk has a **version number**. On every new lease grant, the master increments it and informs the up-to-date replicas, *before* any client can write.
- A replica that was down misses the increment, so when its chunkserver restarts and reports its chunks and versions, the master sees it's **stale**. It treats stale replicas as nonexistent and garbage-collects them.
- If the master sees a version **higher** than its own record, it assumes it failed while granting a lease and adopts the higher version.
- The version number is included when telling clients about lease holders, and in clone instructions, so **clients and chunkservers verify** that they're touching up-to-date data.

---

## 5. Fault Tolerance & Diagnosis (§5)

### 5.1 High availability

- **Fast recovery:** the master and chunkservers restore their state and start **in seconds**, however they terminated. **"We do not distinguish between normal and abnormal termination; servers are routinely shut down just by killing the process."** This is crash-only design, the same idea as Hamilton's "hard-fail it" (#015 §2.1).
- **Chunk replication:** 3 copies by default, across racks. They were exploring **parity / erasure codes** for read-only data.
- **Master replication:** the op log and checkpoints are replicated on multiple machines; a mutation commits only after it's flushed **locally and on all master replicas**. If the master's machine dies, external monitoring starts a new master elsewhere. **Clients use a DNS alias** for the master, so it can move (Hamilton: "always use a CNAME").
- **Shadow masters:** read-only access while the primary master is down. They may lag by a fraction of a second; they replay the op log and poll chunkservers themselves.

### 5.2 Data integrity

- Each chunk is divided into **64 KB blocks, each with a 32-bit checksum**, kept in memory and logged persistently, separately from user data.
- **Each chunkserver verifies its own data independently.** Comparing across replicas is impractical, and replicas may legitimately differ (record-append duplicates).
- **On read:** verify the checksums of the overlapping blocks *before* returning data, so corruption **never propagates**. On a mismatch: return an error, report it to the master, the client reads another replica, and the master re-clones.
- **Appends** update the checksum of the last partial block incrementally; **overwrites** must verify the first and last blocks of the range first, so they don't mask existing corruption.
- **Idle-time scrubbing** of inactive chunks catches corruption in rarely read data, so a corrupted replica can't fool the master into thinking there are enough good copies.

### 5.3 Diagnostic tools

Extensive **diagnostic and RPC logs**, sequential and asynchronous, so the overhead is minimal. They record significant events and every RPC request and reply (minus file data). Matching requests with replies across machines lets them **reconstruct the full interaction history**, and the logs double as traces for load testing. Recent events are kept in memory for live monitoring. (Hamilton §8: "instrument everything".)

---

## 6. Measurements (§6), 2003 hardware

### 6.1 Micro-benchmarks

One master, two master replicas, 16 chunkservers, 16 clients; dual 1.4 GHz PIII machines with 2 GB RAM, 100 Mbps NICs, and two switches joined by a 1 Gbps link.

| Workload | Result | Network limit | Why below the limit |
|----------|--------|---------------|---------------------|
| **Reads** (N clients, random 4 MB regions) | 10 MB/s for 1 client (80% of 12.5 MB/s); **94 MB/s aggregate for 16** (~75% of 125 MB/s) | 125 MB/s (the 1 Gbps inter-switch link) | More readers → more collisions on the same chunkserver |
| **Writes** (N clients, separate files) | 6.3 MB/s for 1 client; **35 MB/s for 16** (~half the limit) | 67 MB/s (each byte goes to 3 of 16 servers) | The network stack interacted poorly with pipelining; collisions are likelier (3 replicas per write) |
| **Record appends** (N clients, **one** file) | 6.0 MB/s → 4.8 MB/s for 16 clients | The last chunk's chunkservers | Congestion. In practice, apps append to *many* files, so this doesn't matter |

### 6.2 Real clusters (A = research & development, B = production)

| | A | B |
|---|---:|---:|
| Chunkservers | 342 | 227 |
| Disk space available / used | 72 TB / 55 TB | 180 TB / 155 TB |
| Files (dead files) | 735 k (22 k) | 737 k (232 k) |
| Chunks | 992 k | 1550 k |
| Metadata at the **master** | **48 MB** | **60 MB** |

- The master holds only **tens of MB**, about **100 bytes per file**, so memory really doesn't limit capacity. A restart reads that metadata in seconds; the master then needs **30–60 s** to learn chunk locations from all the chunkservers.
- The **master handled 200–500 operations/s**. An earlier version was bottlenecked on **linear scans of large directories**; switching to **binary search** fixed it, a Big-O lesson in the wild (#037).
- **Recovery:** after killing one chunkserver (15 000 chunks, 600 GB), everything was re-replicated in **23.2 minutes** (440 MB/s, with cloning throttled to 91 concurrent clones at 6.25 MB/s each). After killing two, the **266 chunks left with a single replica** were prioritised and back to ≥ 2 replicas within **2 minutes**.

### 6.3 Workload breakdown

- Reads are **bimodal** (small < 64 KB seeks, or large > 512 KB sequential); so are writes.
- **Appends dominate** in production cluster Y (writes to record appends ≈ 3.7 : 1 by bytes). **Overwrites were < 0.0001%** of bytes mutated in cluster X and **0.05%** in Y, mostly from **client retries**, not intentional overwrites.
- Master requests are mostly **FindLocation** (~65%) and **FindLeaseHolder**.

---

## 7. Experiences (§7)

- Usage grew from production backends to R&D, which needed **permissions and quotas**, added later. "Users sometimes are not [disciplined]."
- **Disks lied to the Linux driver** about the IDE protocol versions they supported, which **silently corrupted data**. That motivated the **checksums** (and kernel fixes).
- Linux 2.2 `fsync()` cost was **proportional to file size**, which was a problem for the big op logs. Worked around, then they moved to 2.4.
- A single **reader-writer lock** in the kernel's address space blocked the network thread during `mmap` page-ins, causing transient timeouts under light load. They replaced `mmap()` with `pread()`, at the cost of an extra copy.

---

## 8. Related Work & Conclusions (§8–§9)

- Compared with AFS, xFS, Swift, Frangipani, GPFS, Lustre and NASD: GFS uses **only replication** (not RAID or erasure coding), **no caching below the FS interface**, and a **centralised master** ("to simplify the design, increase its reliability, and gain flexibility"), rather than distributed consensus for metadata.
- **Conclusions:** treat failures as the norm; optimise for huge, append-mostly files; extend *and* relax the standard interface; get fault tolerance from monitoring, replication and fast automatic recovery; **separate control (master) from data (chunkservers)**; keep the master off the data path with **large chunks and leases**.

---

## 9. Concepts → Where They Appear in the Roadmap

| GFS idea | Roadmap |
|----------|---------|
| Single master + leases; the risk of a central point | Consensus & coordination, Raft/Chubby (rows 442–466; Chubby paper, row 140) |
| Replication across racks, re-replication priority | Replication & CAP (rows 342–366) |
| Relaxed consistency, defined vs consistent | Consistency models (rows 348, 362–363) |
| At-least-once append, dedupe by unique id | Idempotency keys, delivery guarantees (rows 373, 429) |
| Op log + checkpoint + replay | Write-ahead logs; *The Log* paper (row 390); Kafka (rows 367–391) |
| Chunked, sharded data + a metadata service | Bigtable (row 215), MapReduce (row 65), which ran **on** GFS |
| Checksums, scrubbing | Data integrity in the storage rows (P3) |
| Lazy GC, soft delete | Hamilton (#015 §7); data-layer design |
| Lock ordering to prevent deadlock | Concurrency I: deadlock prevention (rows 54–55) |
| LRU buffer cache on chunkservers | The LRU project (#032–#036) |
| Linear scan → binary search fixed the master bottleneck | Big-O (#037) |

---

## 10. Discussion Questions (answer in #041 or your notebook)

1. Why is a **single master** not a bottleneck here, and under what workload *would* it become one?
2. Why does the master **not persist chunk locations**? What would go wrong if it did?
3. Explain *consistent but undefined* with a concrete two-writer example.
4. Why is record append **at-least-once** instead of exactly-once, and what does the application have to do about it?
5. Why push data along a **chain** rather than from the client to all replicas in parallel?
6. What does the **chunk version number** protect against, and when exactly is it incremented?
7. Compare GFS's lazy deletion with Hamilton's "soft delete only". What does each protect against?
8. Which GFS design choices would you **not** make for a workload of many small files with random overwrites?

---

## Glossary

| Term | Meaning |
|------|---------|
| **Chunk** | A 64 MB unit of file data, identified by a 64-bit handle, replicated (3× by default) |
| **Chunkserver** | A machine storing chunk replicas as local Linux files |
| **Master** | Holds all metadata in RAM; coordinates leases, placement, GC |
| **Lease** | A time-limited (60 s, renewable) grant making one replica the **primary** for mutations |
| **Primary / secondary** | The primary orders mutations; secondaries apply them in that order |
| **Operation log** | Persistent, replicated log of metadata changes; also the logical clock |
| **Checkpoint** | A compact snapshot of master state, allowing log truncation and fast recovery |
| **Consistent / defined** | All replicas show the same data / and it's exactly what the mutation wrote |
| **Record append** | Atomic, at-least-once append at an offset GFS chooses |
| **Shadow master** | A read-only, slightly lagging replica of the master |
| **Chunk version number** | Incremented on each lease grant; detects stale replicas |

---

## Handwritten Cheat Sheet

```
GFS (SOSP'03) — huge files · append-mostly · streaming reads · commodity HW, failures = norm
ARCH: 1 master (metadata in RAM) + chunkservers (data) + client lib
      control via master, DATA NEVER through master; clients cache metadata, NOT data
CHUNK: 64 MB, 64-bit handle, 3 replicas across racks; <64 B metadata/chunk
METADATA: namespace + file→chunks (op log + checkpoints, replicated) · locations (NOT persisted, polled)
CONSISTENCY: serial write = defined · concurrent writes = consistent-undefined · failure = inconsistent
   apps: append, checkpoint, self-validating records (checksums) + unique ids (dedupe)
LEASE: 60 s, primary orders mutations (serial #s); master waits for expiry before a new primary
WRITE: push data (chain, pipelined, nearest-first) → write req to primary → forward → acks → reply
   ideal push time B/T + R·L (1 MB ≈ 80 ms)
RECORD APPEND: at-least-once, atomic, GFS picks offset; pad chunk if no fit; ≤ ¼ chunk
SNAPSHOT: revoke leases → copy metadata → copy-on-write chunk C' on same servers
NAMESPACE LOCKS: read locks on ancestors + R/W on leaf; total order → no deadlock
GC: lazy — hidden rename, deleted after 3 days (undelete possible); orphans via HeartBeat
STALE: chunk version # bumped on each lease grant
FT: crash-only restart in seconds · re-replication by priority · shadow masters · DNS alias
INTEGRITY: 32-bit checksum per 64 KB block, verified before returning data; idle scrubbing
NUMBERS: 16 readers 94 MB/s · 16 writers 35 MB/s · master 48–60 MB metadata, 200–500 ops/s
   1 dead chunkserver (600 GB) re-replicated in 23.2 min; 2 dead → critical chunks fixed in 2 min
```

---

**Next note:** #041 — Paper notes (summary + 3 takeaways) + plan next week (Concurrency I)
