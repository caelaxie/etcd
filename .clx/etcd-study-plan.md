# etcd Study Plan for Kubernetes Users

## Purpose

This study plan is for someone who already understands Kubernetes at a practical level and wants to build a solid etcd mental model without getting lost in implementation details too early.

The path is intentionally staged:

1. Build the operational model first.
2. Learn the API and data model that Kubernetes depends on.
3. Learn routine etcd operations and recovery concerns.
4. Learn the internals that explain the guarantees.
5. Use the codebase only after the concepts are stable.

By the end of this plan, you should be able to:

- explain why etcd is the Kubernetes control plane's source of truth
- reason about revisions, watches, leases, transactions, quorum, and consistency
- use `etcdctl` for basic inspection and maintenance
- understand the major failure and recovery paths
- navigate the etcd repository without guessing

## How To Use This Plan

- Treat each phase as a checkpoint, not just a reading list.
- Do the exercises, even if they seem simple.
- Write short notes in your own words after each phase.
- Do not start reading source code until Phase 4 unless you are blocked by curiosity.

Recommended cadence:

- 4 weeks if you want steady progress
- 2 weeks if you can spend concentrated daily time
- 1 week only if you already know distributed systems terminology

## Dependency Graph

- `T1` Set context and build the top-level mental model. `depends_on: []`
- `T2` Learn CLI operations and the core API concepts. `depends_on: [T1]`
- `T3` Learn operational maintenance, health, and recovery. `depends_on: [T2]`
- `T4` Learn internals: Raft, reads, writes, MVCC, persistence. `depends_on: [T2, T3]`
- `T5` Learn repository structure and code orientation. `depends_on: [T4]`
- `T6` Consolidate knowledge with checkpoints, glossary, and next-track options. `depends_on: [T1, T2, T3, T4, T5]`

## Week-by-Week Schedule

### Week 1

- Complete Phase 1
- Start Phase 2 through watches and leases
- Write a one-page note: "What Kubernetes needs from etcd"

### Week 2

- Finish Phase 2
- Complete Phase 3
- Run the admin-oriented `etcdctl` exercises

### Week 3

- Complete Phase 4
- Draw the write path and read path from memory
- Write a short explanation of linearizable vs serializable reads

### Week 4

- Complete Phase 5
- Review the glossary and checkpoints
- Pick one follow-on track: operator, developer, or internals

## Phase 1: Mental Model and Kubernetes Framing

### Objective

Understand what etcd is, why Kubernetes uses it, and which properties matter before reading any deep internals.

### Read

- Repo overview: [README.md](../README.md)
- Official docs home: <https://etcd.io/docs/latest/>
- Official learning section: <https://etcd.io/docs/latest/learning/>

### Focus Questions

- What problem does etcd solve for Kubernetes?
- Why is etcd a better fit for control-plane state than a general-purpose database?
- What does "distributed reliable key-value store" actually mean in practice?
- Why do quorum and leader health affect kube-apiserver behavior?

### Concepts To Learn

- cluster member
- leader
- follower
- quorum
- revision
- watch
- strong consistency
- failure domain

### Exercises

1. Write a 5-10 sentence summary of how kube-apiserver depends on etcd.
2. Explain, in plain language, what happens when the etcd leader is unavailable.
3. List three reasons Kubernetes needs strong consistency from etcd.

### Exit Criteria

You can explain all of the following without notes:

- what etcd stores for Kubernetes at a high level
- why leader and quorum matter
- what a revision roughly represents
- why watch-based workflows are central to Kubernetes

## Phase 2: CLI, API, and Data Model

### Objective

Turn etcd from an abstract system into something concrete by using the CLI and learning the API model Kubernetes relies on.

### Read

- CLI guide: [etcdctl/README.md](../etcdctl/README.md)
- Go client overview: [client/v3/README.md](../client/v3/README.md)
- Official developer guide: <https://etcd.io/docs/latest/dev-guide/>
- Official API learning material: <https://etcd.io/docs/latest/learning/api/>

### Focus Areas

- basic key-value operations
- range reads and prefixes
- watches
- transactions
- leases
- request timeouts
- linearizable vs serializable reads
- revision-aware behavior

### Recommended Hands-On Commands

Use a local test cluster or disposable environment.

```bash
export ETCDCTL_API=3
etcdctl put app/config "v1"
etcdctl get app/config
etcdctl get --prefix app/
etcdctl watch app/config
etcdctl lease grant 60
etcdctl txn -i
etcdctl endpoint status --cluster --write-out=table
```

### Exercises

1. Create a few prefixed keys and retrieve them with `get --prefix`.
2. Start a watch on one key and a prefix, then observe how updates appear.
3. Create a lease-backed key and explain when it disappears.
4. Write a transaction that updates a key only if a condition is met.
5. Compare a linearizable read with a serializable read in your notes.

### Kubernetes Mapping

Connect each etcd concept to the Kubernetes control plane:

- key/value: stored object state
- revision: change ordering for watch consumers
- watch: API server and controller change propagation
- lease: ephemeral ownership and liveness patterns
- transaction: conditional state updates

### Exit Criteria

You can explain:

- why watches are revision-based
- how leases differ from ordinary keys
- when serializable reads may be acceptable and when they are not
- why Kubernetes components care about ordering and monotonic change streams

## Phase 3: Operations, Maintenance, and Recovery

### Objective

Learn the admin topics that matter when etcd backs a Kubernetes control plane.

### Read

- Official operations guide: <https://etcd.io/docs/latest/op-guide/>
- Sample config: [etcd.conf.yml.sample](../etcd.conf.yml.sample)
- Optional historical example: [hack/kubernetes-deploy/README.md](../hack/kubernetes-deploy/README.md)

Prioritize these official docs topics:

- clustering
- configuration
- security and TLS
- maintenance
- disaster recovery
- tuning

### Operational Topics To Understand

- member health
- endpoint status and endpoint health
- snapshots and restore
- compaction
- defrag
- quota pressure and alarms
- peer/client TLS
- cluster membership changes

### Exercises

1. Run `etcdctl endpoint status --cluster --write-out=table` and explain each major column.
2. Run `etcdctl endpoint health --cluster` and describe what success means.
3. Take a snapshot and inspect it with `snapshot status`.
4. Write a note explaining the difference between compaction and defrag.
5. Write a note explaining why losing quorum is worse than losing a single member.

### Incident Thinking Checklist

When etcd-related issues affect Kubernetes, be ready to ask:

- Is the cluster healthy and does it still have quorum?
- Is a leader elected and stable?
- Are requests timing out because of network or disk problems?
- Is backend size growing because compaction or defrag has been neglected?
- Is there evidence of TLS or certificate problems?
- Is snapshot and restore practice documented and tested?

### Exit Criteria

You can explain:

- what to inspect first during etcd-related Kubernetes incidents
- how backup and restore fit into operations
- why compaction and defrag are routine maintenance topics
- what kinds of failures degrade performance vs break correctness

## Phase 4: Internals and Consistency

### Objective

Understand why etcd's behavior is trustworthy, not just how to operate it.

### Read

- Internal parts diagram: [Documentation/etcd-internals/diagrams/etcd_internal_parts.png](../Documentation/etcd-internals/diagrams/etcd_internal_parts.png)
- Consistent read workflow: [Documentation/etcd-internals/diagrams/consistent_read_workflow.png](../Documentation/etcd-internals/diagrams/consistent_read_workflow.png)
- Leader write workflow: [Documentation/etcd-internals/diagrams/write_workflow_leader.png](../Documentation/etcd-internals/diagrams/write_workflow_leader.png)
- Follower write workflow: [Documentation/etcd-internals/diagrams/write_workflow_follower.png](../Documentation/etcd-internals/diagrams/write_workflow_follower.png)
- Consistency and terminology: [tests/robustness/README.md](../tests/robustness/README.md)

### What To Learn Here

- Raft roles and log replication
- why writes go through the leader
- how consistent reads differ from cheaper reads
- how MVCC enables revisioned history
- how snapshots, WAL, and backend storage fit together at a high level
- what strict serializability means in practice

### Exercises

1. Draw the write path from client request to committed state.
2. Draw the consistent read path and note where leader/quorum coordination matters.
3. Write a short explanation of MVCC using revisions rather than rows or tables.
4. Explain why stale reads are possible in some modes and why that is sometimes acceptable.
5. Explain how watches depend on ordered revisions rather than on wall-clock time.

### Kubernetes Mapping

- controllers depend on ordered change streams
- API server storage semantics depend on consistency guarantees
- recovery and replay depend on durable, ordered state changes
- slow or unstable etcd affects the entire control plane, not just one component

### Exit Criteria

You can explain:

- why a write is not "done" until consensus conditions are satisfied
- why linearizable reads cost more than cheaper local reads
- why revisions are central to correctness
- how Raft and MVCC combine to serve Kubernetes needs

## Phase 5: Codebase Orientation

### Objective

Build a practical map of the repository so future exploration is intentional rather than random.

### Read

- Module boundaries: [Documentation/contributor-guide/modules.md](../Documentation/contributor-guide/modules.md)
- Repo overview again: [README.md](../README.md)

### Major Areas To Recognize

- `client/`: official client libraries and related client-side behavior
- `server/`: etcd server implementation
- `etcdctl/`: operational CLI
- `etcdutl/`: utility tooling for lower-level tasks
- `tests/`: integration and robustness coverage
- `api/`: API definitions and protobuf-related boundaries

### Suggested Orientation Path

1. Read the module boundaries first.
2. Revisit the CLI because it reflects user-facing operations.
3. Skim `server/` only after you know what reads, writes, watches, and leases should mean.
4. Use `tests/robustness` when trying to understand guarantees rather than just mechanics.

### Optional Deep Dives

- `server/etcdserver` for request flow and server behavior
- storage and MVCC-related code paths for revisions and watches
- robustness tests for behavior under faults

### Exercises

1. Make a short map of which directory you would inspect for each of these topics:
   - watch behavior
   - CLI admin commands
   - consistency expectations
   - server request handling
2. Write one paragraph on the difference between "using etcd" and "understanding etcd internals."

### Exit Criteria

You know where to start in the repo when investigating:

- a client-side API question
- an operational tooling question
- a storage or consistency question
- a distributed-systems behavior question

## Checkpoints

### Checkpoint A: Operational Understanding

You can answer:

- Why is etcd so critical to Kubernetes?
- What does it mean for etcd to lose quorum?
- What does a watch stream buy Kubernetes?
- Why is a snapshot not the same thing as a running healthy cluster?

### Checkpoint B: API and Data Model

You can answer:

- What is the difference between a key's value and its revision history?
- What is a lease and when would you attach one?
- Why would you use a transaction instead of separate writes?
- What is the tradeoff between serializable and linearizable reads?

### Checkpoint C: Internals

You can answer:

- Why do writes flow through the leader?
- What work must happen before a write is considered committed?
- Why are watches tied to revisions?
- How do MVCC and Raft serve different roles in etcd?

## Glossary

- **quorum**: the minimum majority of voting members needed to make progress safely
- **leader**: the Raft member that coordinates writes and replication
- **follower**: a non-leader member that replicates log entries and participates in elections
- **revision**: a monotonically increasing version marker for changes to the key space
- **MVCC**: multi-version concurrency control; keeps revisioned history rather than only the latest value
- **watch**: a stream of changes starting from a current or specified revision
- **lease**: a time-bound association that can expire keys automatically
- **compaction**: removal of old revision history that is no longer needed for retention
- **defrag**: reclaiming backend space after deletions and compaction have left fragmentation
- **snapshot**: a point-in-time persisted capture used for backup or recovery workflows
- **linearizable read**: a read that reflects the latest committed state with strong ordering guarantees
- **serializable read**: a cheaper read that may return stale data
- **WAL**: write-ahead log used to persist ordered state changes before broader recovery/use

## Follow-On Tracks

### Operator Track

Choose this if your main goal is running or troubleshooting Kubernetes control planes.

Focus next on:

- backups and restore drills
- TLS and certificate rotation
- endpoint health patterns
- latency, disk, and quota failure modes
- maintenance automation

### Developer Track

Choose this if your main goal is using etcd programmatically.

Focus next on:

- clientv3 usage patterns
- watch handling and retry behavior
- transactions and compare clauses
- lease-backed coordination patterns

### Internals Track

Choose this if your main goal is understanding why etcd behaves the way it does.

Focus next on:

- deeper Raft mechanics
- server request flow
- persistence and snapshot interactions
- robustness testing and fault injection

## Recommended Output After Finishing

When you finish this plan, write one final note with these headings:

- "What etcd guarantees Kubernetes needs"
- "What can go wrong operationally"
- "What I still need to learn"
- "Which repo areas I would inspect first during a real incident"

If you can write that note clearly from memory, the study plan has done its job.
