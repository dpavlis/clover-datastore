# 11. Phasing


```mermaid
flowchart LR
  P0["P0 · PoC<br/>Calcite compile on 3 DBs<br/>lease executor + worker pool,<br/>lease vs proxied throughput,<br/>extract managed storage layer"] --> P1["P1 · Store core<br/>shared namespaces, catalog,<br/>Reader/Writer/Query, lease registry,<br/>REST read + small writes + tokens,<br/>ACL + row policy"]
  P1 --> P2["P2 · Run scopes<br/>run namespaces, tree record, lifetimes,<br/>cleanup + sweeper, publish, quotas,<br/>REST upload sessions"]
  P1 --> P3["P3 · Replicas<br/>FULL + DELTA refresh by<br/>paired replica subgraph,<br/>schedules, freshness contract,<br/>retention, Replicas page + approvals"]
  P3 --> P4["P4 · Wrangler<br/>SourceReplicaProvider, replica-fed<br/>previews, source browse, staged files,<br/>lookup index + storeLookup"]
  P2 --> P4
  P4 --> P5["P5 · Advanced<br/>SQL join pushdown for lookups,<br/>DM foreign datasets, replica<br/>change feed, JSON container types,<br/>materialised queries"]
```

Exit criteria per phase:

- **P0**:
  - Base §10 acceptance checks pass.
  - A lease loads at least X rows per second from a worker with the bulk API, and proxied mode is measured for comparison (X to be set).
  - The worker account cannot run DDL, cannot touch catalog tables, and has no grant on a table with row policy.
  - The store's worker pool cannot be opened from a graph DB connection by JNDI name (D31).
  - The DM test suite passes on the extracted storage layer.
- **P1**:
  - Base §10 join and aggregate checks pass from REST and from a graph, with row policy applied below each scan.
  - Without the `dataStore` licence module, every store entry point refuses with a licence message.
  - A namespace linked to a sandbox follows a change of that sandbox's permissions at once.
  - A date written on a node in one JVM time zone reads back as the same instant on a node in another.
  - A truncate waits for a leased writer on another node to finish, and no new lease is issued meanwhile (D32).
  - Two children of one jobflow read the same version of a dataset across a publish between them (D32).
  - A REST client paging through a dataset sees one version across a publish, a repeated write with the same idempotency key writes once, and a token limited to one namespace cannot read another (D34).
- **P2**:
  - A tree with a daemon child, a crash mid-run and a cluster failover leaves no orphan tables after one sweeper interval.
  - With the default switch, a tree that fails or crashes leaves no tables after one sweeper interval. With `datastore_keep_namespace=ON_FAILURE`, a child failure that the root handles keeps the tree's data for `retainOnFailure` (D39).
  - Kept data beyond `datastore.run.keptMaxBytes` is released oldest first (D39).
  - `run.*` in a data service tree is refused with the alternatives named.
  - A scratch namespace is dropped when its TTL expires.
  - `run.*` in a tree whose root has no run record is refused, naming `saveRunRecord` (D30).
  - Purging the execution history does not drop a `RETAINED` dataset before its retention ends, and does not release a namespace while a daemon member runs (D30).
  - Publishing a table created with `publishTo` moves no rows on any of the three databases (D33).
  - A REST upload session is invisible to readers until its commit, and an abandoned one is dropped (D34).
- **P3**:
  - A failed refresh never changes what readers see. Snapshot rollback works for FULL replicas and for `COPY_AND_SWAP` incremental ones; an `IN_PLACE` replica reports that it has none (D35).
  - A run pinned to a `COPY_AND_SWAP` replica reads the same rows before and after an incremental refresh (D35).
  - An incremental replica whose source did not change for longer than `hardExpiry` stays `FRESH` while its refreshes succeed (D36).
  - The reference Salesforce replica subgraph picks up a row committed at the source after a refresh with a timestamp before that refresh's cursor (D36).
  - A connector without a replica subgraph is replicated by running the connector, as FULL. A replica subgraph that answers `FULL` to a `DELTA` request is applied as a swap. `replicaState` is stored only when the refresh commits (D40).
  - The Replicas page lists every configured replica with its last update, time until obsolete and row count. *Purge data* leaves the definition and the next refresh reloads in full (D41).
  - Changing a library's configuration marks its replicas `INCOMPATIBLE`. Upgrading the library to a version with the same connector output keeps them (D38).
  - A source asking for a subset of a replica's `projection` fields is served from it with exactly those columns (D38).
  - A FULL refresh that returns more than `maxShrink` fewer rows is refused.
  - A library upgrade that changes connector metadata marks its replicas `INCOMPATIBLE`.
  - An incremental refresh applies Salesforce tombstones as deletes, and a gap longer than `maxGap` escalates to FULL.
  - Each incremental refresh keeps its delta batch under snapshot retention.
  - A replica requested from Wrangler stays live-read until an admin approves it.
- **P4**:
  - The Wrangler lookup parity suite ([§9.5](09-wrangler.md#95-lookups)) passes.
  - An Excel lookup over 1M rows runs in bounded heap.
  - A recipe over a Salesforce replica produces output identical to the same recipe read live from the same data.
  - Approving a replica with `substitution: OPT_IN` changes the output of no existing job. With `ALL_MATCHING`, the admin UI lists the jobs that switch before the setting is saved (D37).

---

← [10. Components and APIs: delta to Base §4–§8](10-components-and-apis.md) · [Index](README.md) · [12. Decision register](12-decisions.md) →
