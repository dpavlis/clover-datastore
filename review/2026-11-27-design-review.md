# Design review: Data Store and replicas

**Date:** 27 November 2026 · **Reviewed:** spec chapters 1–13 and Appendix A at commit `973968c` · **Checked against:** the CloverDX product source (`cloverdx` working copy)

**Scope.** Two areas, as asked: the store itself (for graphs, Wrangler and outside applications over REST) and replicas of data sources. Wrangler lookups and staging (§9.5–9.6) are covered only where they touch those two. The *Base* spec is not in this repository, so anything it may already answer is marked **(Base?)**.

**How to read the findings.** Each has an id (`S` store, `R` replica, `F` fit with the product), a severity, and a basis:

- **Severity:** *High* = the design does not work as written, or loses or exposes data. *Medium* = works, but will hurt in practice. *Low* = wording, or a small gap.
- **Basis:** *code* = checked in the product source (path given). *spec* = two places in the spec disagree. *reasoning* = follows from how the databases or the source systems behave; not tested.

---

## 1. Verdict

The overall shape is sound and fits the product: a separate subsystem beside the Data Manager, namespaces as the unit of isolation, pointer swaps for atomic publish, replicas as ordinary datasets with a source binding, and one Server-side hook for Wrangler. None of the findings below asks for a different architecture.

Three things need a design change before P0, because later phases build on them:

1. **The run namespace key and its cleanup rule do not match how run records behave** (S1, S2). Non-persistent runs have no run record and a negative id, and the execution history is purged after 30 days, one node at a time. As written, the sweeper would drop data under a running job and would drop `RETAINED` data early.
2. **Snapshot pinning, rollback and reproducibility do not hold for incremental replicas** (R1). An incremental refresh merges into the current table in place, so there is no immutable snapshot to pin.
3. **Leases make ACL a soft boundary by default, and the bypass needs no Java code** (S3). If the store's worker pool is registered the way `worker.jndi.datasource` pools are, any graph can open it by JNDI name.

For replicas, the biggest practical risk is not technical. **Approving a replica silently switches every existing matching Wrangler job from live data to data up to a day old** (R7), and identity-only matching will fragment into many near-identical replicas (R9).

The spec says little about the two things you named as the main purpose: the REST API for outside applications is one table row (S12), and the front end is one line per screen. Sections 5 and 6 propose both front ends.

---

## 2. Data store

### 2.1 Where the code contradicts the spec

**S1 · High · code — A run without a run record breaks the namespace key and the sweeper.**
§7.2 keys the namespace by root run id and §7.6 has the sweeper release a namespace whose root run record is gone. The spec treats only Wrangler previews and the profiler as record-less. In the code, more runs are non-persistent:

- a schedule or task with `saveRunRecord=false` (`S/persistent/TaskExecute.java:52`)
- every data service request by default (`S/persistent/DataServiceJob.java:95`)
- every child of a non-persistent parent, which inherits the flag (`ServerAuthorityProxy.java:1348`)

Such a run gets a **negative id from an in-memory sequence** (`S/graph/runtime/RunIdSequenceGenerator.java`). So:

- The orphan rule releases its namespace at the next sweep, while the job is still running.
- A negative id is not unique over time or across nodes, so two trees can share `run_<id>`.

*Fix:* refuse `run.*` when the root is non-persistent, with an error that names the `saveRunRecord` setting. Or key the namespace by a store-issued id and register the tree in the store catalog at first use, so the store does not depend on the run record existing.

**S2 · High · code — History purge drops retained data, and can drop it under a running member.**
The archivator deletes run records older than 30 days by default, per node, with no tree awareness (`S/tasks/TaskProcessorArchivator.java:462-533`; `installer.basic.properties:93-100`). Deleting a sandbox, or importing a server configuration, deletes all its run records too (`PersistenceServiceImpl.java:612`). Combined with the orphan rule in §7.6:

- A `RETAINED` dataset with a retention longer than the history retention is dropped when the root record is purged. That defeats the "audit evidence" use in §7.3.
- A root that finished long ago can be purged while a daemon child still runs. The namespace is then released under it, against "tables are never dropped under a running member".
- After the purge, nobody can "see the root run in the execution history", so the access rule in §7.3 has nothing to check.

*Fix:* the store keeps its own record of the tree (root id, sandbox, job file, user, start and end time, outcome) in the namespace row, written at creation and at release. Retention and access then use that row, not the run record. The orphan rule applies only to `ACTIVE` and `DRAINING` namespaces, and only after the member query also finds nothing running.

**S3 · High · code — The worker pool is reachable from an ordinary graph.**
§5.3 says the store "configures a worker datasource automatically", following `worker.jndi.datasource[n]`. Those datasources are registered as JNDI resources in the worker (`cloveretl.server.worker/.../WorkerApplicationConfig.java:256-286`), and a graph DB connection can name a JNDI resource. So the bypass is not "arbitrary user Java code", as the trust-boundary paragraph says. It is a `DBInputTable` with a JNDI connection. With the default `lease.mode=ALL`, any user who can run a graph can then read and change every table without a row policy: other trees' run tables, replicas (which D12 restricts to the connector's execute groups) and staged files.

*Fix:* keep the pool private to the lease executor and never bind it in JNDI. Then state the remaining boundary honestly in §5.3 and G5: with leases on, ACL is enforced on platform paths and is *advisory* against code running in the worker. Consider `SYSTEM_JOBS` as the default and `ALL` as the opt-in, since the store's own bulk jobs (refresh, staging, index builds) are where the throughput matters.

**S4 · Medium · code — `datastore.run.keepNamespace` cannot be a job config property as described.**
§7.7 says it is read through `JobConfigDataSourceFactory`, "which already serves the job log level". That class is only a lookup-context holder (`S/graph/runtime/JobConfigDataSourceFactory.java:31-62`). The real mechanism is `GraphConfigManager.getPropertyValue`, and its keys are a **closed engine enum** with underscore names (`cloveretl.engine/src/org/jetel/jobconfig/PropertyKey.java:28-50`). An unknown key is stored as an "additional" property and becomes a graph parameter. The same applies to "quota overridable per sandbox and per job config" in §7.12.

*Fix:* add `datastore_keep_namespace` (and the quota keys) to `PropertyKey`, which is an engine change to list in §5.1.

**S5 · Medium · code — Data service trees are recognised only on the node that runs the root.**
D23 relies on the method of `JobQueue.isChildOfDataService`. That looks the root up in the **local** running-jobs map (`S/job/queue/JobQueue.java:657-668`), so a child delegated to another cluster node is not recognised and would get a run namespace. *Fix:* carry a "data service tree" flag in the execution command, or resolve it at the store from the root's job type.

**S6 · Low · code — Smaller inaccuracies.**

- §7.13 names `ExecutionTreeController` for the new *Data* tab. The tabs are owned by `ExecutionsController` (`detailTabIds`, around line 109) and `executions_right.xml`. `ExecutionTreeController` only drives the hierarchy tree.
- §7.6 lists "timeout" among the job end events. The `TIMEOUT` event does not mean the job ended (`S/persistent/enums/GenericJobEventType.java`). Use the run status, not the event, and check that `Result.isStop()` treats `TIMEOUT` the way §7.5 assumes.
- §3 says the preview cache has a max age of 3600 s. The age is not checked when a cache file is used, only by a cleanup that runs every 10 minutes (`WranglerManager.java:3248`).
- §8.4 cites an `includeDeleted` attribute. The attribute is `readMode` with value `includeDeleted` (`AbstractSalesforceReader.java:37`).
- Job end events are delivered asynchronously and best-effort, and other cluster nodes receive only a wrapper event (`ServerEventNotificator.java:96-205`, `ClusterInfoGatherer.java:1437`). The sweeper in §7.6 already covers this, so it is correctness-critical, not a backup. `datastore.maintenance.interval` therefore bounds how long temporary data outlives a tree.

### 2.2 Where the spec contradicts itself

**S7 · High · spec + reasoning — Publish `REPLACE` is not O(1) on two of three dialects as specified.**
§7.4 says `REPLACE` is "O(1) and atomic on every dialect". Two other sections make that false:

- §7.11 creates `TEMPORARY` tables `UNLOGGED` on PostgreSQL. Publishing one into a shared dataset needs `ALTER TABLE … SET LOGGED`, which rewrites the whole table and writes it to the WAL.
- §7.11 and `datastore.run.schema` allow run tables in a separate DB schema. Oracle cannot move a table between schemas without copying it. Either the publish copies, or the published table stays in the run schema with a `DSR_` name forever, which defeats the separate accounting.

*Fix:* add a create option `publishable` (or a `StoreWriter` target "build for publish into `crm.customers`"). Such a table is created logged, in the shared schema, with the target's indexes, and is still listed in the run namespace until it is published or dropped.

**S8 · High · spec — Locks and schema checks in §7.8 cannot work for leased writers.**
§7.8 uses a "catalog row lock" for replace and truncate, and makes "writers holding an older schema version fail fast". §5.3 gives the worker account no rights on catalog tables and has the worker execute a precompiled statement. A leased writer therefore cannot take or see a catalog lock, and nothing tells it the schema moved. If the core holds the row lock on the writer's behalf, each active writer pins a core connection and an open transaction for its whole duration.

*Fix:* add an **active lease registry** in the catalog (dataset, physical table, kind, run id, node, issued, heartbeat). Replace, truncate, alter and drop check it and wait or fail. The same table then answers four other open points: snapshot pins (§8.6), the grace period for `RETIRED` tables (§7.4), forced release (§7.6), and "who is using this dataset right now" for the admin UI.

**S9 · Medium · spec — Pinning is described at three different scopes.**
§6.2: readers resolve the pointer "once per statement". §8.6: a job pins "when it is converted or starts" and "the whole run" reads that snapshot. §5.3: the lease, issued "per component and statement", holds the pinned tables. Two children in one jobflow that read the same replica can therefore see different snapshots. *Fix:* pin per **tree** (the first resolution in a root run fixes the snapshot for all members), which the run namespace already gives a place to store. A REST client that pages needs the same thing: a snapshot token returned with page one (S12).

**S10 · Medium · spec — Permission model overlaps.**

- `data_store_create` is "create datasets in namespaces where granted" in §5.6, but §6 lets its holder create shared and scratch **namespaces**. `MANAGE` already covers creating datasets. Decide: `data_store_create` = create namespaces; `MANAGE` = create datasets in one.
- A replica can live in a shared namespace (`crm.sf_account`). A namespace `WRITE` or `MANAGE` grant, including one derived from sandbox `write` (D24), then lets users write to or drop a dataset whose content "is owned by a source binding". State that a replica dataset is read-only to everyone but its refresh, and can be dropped only through the replica definition.
- §6.2 models "replica" twice: as a namespace kind and as `DATASET ||--o| REPLICA_DEF`. The second is the real one.
- `REPLICA_DEF.source_kind` lists `CONNECTOR JOB FILE`; §8.1 also has `QUERY`.
- The namespace `state` enum is the run lifecycle; it has no meaning for shared namespaces.
- Reserved names are not stated. A user-chosen name must not be `run`, `run_*`, `replica`, `dm`, `staged`. Scratch creation is "idempotent for the same creator and name", so another user can pre-create `tmp_<id>` and block or capture a jobflow. Reserve a prefix per creator or fail loudly.

**S11 · Medium · spec — Kept run data widens who can see restricted rows.**
§5.6 says data copied from a row-policy dataset "keeps its policy… because the policy was applied when it was read". After the run, §7.3 makes kept and retained data readable by everyone who may see the root run. In the code that is everyone with **read on the sandbox** (`ServerFacadeImpl.java:2016-2088`); there is no owner check. So rows that only the run's user could read under the row policy become readable to the sandbox's readers for `retainOnFailure`. The same holds for ACL. This is like edge debug data today, but the store promises a hard boundary (G5). *Fix:* kept data is readable by the run's user and by `data_store_admin`; wider access is opt-in per sandbox.

### 2.3 Architecture weaknesses

**S12 · High · reasoning — The REST API for outside applications is not designed.**
§10 gives one row of routes, P1 delivers "REST read", and no phase delivers REST write. For the stated purpose (any outside application) the spec needs at least **(Base?)**:

| Topic | What to decide |
|---|---|
| Identity | Service accounts or API tokens for applications, scoped to namespaces. Today the principal is "the authenticated user" only. |
| Reading | Cursor (keyset) paging with a snapshot token, so pages stay consistent across a publish or refresh. Offset paging on millions of rows is slow and inconsistent. |
| Writing | Bulk ingest format (NDJSON, CSV), upsert by key, an idempotency key so a retried request does not duplicate rows, and the expected schema version. |
| Staged ingest | The REST equivalent of build-then-publish: open an upload session, send chunks, commit. It maps onto a scratch namespace plus `PUBLISH`. |
| Long work | Asynchronous queries and loads with a status resource, since `datastore.query.timeout` is five minutes. |
| Limits | Per-principal rate and size limits; clear error model. |
| Discovery | Schema as JSON Schema, plus the OpenAPI document. |

**S13 · Medium · reasoning — The catalog in the store DB needs an owner identity.**
D5 puts the catalog beside the data, while users, groups, sandboxes and run records stay in the system DB and are referenced by id. Run ids restart when a system DB is recreated, and D2 allows the store to share the DM database, which test and staging servers sometimes point at together. Two servers on one store DB, or a restored system DB, would make each sweeper see the other's namespaces as orphans and drop them. *Fix:* store the server (cluster) instance id in the catalog, refuse to start against a catalog owned by another instance, and document that the two databases are backed up and restored as a pair. Also define what happens to grants when a user or group is deleted (no foreign key can enforce it).

**S14 · Medium · reasoning — Cross-dialect behaviour is unspecified where it changes results.**
D6 lists types but not semantics. These differ between the three databases and will show up as "the same job gives different output":

- Oracle stores an empty string as `NULL`.
- SQL Server's default collation is case-insensitive, and it ignores trailing spaces in comparisons. A key such as a 15-character Salesforce id, which is case-sensitive, would collapse distinct rows in a `MERGE`. Key columns need a binary collation.
- `number` (double) can hold NaN and infinity in CloverDX; SQL Server `float` cannot.
- Unbounded `string` maps to `nvarchar(max)` or `CLOB`, which cannot be indexed or used as a key.
- Sort order for strings depends on the database collation, so a sorted *browse* differs from a Wrangler sort.
- Column naming is not specified at all: field names that are reserved words, too long, or differ only by case.

*Fix:* add a short "portable semantics" section to §5.5 and make each row an acceptance check in P1.

**S15 · Medium · reasoning — Failed runs can fill the store and block production.**
D21 keeps the data of any tree with a failed member for 24 hours. Quotas are per run namespace, with no global budget. A job that fails every five minutes keeps 288 namespaces a day. If `maxDataStoreBytes` is set, the store then refuses writes that grow storage, so debug leftovers block production loads. *Fix:* a global budget for `KEPT` data with oldest-first eviction, as `staged` already has. Evict `KEPT` and `staged` data before refusing any write under the licence limit.

**S16 · Medium · reasoning — Quota enforcement is described as exact but cannot be.**
§7.12 counts rows "exactly at write time", but leased bulk loads run in workers on several nodes and a `COPY` cannot be stopped cheaply mid-stream. Byte sizes "refreshed on commit" lag on Oracle. *Fix:* say that quotas are checked when a lease is issued and when it completes, so a namespace can overshoot by the size of the loads in flight.

**S17 · Medium · reasoning — Operational points the spec does not mention.**

- **Statistics.** A freshly bulk-loaded or swapped table has no optimizer statistics. Joins from `StoreQuery` will get bad plans until autovacuum or a stats job runs. `ANALYZE` (or the dialect's equivalent) should be part of load commit, publish and refresh.
- **Indexes and keys.** Apart from the lookup index, the spec never says how a dataset gets a primary key or secondary indexes, or what happens to them on publish (the new physical table must have the target's indexes before the swap) **(Base?)**.
- **Connection budget.** One worker per node, each with its own pool, plus the core pools. A graph with twenty store components holds twenty connections. There is no `datastore.*.maxPoolSize` key, and with `type=datamanager` bulk loads compete with the Data Manager UI for the same pool.
- **Lease expiry.** `datastore.lease.ttl=PT10M` must mean "start within", not "finish within", or long reads fail. Aborting a job must cancel the statement in the worker.
- **Audit.** §4.1 promises per-operation audit. No table, retention or screen is specified.
- **Export and import.** §6.1 exports namespaces, grants and links. Dataset schemas, row policies and replica definitions are not listed, yet they are what a dev-to-prod promotion needs. The DM precedent exports the layout (`S/persistent/DataSetXml.java`).
- **Store DB outage.** State that the Server starts without it, and that only store calls fail.

**S18 · Low · reasoning — Restart starts a new tree.**
"Restart job" always creates a new root (`S/job/JobExecutor.java:409-431`). A restarted child of a failed tree therefore cannot see the `run.*` data the failed tree kept, which is the data it most likely needs. Either document the pattern (pass `run_<oldRoot>` as a parameter and read it read-only) or let a restart adopt the kept namespace.

---

## 3. Replicas

### 3.1 Logical inconsistencies

**R1 · High · spec — Incremental refresh breaks pinning, rollback and reproducibility.**
A FULL refresh builds a new table and swaps, so every snapshot is an immutable table. An INCREMENTAL refresh (§8.4) merges the delta **into the current table**. Yet:

- §8.6 says a pinned run "reads that snapshot, even if a refresh swaps the pointer meanwhile". For an incremental replica nothing is swapped; the table changes under the reader. A single statement stays consistent on PostgreSQL and Oracle, not on SQL Server without snapshot isolation, and never across statements or pages.
- §8.7 rule 1 offers "re-point to the previous snapshot" and diffs between refreshes. After a merge there is no previous table.
- P3's exit criterion "snapshot rollback works" cannot be met for incremental replicas.
- "The run's tracking records the snapshot, which makes a run reproducible" does not hold either.

*Options:* (a) say plainly that incremental replicas give statement-level consistency only and no rollback. (b) Apply the delta to a **copy** and swap: `CREATE TABLE AS` inside the database, merge, swap pointer. That costs one table copy per refresh and no source calls, and keeps every promise. I recommend (b) as the default, with in-place merge as an option for very large objects, shown as such in the UI.

**R2 · High · spec — `asOf` for incremental replicas makes quiet objects expire.**
§8.5 defines `asOf` as "the watermark for INCREMENTAL", and §8.4 defines the watermark as the maximum `watermarkField` seen. If the source object does not change for a week, the watermark does not move, so `now − asOf` passes `hardExpiry` and the replica is refused although every refresh succeeded and it is exactly current. *Fix:* `asOf` is the start time of the last successful refresh for both modes. The watermark is a cursor, not a freshness measure.

**R3 · High · reasoning — The watermark rule loses rows.**
Reading "modified after max(watermarkField)" misses a row whose transaction committed after the refresh ran but carries an earlier timestamp. This is a known property of timestamp cursors, including Salesforce's `SystemModstamp`. *Fix:* the capability declares a `lookback` (for example `PT15M`); each refresh re-reads from watermark minus lookback, and the merge by key makes the overlap harmless. Also declare whether the comparison is `>` or `>=`.

Two related cases that only `fullReconcile` repairs, so it must be mandatory for incremental replicas and not an optional line:

- **Rows that leave the filter.** With an identity parameter such as `WHERE`, a row updated so it no longer matches is never returned again, and stays in the replica.
- **Values that change without touching the watermark**, such as Salesforce formula fields.

**R4 · Medium · spec — The state model mixes four independent things.**
`state` in §8.5/§8.9 is one enum holding lifecycle (`REQUESTED`, `EMPTY`, `DORMANT`, `INCOMPATIBLE`), freshness (`FRESH`, `STALE`, `EXPIRED`), activity (`REFRESHING`) and last result (`REFRESH_FAILED`). Consequences:

- While `REFRESHING`, the state no longer says whether the served data is fresh or stale.
- `REFRESH_FAILED` is routed to `onStale` in the §9.2 flowchart even when the data is still inside its freshness target, so a job with `onStale=FAIL` fails for no reason.
- The same flowchart lets `EXPIRED` reach `USE_WITH_WARNING`, against "an expired replica is never served" (§8.6).
- A first load that fails goes `EMPTY → REFRESHING → REFRESH_FAILED` with no snapshot, yet "in `REFRESH_FAILED` the previous snapshot is still served".
- There is no paused or disabled state, and no path to delete.
- §8.5 says the fields are "stored per snapshot", but `state`, `lastAccessAt` and `accessCount` belong to the replica.

*Fix:* four fields. `lifecycle` (REQUESTED, ACTIVE, PAUSED, DORMANT, INCOMPATIBLE), `freshness` computed from `asOf` at read time (NONE, FRESH, STALE, EXPIRED), `activity` (IDLE, REFRESHING), `lastResult` (OK, FAILED with reason). The UI in section 6 shows them as separate columns.

**R5 · Medium · spec — Row numbers and order differ between live, preview and browse.**

- §8.4: in an incremental replica new keys get the next `__src_row` and deleted keys leave gaps. §9.4 step B takes the Wrangler row number "from `__src_row`", while a preview (step A) numbers rows 1..N with the unchanged row filler. The same row gets different numbers in the two views.
- Row order in an incremental replica drifts from the source order. A lookup keeps "the first record per key, in arrival order" (§9.1), so with duplicate keys the replica and the live source can pick different rows. P4's exit criterion "output identical to the same recipe read live" cannot hold as written. Restate it as: the same set of rows; the same result for lookups with unique keys; a documented rule for duplicates.

**R6 · Medium · spec — A replica with a row policy cannot be loaded fast, and its derived data leaks.**

- §5.3 rule 1: the worker account has no grant on a row-policy table, so it is "always served proxied". The refresh writer then also goes through the core, for exactly the large CRM objects where §8.8 recommends a row policy. *Fix:* load into a staged table without the policy, attach policy and revoke the grant at the swap.
- A lookup index (§9.5) is keyed by snapshot and key specification, not by user. Built from a row-policy replica it holds every row. Either refuse the index path for row-policy sources or apply the policy when probing. The same question applies to delta batches and the change feed.

### 3.2 Practical usability

**R7 · High · reasoning — Approving a replica changes existing production jobs without telling their owners.**
§9.3: a `.wsrc` without a `replica` element behaves as `PREFER`, and unattended runs default to `USE_WITH_WARNING` (D17). So the moment an admin approves a replica, every existing Wrangler job with the same identity, including scheduled ones, switches from live data to data up to `freshnessTarget` old. A job that reads Salesforce, writes back, and reads again breaks silently. The only signal is a line in its log.

*Fix:* (1) the approval dialog lists the sources and jobs that will switch (section 6.2); (2) a replica has an "applies to" setting: *sources that opt in* (default) or *all matching sources*; (3) a job's run summary states which sources were read from a replica and as of when.

**R8 · High · code — The identity misses things that change the content, and includes one that should not.**

- **Library configuration.** A connector's credentials and endpoint come from the library's `library.prm` and its OAuth2 connection, which an admin can change in place (`LibraryManager.saveLibraryConfiguration`, `S/library/LibraryManager.java:429-447`). Pointing the same library at another Salesforce org changes every row and leaves "library code + connector path + parameters" unchanged. The existing preview cache handles this by invalidation (`notifyLibraryChange`), and §8.2 already hooks the same notification. Say explicitly that a configuration change marks the replica for a forced FULL rebuild and is not served until then.
- **Library version.** §8.3 says "the library version is not part of the identity". The `libraryCode` that a `.wsrc` stores is the sandbox code, which contains the version: `DataCatalogCoverageLib_0.6` (`E/library/LibraryUtils.java:287-291`). If the identity uses that code, every upgrade orphans all replicas. Use the library's name and id without the version. *(I could not confirm how Wrangler sources are re-pointed on upgrade; that mechanism decides this.)*
- **Secure parameters.** They are stored encrypted in the `.wsrc` and enter the existing cache hash in encrypted form (`WranglerManager.java:2870-2881`). The replica instance would store them in the catalog and show them in the admin UI and REST. Hash them, never display them.
- **`${…}` resolution (D15).** D15 resolves references "against the sandbox's parameters… the same values the generated graph would see". The generated graph links no parameter file (`MGraph.addParameterFile` has no callers), so it is unclear that these references resolve at all today. Confirm the behaviour before building D15 on it.

**R9 · High · reasoning — Identity-only matching will fragment.**
With `FIELDS` and `WHERE` as identity parameters, two users who pick different field lists get two replicas, two approvals, and two nightly extractions of the same object. General predicate pushdown was rightly deferred (Q8), but two cheap steps avoid most of the sprawl:

- **Projection subsumption.** Let the capability mark a parameter as `projection` (a field list). A replica that holds a superset of the requested fields serves the request; `STORE_READER` selects the columns. No author-written predicate mapping is needed.
- **Explicit choice.** Show replicas in the Data Catalog as sources a user can pick directly ("Salesforce Account · replica · as of 01:12"), next to the live connector. This is `WStoreSource`, which §5.1 already lists. Transparent substitution stays, but the explicit route does not depend on parameter matching and makes the staleness a visible choice.
- Define "normalised" in §8.3: trimming, list order and case for list-valued parameters.

**R10 · Medium · reasoning — Refresh timing rules have no precedence.**
A replica has a cron `schedule`, a `window`, retries with backoff, on-demand "Refresh now", `REFRESH_AND_WAIT` from jobs, and escalation to FULL after `maxGap`. The spec does not say which wins. Decide and write down:

- Does "Refresh now" by a reader, or `REFRESH_AND_WAIT` from a job at 14:00, override a 22:00–05:00 window? (Suggest: readers no, admins yes with a confirmation.)
- May an automatic escalation to FULL run outside the window? A FULL load of a large object is the expensive call the window exists to prevent.
- After the Server was down over a scheduled time, does the refresh run at start-up?
- In which time zone are `schedule` and `window`?
- `maxConcurrentRefreshes=4` is global. Source protection needs a limit **per library instance**, since four parallel extractions against one Salesforce org are the problem, not four against four systems. A daily call budget per library would match how such APIs are actually limited.

**R11 · Medium · code — Where the schedule lives.**
§5.1 adds `Task.TaskType.datastore_replica_refresh`, and the replica instance also has a `schedule`. Task types are user-visible in the Schedule section (`data_manager_maintenance` appears there as "Data Manager Expired Row Purge"), and Quartz only fires user schedules (`S/scheduling/QuartzManager.java`). So either each replica creates a visible Schedule in the system DB, duplicating a field of the catalog in the store DB, or the refresh is an internal timing job (`S/timing/*Job.java`). *Recommend:* internal timing job, with the schedule owned by the replica and shown read-only in the Schedule list as a linked entry, so operators see all nightly load in one place.

The refresh is "an ordinary server job" (§8.4), and a run record needs a sandbox and a job file. Say where the generated wrapper graph lives and who can read its log. A system sandbox per store, readable by `data_store_admin`, fits.

**R12 · Medium · code — "Request and approve" has no mechanism to build on.**
D16 notifies data store admins and tracks requests. The product has no in-app inbox and no request queue; the nearest thing is a `mailto:` link in the catalog (`wrangler/modules/catalog/components/connectors/GetAccess.vue`), and email templates take address strings, not groups. This is new work to size in P3. To keep it usable:

- An **auto-approve policy** per library: up to N replicas or M rows without a human, so admins are not a queue for every analyst.
- At approval, choose **first load now** or **at the next window**. Otherwise a replica approved at 10:00 stays `EMPTY` until night and the requester sees nothing happen.
- Tell the requester the outcome, and expire requests nobody handles.
- Before approval the admin cannot know the size or the API cost. Show the preview row count and field count, and the real numbers after the first load.

**R13 · Medium · reasoning — A library upgrade can take all its replicas out of service at once.**
A changed output schema marks a replica `INCOMPATIBLE`: "not served, and the next refresh is a FULL rebuild". After an upgrade that adds one field, every replica of that connector is unserved until rebuilt. `REQUIRE` jobs fail, and `PREFER` jobs all go live in working hours, which is the stampede that `window` is meant to prevent. *Fix:* treat additive changes as compatible; rebuild in the background and swap; and show the affected replicas in the library upgrade dialog before the admin confirms.

**R14 · Medium · reasoning — Validation covers one failure mode.**
`maxShrink` guards a FULL swap against a large drop. Not covered: the first load (no baseline), an incremental refresh that returns a storm of tombstones, and a connector that succeeds with columns suddenly empty (a revoked field permission). Add `minRows`, a tombstone-rate limit for incremental, and optionally a per-column null-rate check against the previous snapshot. A failed validation should keep the staged table for inspection, with a "swap anyway" action for the admin.

**R15 · Low · reasoning — Smaller points.**

- **Default grant.** "Default `READ` = the connector's execute groups" (D12) should be evaluated live, like D24's derived grants. Copied at creation, it drifts when connector permissions change. Note also that executing a connector needs sandbox `execute` **and** group membership (`Security.checkExecutePermission`); the replica grant should mirror both.
- **Compliance purge** (§8.7 rule 4) removes a key from the replica, but the next FULL refresh brings it back unless the source deleted it. Say so. The purge must also cover lookup indexes, staged copies and kept run data built from the replica.
- **Retention cost.** `snapshots: 3` of a multi-gigabyte object is four times its storage. Show the cost next to the setting.
- **Lookup index rebuild.** The index is keyed by snapshot, so it is rebuilt after every refresh, during the first preview that needs it. Rebuild it after the refresh for key specifications used recently.
- **Container types.** D11 stores the connector's native output; D6 defers list, map and variant to P5. A connector with such a field cannot be replicated in P3. State the refusal and the message.
- **Naming.** A "`FILE` replica" has no instance, schedule, approval or admin; it is a cache keyed by fingerprint. Call it a *staged dataset* everywhere and drop `FILE` from the replica kinds. In Wrangler the thing being shadowed is called a *data source connector* in the Data Catalog and a *data source* once added to a workspace (`catalog/messages.ts`); "connector replica" is the term users will recognise.
- **No Salesforce connector ships today** (only test subgraphs). The `library.json` example in §8.2 assumes one with a `MODIFIED_SINCE` parameter. Building the reference connector is P3 work to list.

---

## 4. Fit with the existing product

What fits well, confirmed in code:

- **Data Manager precedent.** Own datasource, own patch stream, `IAuthorityProxy` calls, `MetadataProvider` at design time, public OpenAPI-first REST, MCP tool category with its own permission (`mcp_data_manager`). The store follows each.
- **Naming (D3).** The `store_*` / `data_store_*` names are free, and the existing MCP tools are all `dataset_*`, so the collision D3 avoids is real.
- **Run tree.** `root_run_id` is set on every child path the spec lists, including daemons and cluster partitions (`ServerAuthorityProxy.java:417-466`, `MasterStartingStrategy.java:236`).
- **Web UI location.** `src/server/` holds Vue apps mounted inside the JSF console, and `data-sets` is the model to copy.

Where it does not yet fit:

**F1 · Medium — Events and alerts.** The product's alerting is event listeners with tasks (email, run a job) and the *Alerts and Notifications* tab linked to Operations Dashboard monitors (`S/web/gui/AbstractAlertsController.java`). The spec defines no store events. Add event types (`replica refresh failed`, `replica stale`, `replica expired`, `replica requested`, `namespace draining too long`, `store quota reached`) so that existing listeners and the dashboard work without a second notification system. This also gives R12 its delivery channel.

**F2 · Medium — Data Catalog.** §4.3 adds store datasets as a third catalog provider. Replicas should appear there too, attached to their connector (R9), since the catalog is where Wrangler users already choose sources.

**F3 · Low — Monitoring.** There is no storage or size reporting for managed databases anywhere today (none for Data Manager tables), and no runtime indicator of the Data Manager database connection. The store will be the first subsystem whose size matters to operators, so the overview screen in 5.1 is new ground, not an extension.

**F4 · Low — Designer.** Designer has a data set picker (`DataSetSelectionDialog`) and server metadata through `GuiAuthorityProxy`, but no view that lists server data. With no local store (D22), a developer cannot see what is in a dataset without leaving Designer. See 5.4.

---

## 5. Front end: Data Store

Three audiences: the **administrator** (is it healthy, what is using space, who has access), the **developer** (what is in this dataset, why did my job fail, how do I call it), and the **application developer** using REST.

**Where it lives.** A new embedded app `src/server/data-store` in the Server Console, with a menu entry *Data Store* beside *Data Sets*, guarded by `data_store_use`. Same stack as `data-sets`: Vue, the in-house component library, and the DevExtreme grid with remote paging that the Data Manager editor already uses (`useTGridStore.ts`).

### 5.1 Overview (administrator's landing page)

```
Data Store                                             Store DB: connected · PostgreSQL 16
┌ Storage ───────────────────────────────┐ ┌ Needs attention ─────────────────────────────┐
│ 182 GB of 500 GB licensed   ███████░░░ │ │ ● 2 replicas failed last refresh             │
│ Shared 96 · Replicas 61 · Run (kept) 18 │ │ ● 1 run namespace draining for 9 h (run 48213)│
│ Staged 5 · Scratch 2                   │ │ ● 3 replica requests waiting                 │
└────────────────────────────────────────┘ │ ● Worker on node2: store driver missing      │
┌ Activity now ──────────────────────────┐ └──────────────────────────────────────────────┘
│ 7 active leases · 2 queries · 1 refresh│ ┌ Largest datasets ────────────────────────────┐
│ Run namespaces: 12 active · 4 kept     │ │ crm.sf_account        41 GB   replica        │
│ Maintenance: last 2 min ago, 0 orphans │ │ finance.gl_lines      33 GB   shared         │
└────────────────────────────────────────┘ │ run_48190.stage_orders 9 GB   kept until 14:10│
                                           └──────────────────────────────────────────────┘
```

- **Storage** by namespace kind against the licence limit. This answers S15 at a glance: kept run data is a visible slice.
- **Needs attention** is a single list with links. Each line is also an event (F1), so the same items can send email.
- **Activity now** reads the lease registry (S8): leases and queries with user, job, dataset, duration and a *cancel* action.
- **Connectivity** per cluster node, for the core and the worker separately, including the driver check from §5.3.

### 5.2 Catalog browser (administrator and developer)

Left: a tree of namespaces grouped by kind (*Shared*, *Replicas*, *Scratch*, *Runs with kept data*, *System*). Right: the selected namespace's datasets (name, rows, size, schema version, last write, written by) or one dataset's detail in tabs:

| Tab | Content | For |
|---|---|---|
| **Data** | Server-paged grid, sort and filter, read-only, export to CSV. For a row-policy dataset, a banner says the view is filtered for the current user, and an admin gets *View as user…*. | Developer |
| **Schema** | Fields, types, key and indexes; version history with a diff between versions; *Download as `.fmt`*. | Developer |
| **Access** | Explicit grants and derived grants (D24) shown separately with their origin ("from sandbox `crm_etl`: write"). A box *Check access for user…* that shows the effective level and why. Row policies with a *test as user* preview. Pattern: `DataSetPermissions.vue`. | Admin |
| **Activity** | The per-operation audit: loads, publishes, truncates, schema changes and queries, each with user, run link, row count and duration. | Both |
| **Used by** | Jobs and REST clients that read or wrote the dataset in the last N days, derived from the audit. Nothing like this exists in the product today, and it is what makes dropping or changing a dataset safe. | Both |
| **Storage** | Physical table, retired tables waiting for drop, current pins and leases, index sizes. | Admin |
| **API** | Ready-made `curl`, `StoreReader` and `store_read` snippets for this dataset. | App developer |

Namespace-level actions: create, link to a sandbox, set quota, extend a scratch TTL, drop.

### 5.3 Query console (developer)

A SQL editor for dataset SQL with *Validate* (compile only, shows the result metadata) and *Run* (limited rows, under the user's ACL and row policy). Results in the same grid. *Copy as StoreQuery* and *Copy as REST call*. It runs through the same compiler and limits as `StoreQuery`, so what works here works in a graph.

### 5.4 In the tools developers already use

- **Execution History → *Data* tab** (§7.13), as an embedded Vue element like the Job Inspector tab. Add to what the spec lists: a *Browse* link per dataset into the catalog browser, the quota used, and for a kept namespace the reason it was kept ("member run 48217 ended ERROR"). On a failed run this is the screen people will open first.
- **Designer.**
  - A picker for the dataset attribute on `StoreReader`/`StoreWriter`, modelled on `DataSetSelectionDialog`: namespaces, datasets, and the fields of the selected one.
  - A *Data Store* view for Server projects: browse namespaces, preview data with the existing Data Inspector panel (`cloveretl.gui/.../debugdata/DebugDataPanel`, which already pages lazily and filters), extract metadata to a `.fmt`.
  - For `StoreQuery`: validate against the Server and propagate the result metadata.
  - In the Execution view, a failed run that kept data gets a link *Inspect run data*.
- **Wrangler.** Store datasets as sources and targets in the Data Catalog, under their own heading (D3), with the namespace linked to the workspace offered first (D24).
- **REST.** The store's OpenAPI document in the existing REST API catalog page, and token management for application access once S12 is decided.

---

## 6. Front end: replicas

A replica is an operational object: it has a schedule, a cost at the source, consumers, and a health state. The admin's questions are "is everything fresh", "what failed and why", "who depends on this" and "what will change if I approve, pause or drop it".

### 6.1 Replica board (administrator)

A section *Replicas* inside the Data Store app. One row per replica, with the four state fields from R4 as separate columns:

```
Replicas                         [All] [Failing 2] [Stale 1] [Requests 3] [Unused 4]     + New replica
┌──────────────────┬───────────────────────────┬───────────────┬──────────┬────────┬──────────┬─────────┐
│ Replica          │ Source                    │ Freshness     │ Last run │ Next   │ Rows     │ Used by │
├──────────────────┼───────────────────────────┼───────────────┼──────────┼────────┼──────────┼─────────┤
│ crm.sf_account   │ Salesforce · Account      │ ██░░ 7 h/24 h │ ✓ incr.  │ 01:00  │ 4.1 M    │ 14 jobs │
│ crm.sf_contact   │ Salesforce · Contact      │ ████ 31 h/24 h│ ✗ failed │ retry  │ 9.8 M    │ 9 jobs  │
│                  │                           │   stale       │ 3×       │ 14:20  │          │         │
│ crm.sf_oppty     │ Salesforce · Opportunity  │ █░░░ 2 h/24 h │ ↻ running│ —      │ 1.2 M    │ 3 jobs  │
│ ref.fx_rates     │ ECB rates (job)           │ paused        │ ✓ full   │ —      │ 12 k     │ 0       │
└──────────────────┴───────────────────────────┴───────────────┴──────────┴────────┴──────────┴─────────┘
```

- **Freshness** is a bar of age against the target, turning to *stale* and *expired*. It stays visible while a refresh runs or after one fails.
- **Last run** shows the kind (full or incremental) and the result; a failure shows the retry count and links to the run log.
- **Used by** counts distinct jobs in the last 30 days. Zero is the signal to drop it.
- Filters across the top are the admin's work queues. Bulk actions: refresh, pause, resume.
- A second view, **Source load**, groups replicas by library instance on a 24-hour timeline, showing scheduled refreshes, windows and overlaps. This is where an admin sees that five replicas hit the same Salesforce org at 01:00 (R10).

### 6.2 Requests (administrator)

A queue of `REQUESTED` replicas. Each shows the requester, the connector and parameters, how many users asked for the same thing, and the nearest existing replica if one differs only in its field list (R9). The approval dialog holds the decisions the admin actually has to make:

- schedule, window, freshness target and retention, prefilled from the capability defaults
- the refresh principal
- **first load: now, or at the next window** (R12)
- **applies to: sources that opt in, or all matching sources**, with the list of existing sources and scheduled jobs that would switch (R7)
- who may read, prefilled from the connector's execute groups, with the warning from §8.8 about source-side visibility rules

Reject takes a reason, which is sent to the requester.

### 6.3 Replica detail (administrator, and read-only for readers)

A header with the name, source, the four states, and the actions *Refresh now* (incremental or full), *Pause*, *Roll back*, *Rebuild*, *Delete*. Tabs:

| Tab | Content |
|---|---|
| **History** | One line per refresh: time, kind, duration, rows added, changed and deleted, validation result, link to the run. A chart of row count and duration over time, so a creeping problem is visible before it fails. A failed validation shows what was refused and offers *Inspect staged data* and *Swap anyway* (R14). |
| **Data** | Browse the current snapshot or an older one. *Compare snapshots* gives counts and samples of added, changed and removed rows. |
| **Consumers** | Wrangler sources and jobs that resolve to this replica: mode (prefer or require), their staleness setting, last use, and the snapshot each running job has pinned. Also "near misses": sources on the same connector whose parameters differ, with the difference shown. |
| **Settings** | Schedule, window, freshness, retention and validation, each showing the capability default and whether it is overridden. The estimated storage for the chosen number of snapshots. |
| **Access** | As for datasets, plus the live-derived connector groups (R15). |
| **Snapshots** | The retained snapshots and delta batches with size, age and current pins; *Roll back to this*. For an in-place incremental replica this tab says rollback is not available and why (R1). |

Two confirmation dialogs carry impact information, because these are the moments an admin can cause an outage:

- **Pause, delete or roll back:** "9 jobs read this replica; 2 require it and will fail."
- **Library upgrade** (in the Libraries section): "This version changes the output of `SalesforceObject`. 6 replicas will be rebuilt. Rebuild now in the background, or at the next window?" (R13)

### 6.4 In Wrangler (the user's view)

- **Source header badge**, always shown for a replica-backed source, not only when stale: *Replica · as of today 01:12* with a menu: *Refresh now*, *Read live once*, *Replica settings for this source* (prefer, require, off; maximum staleness). A preview that silently shows yesterday's data is the complaint to prevent.
- **Data Catalog.** A connector that has replicas shows them beneath it as pickable sources with their age (R9). A replicable connector without one offers *Request a replica*, and afterwards shows *Requested by you on 24 Nov · waiting for approval*.
- **Run summary and tracking.** Each source line says *live* or *replica as of …*, so the answer to "which data did this run see" is on the run, as §8.6 intends.
- **Source browsing** (§9.4 step B): the paged grid with server-side sort and filter and an exact row count. Row numbers should come from one rule in both views (R5).

### 6.5 Alerts

The events from F1, wired into the existing *Alerts and Notifications* pattern and the Operations Dashboard: a replica that failed N times, one that will expire within X hours, a refresh that took twice its usual time, a request waiting longer than a day. No new notification system.

---

## 7. Suggested changes to the spec

Grouped by what they touch. Each is small enough for one pull request under the repository's rules.

| Change | Findings | Touches |
|---|---|---|
| Store keeps its own tree record; `run.*` refused for non-persistent roots; orphan rule narrowed | S1, S2, S11, S18 | §7.2, §7.3, §7.6, D21 |
| Private worker pool, restated trust boundary, default lease mode | S3 | §5.3, D4, G5 |
| Active lease registry; one pin scope (per tree) | S8, S9 | §5.3, §6.2, §7.8, §8.6 |
| `publishable` tables; correct the O(1) claim | S7 | §7.4, §7.11 |
| Engine `PropertyKey` additions; data-service flag in the command | S4, S5 | §5.1, §7.7, §7.10, §7.12 |
| Permission model clean-up and reserved names | S10 | §5.6, §6 |
| REST API chapter for applications, with a phase | S12 | §10, §11 |
| Catalog owner identity; backup and restore rule | S13 | §5.4, D5 |
| Portable semantics section | S14 | §5.5, D6 |
| Global budget for kept data; soft quotas | S15, S16 | §7.7, §7.12, §5.8 |
| Operations section: statistics, indexes, pools, audit, export | S17 | new section in §5 |
| Incremental apply mode (copy-and-swap by default) | R1 | §8.4, §8.6, §8.7, D19 |
| `asOf` definition; lookback; mandatory reconcile | R2, R3 | §8.4, §8.5 |
| Four-field state model | R4 | §8.5, §8.9, §9.2 |
| Opt-in substitution and impact lists | R7 | §9.3, D17 |
| Identity: library configuration, version, secure parameters, normalisation; projection subsumption | R8, R9 | §8.3, D9, D15 |
| Refresh precedence, per-library limits, schedule ownership | R10, R11 | §8.4, §5.1 |
| Request workflow details and auto-approve | R12 | §8.2, D16 |
| Compatible schema changes; validation rules | R13, R14 | §8.3, §8.4 |
| Store events | F1 | new section; §5.1 |
| Front end chapter | sections 5 and 6 | new chapter |

**New open questions this review raises** (candidates for `Q20` onwards):

1. Should a restart of a failed job adopt the kept run namespace of the tree it came from? (S18)
2. Is `lease.mode=ALL` the right default, given S3?
3. Copy-and-swap for incremental replicas: default for all sizes, or only below a threshold? (R1)
4. Does an automatic escalation to FULL respect the refresh window? (R10)
5. Does replica substitution apply to existing sources by default, or only to sources that opt in? (R7)
6. How are Wrangler sources re-pointed when a library is upgraded, and what does that mean for replica identity? (R8)
