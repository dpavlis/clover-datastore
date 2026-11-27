# 8. Replicas (shadow datasets)


## 8.1 Concept

A **replica** is a store dataset whose content is owned by a **source binding**, not by user writes. The replica manager keeps it within a freshness target by running a refresh job, and records *how fresh* it is. Consumers state *how fresh it must be*.

| Kind | Source reference | Typical example | Section |
|---|---|---|---|
| `CONNECTOR` | Library code, connector path and parameter values (the Wrangler connector: a `.sgrf` with no input and one output) | Salesforce Account object | [§8.2](#82-two-layers-connector-capability-and-replica-instance)–[§8.9](#89-replica-state) |
| `JOB` | Any graph or subgraph with one output port, plus parameters | A custom REST extraction | Same as `CONNECTOR`, with the capability given in the instance |
| `FILE` | URL plus parser settings and metadata. Freshness is a file **fingerprint**, not a clock. | Excel lookup table | [§9.6](09-wrangler.md#96-staged-file-datasets-excel-and-csv-lookups) (*staged datasets*) |
| `QUERY` | Dataset SQL over other store datasets (materialised view) | Dashboard aggregates | Later phase (Base §11) |

## 8.2 Two layers: connector capability and replica instance

Only the connector's author knows how to find what changed at the source, how deletes show up, and which parameters do not affect content. The operator decides how often to refresh, how long to keep data, and who may read it. The spec separates the two.

**Capability**: declared by the connector author in the library's `library.json`. It is an optional `replica` object beside the existing connector entry in `publicInterface.dataConnectors.sources[]`, so it is versioned with the library:

```json
{
  "path": "graph/subgraph/SalesforceObject.sgrf",
  "description": "Reads one Salesforce object",
  "replica": {
    "replicable": true,
    "refreshJob": "graph/subgraph/SalesforceObjectReplica.sgrf",
    "delta": { "key": ["Id"], "maxGap": "P14D" },
    "parameters": {
      "OBJECT": "identity",
      "FIELDS": "projection",
      "WHERE": "identity",
      "BATCH_SIZE": "volatile",
      "TIMEOUT": "volatile"
    },
    "defaults": { "freshnessTarget": "PT24H", "hardExpiry": "P7D", "schedule": "0 0 1 * * ?" },
    "info": { "changeDetection": "SystemModstamp cursor with a 15-minute overlap", "deletes": "queryAll, IsDeleted" }
  }
}
```

| Capability field | Meaning | Required |
|---|---|---|
| `refreshJob` | The paired **replica subgraph** that produces the replica's content ([§8.4](#84-refresh), D40) | No. Without it the connector itself is run and its whole output is loaded. |
| `delta.key` | The fields that identify a row. The store needs them to merge a delta. | Only if the replica subgraph can return a delta |
| `delta.maxGap` | The longest time between two refreshes for which a delta is still trustworthy | No |
| `parameters` | Which parameters are part of the replica's identity ([§8.3](#83-identity-and-parameters)) | No. Undeclared parameters count as identity. |
| `defaults` | Starting values for a new instance | No |
| `info` | Free text for administrators, shown in the Server Console: how changes and deletes are detected. The store does not act on it. | No |

**Instance**: a row in the store catalog. It names the connector and the concrete parameter values, and holds the operational policy:

```yaml
replica: crm.sf_account               # dataset name (namespace.dataset)
source: { kind: CONNECTOR, library: salesforce, connector: graph/subgraph/SalesforceObject.sgrf,
          parameters: { OBJECT: Account, FIELDS: "Id,Name,Industry,OwnerId,SystemModstamp,IsDeleted" } }
refresh:
  # FULL or DELTA is not set here: the store asks, and the replica subgraph answers (D40)
  apply: COPY_AND_SWAP                # COPY_AND_SWAP | IN_PLACE, for DELTA results (D35)
  schedule: "0 0 1 * * ?"             # from capability defaults unless overridden
  fullReconcile: "0 0 2 * * SUN"      # required when the capability has a delta block (D36)
  window: { notBefore: "22:00", notAfter: "05:00" }
  principal: svc_salesforce
  minOnDemandInterval: PT1H           # rate limit for "Refresh now"
freshness: { target: PT24H, hardExpiry: P7D, onStale: USE_WITH_WARNING }
retention: { snapshots: 3, maxSnapshotAge: P30D, dropIfUnusedFor: P90D }
validation: { maxShrink: 20% }        # refuse a FULL swap that loses more than 20 % of rows
substitution: OPT_IN                  # OPT_IN | ALL_MATCHING: which Wrangler sources read it (D37)
```

The two layers combine as follows:

- **A `JOB` source has no `library.json`.** Its instance carries the capability fields itself.
- **Instances are created in the admin UI or through the REST or MCP API**, by holders of `data_store_admin`. Each instance inherits the capability defaults.
- **Wrangler users request, admins approve (D16).** Any Wrangler user who may execute a replicable connector can choose "Keep a replica of this source". That creates an instance in state `REQUESTED`, with the source's parameters and the capability defaults, and notifies data store admins. An admin approves it, possibly adjusting the schedule, window, retention or readers, which moves it to `EMPTY` and schedules the first load. An admin can also reject it with a reason. Until approval, Wrangler keeps reading the source live and shows "replica requested". Requests are deduplicated by replica identity: a second request for the same connector and parameters joins the first.
- **A library upgrade re-validates the instances of its connectors.** `LibraryManager` already notifies Wrangler of library changes. The replica manager listens on the same hook.

**D9 (replaced by D40).** ~~The capability is declared in `library.json`. Instances live in the store catalog. A connector without a `replica` block can still be replicated, but only with `FULL` refresh and with every parameter treated as identity.~~ D40 ([§8.4](#84-refresh)) keeps all of this and changes what the capability holds: it names a replica subgraph, and no longer describes watermark fields for the store to act on.

## 8.3 Identity and parameters

**D38 (proposed).** Replica identity is: library instance + connector path + the normalised **identity** parameters, sorted by name.

- **The library instance is named without its version.** An installed library's sandbox code is `name[_version][~libraryId]`, for example `DataCatalogCoverageLib_0.6`, and that code is what a `.wsrc` stores in `libraryCode`. The identity uses the name and the library id and leaves the version out. Otherwise every upgrade would orphan all replicas of the library. How Wrangler sources are re-pointed after an upgrade is still to be confirmed (Q23).
- **A change to the library's configuration forces a rebuild.** A connector's endpoint and credentials come from the library's `library.prm` and its OAuth2 connection, and an admin can change both in place. Pointing the same library at another Salesforce org changes every row without changing the identity. So when `LibraryManager` reports a configuration change (the notification that already clears Wrangler preview caches), each replica of that library is marked `INCOMPATIBLE`: it is not served, and its next refresh is a FULL rebuild. Two installations of one library, such as production and sandbox, have different library ids and so different replicas.
- **Normalisation** is: trim surrounding whitespace; for a parameter of a list type (`fields`, `sortKey`), split, trim and, for `projection` parameters, sort the items. Values are otherwise compared exactly, including case.
- **Secure parameters** enter the identity as a salted hash of the decrypted value. The catalog stores them encrypted with the server's secure-parameter key, and the admin UI, REST and MCP never return them.
- `volatile` parameters are dropped from the identity. Undeclared parameters count as identity.
- **`projection` parameters are matched by containment, not equality.** A `projection` parameter is a list of output fields (`FIELDS` in the example). A replica serves a source whose identity parameters are equal and whose projection list is a **subset** of the replica's. The reader then selects only the requested columns, in the requested order. The author declares a parameter as `projection` only if each item is an output field name and leaving an item out changes nothing but that column. When several replicas could serve a source, the one with the fewest fields is used. This is not the parameter-to-predicate pushdown deferred below: no row filter is derived.
- `NUM_OF_PREVIEW_RECORDS` is never part of the identity. `LibraryManager` already excludes it from connector parameters.
- **Values must be concrete.** Wrangler applies runtime parameter values to the source before conversion (`WranglerManager.loadWModelAttributes`, called from `translateJobForExecution`). A value can still contain a `${…}` reference that is resolved only when the generated graph runs. The resolver expands such references against the sandbox's parameters before hashing, the same values the generated graph would see. If a reference cannot be resolved at that point (for example it depends on a value known only at graph runtime), the source does not match any replica: it is read live and the run logs a warning naming the parameter (D15). *To verify in P3:* the generated graph links no parameter file today (`MGraph.addParameterFile` has no callers), so which values a `${…}` reference in a connector parameter actually resolves to must be confirmed before the resolver is built.
- **The library version is not part of the identity.** On every use, the connector's current output metadata is compared with the replica's `connectorSchemaHash`. A mismatch marks the replica `INCOMPATIBLE`. It is not served, and the next refresh is a FULL rebuild.
- **Pushdown parameters, which answer a narrower request from a broader replica, are deferred** (former Q8). They need a parameter-to-predicate mapping that only the connector author can define, and identity-only matching is correct, if wasteful.
- **Connectors that use the caller's own credentials are not replicable** in the first release (former Q13, see [§8.8](#88-security)).

## 8.4 Refresh

**D40 (proposed, [issue #2](https://github.com/dpavlis/clover-datastore/issues/2)). A replica is refreshed by a paired replica subgraph that the connector's author writes.** Working out what changed at a source can be hard, and it differs from source to source: a timestamp cursor, a change token, a diff of two extracts, a bulk export. The store does not try to derive it from declared fields. The author implements it in a subgraph, and the store supplies the mechanics around it: scheduling, staging, validation, applying, swapping, snapshots and state.

**The two subgraphs of a replicable connector**

| | Data source connector | Replica subgraph |
|---|---|---|
| Declared in `library.json` | `sources[].path` (exists today) | `sources[].replica.refreshJob` |
| Called | When the source is read live | When a replica of it is refreshed |
| Parameters | The connector's public parameters | The same public parameters, plus the `REPLICA_*` parameters below |
| Output | Port 0: the connector's records | Port 0: records, in the connector's output metadata. Port 1 (optional): the keys of deleted records. |

**Contract of the replica subgraph**

| Element | Direction | Meaning |
|---|---|---|
| `REPLICA_MODE` | in | `FULL` or `DELTA`: what the store asks for. `FULL` on the first load, on `fullReconcile`, after `maxGap`, after a schema or configuration change, and on an admin's "rebuild". Otherwise `DELTA`, if the capability has a `delta` block. |
| `REPLICA_STATE` | in | The opaque state string this subgraph returned from its last successful refresh. Empty on the first load. The store never interprets it: a timestamp, a token or a small JSON document are all fine. |
| `REPLICA_LAST_REFRESH` | in | Start time of the last successful refresh, in UTC |
| Port 0 | out | For `FULL`: every row. For `DELTA`: new and changed rows. |
| Port 1 | out | For `DELTA`: keys of rows to delete, in the metadata of the `delta.key` fields. Not used for `FULL`. |
| `replicaResult` (dictionary) | out | `FULL` or `DELTA`: what the subgraph actually produced. It may answer `FULL` to a `DELTA` request, for example when it finds its state too old. It may not answer `DELTA` to a `FULL` request. |
| `replicaState` (dictionary) | out | The new state, stored only if the refresh commits |

The replica manager generates a **wrapper graph**: `SUBGRAPH(replica subgraph, parameters) → StoreWriter(staged table)`, with port 1 going to a second staged table. The SUBGRAPH is wired the same way `WConnectorSource.addReaderComponent()` wires a connector. The refresh is an ordinary server job:

- execution history, logs, worker placement and the job queue all apply
- its run id is recorded as `refreshRunId`
- the replica subgraph does not write to the store itself. It only emits records, so every refresh passes the same validation and swap (whether it may write directly is Q26).

**A connector without a replica subgraph** can still be replicated. The wrapper then runs the connector itself and treats its output as `FULL`. This costs the author nothing and suits sources small enough to reload each time.

**Applying a `FULL` result (snapshot and swap)**

1. Create a staged physical table.
2. Run the wrapper graph into it. Assign `__src_row` (arrival ordinal) per row.
3. Validate the load:
   - The connector schema hash must match.
   - The row-count change must be within `maxShrink`. This protects against a subgraph that "succeeds" with an empty result because of an expired credential or a wrong filter.
4. Swap the dataset's current-table pointer in one catalog transaction (as in [§7.4](07-run-namespaces.md#74-publish-build-in-isolation-swap-atomically)).
5. Mark the old table `RETIRED` and keep it as a snapshot under the retention rules.

A failed run or a failed validation leaves the previous snapshot current, keeps the old `REPLICA_STATE`, and sets `REFRESH_FAILED`.

**Applying a `DELTA` result (merge by key)**

1. Run the wrapper graph into a staged delta table and a staged deletes table.
2. Apply both by `delta.key`, in the way the instance's `apply` says (D35):
   - `COPY_AND_SWAP` (default): copy the current table inside the database (`CREATE TABLE … AS SELECT`, with its indexes), `MERGE` the delta into the copy and delete the listed keys, then swap the pointer as a FULL refresh does. The previous table becomes a `RETIRED` snapshot.
   - `IN_PLACE`: the same merge and delete on the current table, in one transaction.

   In both, new keys get the next `__src_row` values and updated keys keep theirs.
3. Store `replicaState` only when the swap or the merge commits.
4. Keep the staged delta and deletes tables as **delta batch N** (a numbered snapshot of kind `DELTA`) instead of dropping them, under the snapshot retention rules ([§8.7](#87-retention)). This prepares the change feed (D29). A delta row identical to the stored row is left out of the batch, so the change feed does not report rows that a subgraph re-read only as a safety overlap.

**D35 (proposed). A delta is applied to a copy and swapped in, unless the instance chooses `IN_PLACE`.** Merging into the current table would change it under running readers, so the guarantees of [§8.6](#86-consumer-contract-and-snapshot-pinning) and [§8.7](#87-retention) would hold for FULL replicas only.

| | `COPY_AND_SWAP` | `IN_PLACE` |
|---|---|---|
| A pinned run reads one unchanging snapshot | Yes | No. Each statement is consistent on PostgreSQL and Oracle, and on SQL Server only with read-committed snapshot isolation. A run can see different data in two statements. |
| Roll back to the previous snapshot; diff two snapshots | Yes | No. The admin UI says so. |
| Tracking's `snapshotId` reproduces what the run saw | Yes | No. It identifies the batch the run started on. |
| Cost per refresh | One copy of the table inside the database, plus the merge. No extra source calls. | The merge only |
| Storage | One extra table per retained snapshot | None extra |

`IN_PLACE` is for objects so large that a nightly copy is not acceptable. The threshold at which an admin should choose it is to be measured in P3 (Q21).

**D36 (proposed). What the store guarantees for delta replicas, and what the author must.** The store cannot check that a replica subgraph finds every change. It does three things that do not depend on how the subgraph works:

- **`asOf` is the start time of the last successful refresh** ([§8.5](#85-freshness-metadata)), never a value taken from the data.
- **`fullReconcile` is required.** An instance whose capability has a `delta` block must have a `fullReconcile` schedule; creating one without it is refused. A periodic FULL refresh is the only repair for what a delta misses.
- **`maxGap`** forces a FULL refresh when the last success is too long ago for the source to still report its deletes.

The author's guide for replica subgraphs lists the cases a delta commonly misses, and the reference Salesforce replica subgraph handles the first:

- **Late commits.** A row whose transaction commits after a refresh has read past its timestamp is never seen by a strict "modified after the cursor" query. Re-read an overlap (the reference subgraph goes back 15 minutes from its stored cursor); the merge by key makes the overlap harmless.
- **Rows that leave the filter.** With a parameter that filters rows (`WHERE`), a row changed so that it no longer matches is not returned again, and stays in the replica until the next reconcile.
- **Changes that do not move the cursor**, such as a Salesforce formula field whose inputs changed on another object.

**Change feed (D29, decided for Q12).** Incremental replicas keep no SCD2 history and offer no "as of T" reads. What they do keep, from P3 on, is the delta batches, which hold the changed rows and the tombstones of each refresh. In P5 a reader can ask for **changes since batch N**:

- `StoreReader` gains `changesSince` (the last batch the consumer processed). It returns the rows of all later batches with an operation column (`UPSERT`, `DELETE`), plus the new last batch number for the consumer to store.
- REST adds `/replicas/{id}/changes?since=N`.
- Downstream jobs, for example a load into a warehouse, then process only what changed instead of the whole replica.
- **If batch N+1 has already been dropped by retention**, the call fails with an error telling the consumer to take a full read. Changes are never silently skipped.
- A FULL refresh (a `fullReconcile`, or a switch after `maxGap`) cannot be expressed as a delta. It starts a new batch sequence, and consumers that pass an older batch number get the same full-read error.

Batches cost almost nothing to keep, because the staged delta table already exists. Keeping them from P3 means the P5 feature needs no migration.

**Deletes.** How deletes are found is the replica subgraph's business. The Salesforce components give the reference connector what it needs: `SalesforceReader` and `SalesforceBulkReader` have `readMode=includeDeleted`, which switches to `queryAll` and returns deleted records with `IsDeleted = true`. The replica subgraph routes those to its deletes port.

- **Gap limit.** Salesforce returns deleted records only for a limited time (the recycle bin retention; the exact period needs checking). The capability's `maxGap` captures that limit: if the last successful refresh is older than `maxGap`, the manager asks for `FULL`.
- **Hard deletes.** Records removed from the recycle bin are never returned. `fullReconcile` runs a periodic FULL refresh to catch them.

**Operational rules**

| Rule | Mechanism |
|---|---|
| Single flight | At most one refresh per replica, enforced by a conditional update of its state. A demand that arrives during a refresh joins it. |
| Source protection | `window` (allowed hours) and `datastore.replica.maxConcurrentRefreshes` |
| On-demand refresh | "Refresh now" in Wrangler, the admin UI, REST or MCP. Allowed to holders of `READ` on the replica, limited by `minOnDemandInterval`. A request inside the interval joins the last refresh, or is refused with the time of the next allowed one. |
| First load | The replica is `EMPTY` until the first FULL refresh commits. Consumers in `PREFER` mode read live in the meantime ([§9.3](09-wrangler.md#93-behaviour-per-usage)). |
| Retries | `REFRESH_FAILED` retries with exponential backoff, capped by the next scheduled run |

```mermaid
sequenceDiagram
  autonumber
  participant QZ as Quartz / demand
  participant RM as Replica manager
  participant JE as Job execution (worker)
  participant SF as Salesforce
  participant DB as Store DB

  QZ->>RM: refresh crm.sf_account
  RM->>DB: state → REFRESHING (conditional update, single flight)
  RM->>RM: gap since last success > maxGap? then REPLICA_MODE = FULL else DELTA
  RM->>DB: create staged table DS_007001
  RM->>JE: run wrapper graph (replica subgraph) as svc_salesforce, REPLICA_STATE = stored state
  JE->>SF: queryAll (includeDeleted)
  JE->>DB: StoreWriter → DS_007001 (bulk)
  JE-->>RM: finished OK (replicaResult, replicaState, rows)
  RM->>DB: validate schema hash and row delta
  RM->>DB: copy current, MERGE delta into the copy, swap pointer (FULL: swap only, IN_PLACE: MERGE into current)
  RM->>DB: snapshot row (asOf, replicaState, rows), state → FRESH
  RM->>RM: invalidate Wrangler preview caches for this connector and parameters
```

## 8.5 Freshness metadata

Stored per snapshot. Exposed through REST, MCP and the admin UI, in the Wrangler source badge, and in job tracking.

| Field | Meaning |
|---|---|
| `snapshotId` | Physical version that a consumer pins ([§8.6](#86-consumer-contract-and-snapshot-pinning)) |
| `asOf` | The time at which the replica was last known to match the source: the **start time of the last successful refresh**, for FULL and DELTA refreshes alike (D36). It is never taken from the data. An object that nobody changed for a week is still current, so measuring staleness from its newest row would expire it. |
| `loadedAt` | When the load committed |
| `refreshRunId` | Run of the refresh job, linking to its execution history and log |
| `rows`, `bytes` | Size |
| `replicaState` | The opaque state the replica subgraph returned (D40). It is passed back on the next refresh and never used to judge freshness. |
| `batchNo` | Number of the delta batch for a delta refresh (D29) |
| `connectorSchemaHash`, `libraryVersion` | Detect drift after a library upgrade |
| `state` | `REQUESTED`, `EMPTY`, `FRESH`, `STALE`, `EXPIRED`, `REFRESHING`, `REFRESH_FAILED`, `INCOMPATIBLE`, `DORMANT` |
| `lastAccessAt`, `accessCount` | Feeds `dropIfUnusedFor` and usage reporting |

## 8.6 Consumer contract and snapshot pinning

Each consumer (StoreReader, Wrangler source, REST read, SQL) may state `maxStaleness` and `onStale`. If it does not, the replica's defaults apply.

| `onStale` | Behaviour when `now − asOf > maxStaleness` |
|---|---|
| `FAIL` | Error naming the replica, `asOf` and the policy |
| `USE_WITH_WARNING` | Read the current snapshot and log a warning with `asOf` (job log, Wrangler badge) |
| `REFRESH_AND_WAIT` | Trigger or join a refresh, wait up to `waitTimeout`, then fall back to `USE_WITH_WARNING` |
| `LIVE` | Bypass the replica and run the source directly. Wrangler only, never inside SQL. |

`hardExpiry` overrides `USE_WITH_WARNING`: an expired replica is never served, whether silently or with a warning.

**Pinning.** A consumer resolves the replica to a `snapshotId` once, at the scope given in [§6.2](06-namespaces.md#62-catalog-model) (D32):

- a job: once per **tree**, at the first resolution by any member (for a Wrangler job, at conversion). The pin is stored with the tree's namespace record.
- a REST or MCP read: once per request, carried across pages by a snapshot token
- a SQL statement outside a tree: when it is compiled
- a tree whose root has no run record (D30) has no namespace record to hold a pin. Each of its statements resolves the pointer on its own.

The whole tree then reads that snapshot, even if a refresh swaps the pointer meanwhile. The run's tracking records the snapshot, which makes a run reproducible ("which data did this job see?"). Retention never drops a snapshot that a tree has pinned or a lease is reading. Pins are released when the tree's namespace is released.

This holds for FULL replicas and for delta replicas applied with `COPY_AND_SWAP`. An `IN_PLACE` replica has one table that every refresh changes, so a pin there fixes only which table is read, not its content (D35).

## 8.7 Retention

Four independent rules, all enforced by `datastore_maintenance`:

1. **Snapshots.** Keep the last *N* snapshots and none older than *T* (pinned ones excepted). This supports rollback ("re-point to the previous snapshot") and diffs between refreshes. Delta batches (D29) are not counted against *N*. They are kept up to age *T*, so the change feed reaches back as far as the snapshots do. An `IN_PLACE` delta replica (D35) has snapshots only from its FULL refreshes, so rollback returns it to the last reconcile.
2. **Rows.** Delete rows older than a field-based age, like DM `retention_days`.
3. **Unused replicas.** After `dropIfUnusedFor` without access, drop the data (state `DORMANT`) and keep the definition. The next use triggers a refresh.
4. **Compliance purge.** Remove a key, such as a person id, from the current table, all retained snapshots and all delta batches, and record it in the store audit.

## 8.8 Security

Who may read a replica follows from who could already get the same data live:

- **Default `READ` grant = the connector's execute groups.** Library connectors already carry execute and read groups (`S/library/DataConnector`). A connector that runs on the library's own credentials, such as a Salesforce integration user, already shows everyone who may execute it the same rows the replica holds. Defaulting the replica ACL to those groups exposes nothing new.
- **Wider grants are explicit.** The admin UI warns that replica readers bypass the source system's own visibility rules, such as Salesforce sharing rules.
- **Row policy narrows** where needed, for example `OwnerId IN user.attributes.sfUserIds`.
- **Connectors that use the caller's own credentials** (a per-user OAuth token, for example) make the source output user-specific. They are **not replicable** in the first release. Per-principal replicas (one private copy per user) are a possible later extension.
- **Refresh principal.** The replica runs as a named user, typically a service account, with execute permission on the connector and `MANAGE` on the replica. Credentials come from the library's parameters or secrets, as for any job.

## 8.9 Replica state

```mermaid
stateDiagram-v2
  [*] --> REQUESTED: requested from Wrangler
  REQUESTED --> EMPTY: approved by admin
  REQUESTED --> [*]: rejected
  [*] --> EMPTY: created by admin
  EMPTY --> REFRESHING: schedule, demand or manual
  REFRESHING --> FRESH: commit OK
  REFRESHING --> REFRESH_FAILED: job or validation failed
  REFRESH_FAILED --> REFRESHING: retry with backoff
  FRESH --> STALE: age over target
  STALE --> EXPIRED: age over hardExpiry
  STALE --> REFRESHING
  EXPIRED --> REFRESHING
  FRESH --> REFRESHING: scheduled refresh
  FRESH --> INCOMPATIBLE: connector schema changed
  STALE --> INCOMPATIBLE: connector schema changed
  INCOMPATIBLE --> REFRESHING: forced FULL rebuild
  FRESH --> DORMANT: unused for dropIfUnusedFor
  DORMANT --> REFRESHING: next demand
```

In `REFRESH_FAILED` the previous snapshot is still served. A failed refresh makes the replica staler. It does not make it unavailable.

## 8.10 Management in the Server Console (D41)

**D41 (proposed, [issue #2](https://github.com/dpavlis/clover-datastore/issues/2)). The Server Console gets a Replicas page.** It is the one place where an administrator sees which replicas are configured, which of them hold data, and how healthy each is.

**Where.** A *Replicas* entry in the Server Console menu, in the Data Store section, shown to holders of `data_store_admin`. It is part of the embedded `data-store` web app ([§5.1](05-architecture.md#51-placement-in-the-codebase)), built like the existing *Data Sets* page. Holders of `READ` on a replica see that replica read-only, with *Refresh now* only.

**Overview: one row per configured replica**

| Column | Content |
|---|---|
| Replica | Dataset name, and whether it is materialised (holds data) or only configured (`REQUESTED`, `EMPTY`, `DORMANT`) |
| Data source | Library, connector and the identity parameters. Secure values are never shown. |
| Last updated | `asOf` and `loadedAt` of the current snapshot, with the kind of the last refresh (FULL or DELTA) |
| Time until obsolete | Time left to the freshness target, then to `hardExpiry`, shown as a bar: fresh, stale, expired |
| Last refresh | Result and duration, with a link to the run in the Execution History. For a failure: the reason and the retry count. |
| Next refresh | The next scheduled time, or *paused* |
| Rows and size | Of the current snapshot, plus the storage held by retained snapshots |
| Used by | Distinct jobs that read it in the last 30 days, from `accessCount` |

Filters select the replicas that need attention: failing, stale or expired, requested, unused.

**Actions**

| Action | Effect | Who |
|---|---|---|
| Refresh now | Starts a refresh, or joins a running one. The admin chooses *as scheduled* or *full rebuild* (`REPLICA_MODE = FULL`). | Admin. A reader gets *as scheduled* only, within `minOnDemandInterval`. |
| Purge data | Drops the current table, all snapshots and delta batches, and clears `REPLICA_STATE`. The definition stays and the replica becomes `DORMANT`, so the next demand or schedule reloads it in full. The dialog lists the jobs that read the replica and says which of them require it. | Admin |
| Pause / resume | Stops scheduled and on-demand refreshes. The data stays served under the freshness rules. | Admin |
| Roll back | Re-points to an earlier snapshot ([§8.7](#87-retention)). Not offered for an `IN_PLACE` replica (D35). | Admin |
| Edit | Schedule, window, freshness, retention, validation, substitution (D37) and readers | Admin |
| Delete | Purges the data and removes the definition | Admin |
| Approve / reject | For a `REQUESTED` replica (D16) | Admin |

**Detail of one replica.** The refresh history (time, kind, rows added, changed and deleted, duration, validation result, run link), the retained snapshots with their sizes and pins, the consumers, and the capability's `info` text, which tells the admin how this connector detects changes.

The same data and actions are in REST ([§10](10-components-and-apis.md#10-components-and-apis-delta-to-base-48)): `GET /replicas` returns the overview rows, and `POST /replicas/{id}/refresh`, `/purge`, `/pause` and `/resume` are the actions. The page uses only that API.

---

← [7. Run-scoped namespaces](07-run-namespaces.md) · [Index](README.md) · [9. Wrangler integration](09-wrangler.md) →
