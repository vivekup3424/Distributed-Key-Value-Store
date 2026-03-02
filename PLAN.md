# Distributed Key-Value Store — Build Plan

## What are we building?

A **key-value store** is like a dictionary: you store a value under a name, and look it up later.

```
PUT("username", "alice")
GET("username")  →  "alice"
```

What makes this hard is that we're running this on **3 computers at once**. If one crashes, the other two keep working and no data is lost.

---

## The core problem: who's in charge?

When 3 computers share data, they need to agree on one **leader** — the computer that accepts all writes and tells the others what to do.

We use an algorithm called **Raft** to handle this. It works like an election:

- Normally, one node is the leader
- If the leader goes silent (crash, network issue), the others notice and hold an election
- A new leader is chosen once **2 out of 3** nodes agree (called a majority/quorum)

---

## Write flow (what happens when you store something)

```
Client: PUT("color", "blue")
  1. Request goes to the leader
  2. Leader writes it to its log (on disk — survives crashes)
  3. Leader sends the entry to the other 2 nodes
  4. Once 2 of 3 nodes confirm → entry is "committed"
  5. Leader applies it to the in-memory store
  6. Leader replies "success" to the client
```

---

## Read flow (what happens when you fetch something)

```
Client: GET("color")
  1. Request goes to the leader
  2. Leader checks it's still the leader (sends a quick heartbeat)
  3. Leader reads from the in-memory store
  4. Returns "blue"
```

Reads always go through the leader so you never get stale/outdated data.

---

## Glossary (plain English)

| Word | What it means |
|---|---|
| **Node** | One of the 3 computers in the cluster |
| **Leader** | The node currently in charge of accepting writes |
| **Follower** | A node that copies data from the leader |
| **Candidate** | A follower trying to become the new leader |
| **Term** | An election round number — increments every time a new election starts |
| **Log** | An ordered list of every write command ever received |
| **Commit** | A log entry is "committed" once the majority of nodes have it |
| **Quorum** | Majority — for 3 nodes, that's 2 |
| **WAL** | Write-Ahead Log — write to disk *before* doing anything, so you can recover after a crash |
| **Snapshot** | A full saved copy of the current data, so you don't replay the entire log on restart |
| **gRPC** | How nodes talk to each other over the network (like a remote function call) |
| **RocksDB** | A fast on-disk key-value store we use to persist data |

---

## Tech stack (and why)

| Tool | Role |
|---|---|
| Java 17 | Main language |
| gRPC + protobuf | Nodes communicate with each other using this |
| RocksDB | Saves data to disk so it survives restarts |
| Gradle | Build tool |
| Docker Compose | Runs all 3 nodes locally with one command |
| Prometheus + Grafana | Dashboards to monitor the cluster health |
| JUnit 5 | Automated tests |

---

## Project structure (simplified)

```
dkv/
├── proto/          ← Define the network messages (what Get/Put/Vote look like)
├── core/           ← All the real logic lives here
│   ├── raft/       ← Leader election, log replication
│   ├── storage/    ← Writing to disk (WAL, snapshots)
│   └── server/     ← Handle incoming requests from clients and other nodes
├── client/         ← A library clients use to talk to the cluster
├── docker/         ← Run the 3-node cluster locally
└── scripts/        ← Helper scripts (kill a node, run benchmarks)
```

---

## Build phases

### Phase 1 — One node, basic API (Week 1–2)

Goal: a single node that stores key-value pairs in memory.

- Set up the Maven project
- Define `Get`, `Put`, `Delete` in a `.proto` file
- Implement those operations backed by a `HashMap`
- Package into Docker, write a basic test

At the end: `docker run dkv`, then `PUT/GET` works.

---

### Phase 2 — Leader election (Week 3)

Goal: 3 nodes can elect a leader.

- Each node starts as a **follower**
- If a follower doesn't hear from a leader for 150–300ms, it starts an election
- It sends `RequestVote` to the other nodes
- If it gets 2+ votes, it becomes the leader
- Write a test: start 3 nodes, assert exactly one becomes leader

---

### Phase 3 — Log replication (Week 4)

Goal: a write to the leader gets copied to both followers.

- Leader appends the write to its local log
- Leader sends `AppendEntries` to both followers
- Once both respond OK → leader commits and applies it
- Write a test: write to leader, read back from any node

---

### Phase 4 — Durability (Week 5)

Goal: a crashed node can restart and rejoin the cluster.

- Save the log to disk using RocksDB (so it survives crashes)
- On restart, replay the log to rebuild state
- Implement snapshots: periodically save the full state to disk and throw away old log entries
- If a follower is too far behind, send it a snapshot directly
- Write a test: kill a node, restart it, verify it catches up

---

### Phase 5 — Monitoring (Week 6)

Goal: see what's happening inside the cluster.

- Expose metrics: current term, number of leader changes, request latency
- Prometheus scrapes those metrics
- Grafana shows them as graphs
- Add structured JSON logs so you can grep for events

---

### Phase 6 — Chaos testing + benchmarks (Week 7)

Goal: prove the system works under failure.

- Kill the leader mid-write → verify a new leader is elected within 500ms and no data is lost
- Cut off one node from the network → verify the other two keep working → reconnect → verify the isolated node catches up
- Run a load test, record throughput and latency
- Write up results in the README

---

## Node state transitions

A node is always in one of three states:

```
FOLLOWER  ---(election timeout)--→  CANDIDATE
CANDIDATE ---(wins majority vote)→  LEADER
CANDIDATE ---(sees higher term)--→  FOLLOWER
LEADER    ---(sees higher term)--→  FOLLOWER
```

---

## Key correctness properties (what makes Raft safe)

1. **One leader per term** — two nodes can never both think they're leader at the same time
2. **Leader never loses data** — the leader only appends, never overwrites its log
3. **Committed data survives elections** — any future leader will have all committed entries
4. **All nodes apply the same commands in the same order** — so all nodes end up with the same data

---

## References

- [Raft paper (the original research)](https://raft.github.io/raft.pdf)
- [Raft visualized (interactive)](https://raft.github.io)
- [gRPC Java docs](https://grpc.io/docs/languages/java/)
- [RocksDB Java basics](https://github.com/facebook/rocksdb/wiki/RocksJava-Basics)
