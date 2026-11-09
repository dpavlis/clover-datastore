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

Only the connector's author knows which parameter carries a "modified since" value, which output field is the change watermark, how deletes show up, and which parameters do not affect content. The operator decides how often to refresh, how long to keep data, and who may read it. The spec separates the two.

**Capability**: declared by the connector author in the library's `library.json`. It is an optional `replica` object beside the existing connector entry in `publicInterface.dataConnectors.sources[]`, so it is versioned with the library:

```json
{
  "path": "graph/subgraph/SalesforceObject.sgrf",
  "description": "Reads one Salesforce object",
  "replica": {
    "replicable": true,
    "key": ["Id"],
    "incremental": {
      "watermarkField": "SystemModstamp",
      "watermarkParameter": "MODIFIED_SINCE",
      "deletes": { "mode": "TOMBSTONE", "flagField": "IsDeleted", "maxGap": "P14D" }
    },
    "parameters": {
      "OBJECT": "identity",
      "FIELDS": "identity",
      "WHERE": "identity",
      "BATCH_SIZE": "volatile",
      "TIMEOUT": "volatile"
    },
    "defaults": { "freshnessTarget": "PT24H", "hardExpiry": "P7D", "schedule": "0 0 1 * * ?" }
  }
}
```

**Instance**: a row in the store catalog. It names the connector and the concrete parameter values, and holds the operational policy:

```yaml
replica: crm.sf_account               # dataset name (namespace.dataset)
source: { kind: CONNECTOR, library: salesforce, connector: graph/subgraph/SalesforceObject.sgrf,
          parameters: { OBJECT: Account, FIELDS: "Id,Name,Industry,OwnerId,SystemModstamp,IsDeleted" } }
refresh:
  mode: INCREMENTAL                   # FULL | INCREMENTAL (INCREMENTAL needs the capability)
  schedule: "0 0 1 * * ?"             # from capability defaults unless overridden
  fullReconcile: "0 0 2 * * SUN"
  window: { notBefore: "22:00", notAfter: "05:00" }
  principal: svc_salesforce
  minOnDemandInterval: PT1H           # rate limit for "Refresh now"
freshness: { target: PT24H, hardExpiry: P7D, onStale: USE_WITH_WARNING }
retention: { snapshots: 3, maxSnapshotAge: P30D, dropIfUnusedFor: P90D }
validation: { maxShrink: 20% }        # refuse a FULL swap that loses more than 20 % of rows
```

The two layers combine as follows:

- **A `JOB` source has no `library.json`.** Its instance carries the capability fields itself.
- **Instances are created in the admin UI or through the REST or MCP API**, by holders of `data_store_admin`. Each instance inherits the capability defaults.
- **Wrangler users request, admins approve (D16).** Any Wrangler user who may execute a replicable connector can choose "Keep a replica of this source". That creates an instance in state `REQUESTED`, with the source's parameters and the capability defaults, and notifies data store admins. An admin approves it, possibly adjusting the schedule, window, retention or readers, which moves it to `EMPTY` and schedules the first load. An admin can also reject it with a reason. Until approval, Wrangler keeps reading the source live and shows "replica requested". Requests are deduplicated by replica identity: a second request for the same connector and parameters joins the first.
- **A library upgrade re-validates the instances of its connectors.** `LibraryManager` already notifies Wrangler of library changes. The replica manager listens on the same hook.

**D9.** The capability is declared in `library.json`. Instances live in the store catalog. A connector without a `replica` block can still be replicated, but only with `FULL` refresh and with every parameter treated as identity.

## 8.3 Identity and parameters

Replica identity is: library code + connector path + the normalised **identity** parameters, sorted by name.

- `volatile` parameters are dropped from the identity. Undeclared parameters count as identity.
- `NUM_OF_PREVIEW_RECORDS` is never part of the identity. `LibraryManager` already excludes it from connector parameters.
- **Values must be concrete.** Wrangler applies runtime parameter values to the source before conversion (`WranglerManager.loadWModelAttributes`, called from `translateJobForExecution`). A value can still contain a `${…}` reference that is resolved only when the generated graph runs. The resolver expands such references against the sandbox's parameters before hashing, the same values the generated graph would see. If a reference cannot be resolved at that point (for example it depends on a value known only at graph runtime), the source does not match any replica: it is read live and the run logs a warning naming the parameter (D15).
- **The library version is not part of the identity.** On every use, the connector's current output metadata is compared with the replica's `connectorSchemaHash`. A mismatch marks the replica `INCOMPATIBLE`. It is not served, and the next refresh is a FULL rebuild.
- **Pushdown parameters, which answer a narrower request from a broader replica, are deferred** (former Q8). They need a parameter-to-predicate mapping that only the connector author can define, and identity-only matching is correct, if wasteful.
- **Connectors that use the caller's own credentials are not replicable** in the first release (former Q13, see [§8.8](#88-security)).

## 8.4 Refresh

The replica manager generates a **wrapper graph**: `SUBGRAPH(connector, parameters [+ watermark]) → StoreWriter(staged table)`. The SUBGRAPH is wired the same way `WConnectorSource.addReaderComponent()` wires it for Wrangler, so the connector runs exactly as it does in a Wrangler job. The refresh is an ordinary server job:

- execution history, logs, worker placement and the job queue all apply
- its run id is recorded as `refreshRunId`

**FULL (snapshot and swap)**

1. Create a staged physical table.
2. Run the wrapper graph into it. Assign `__src_row` (arrival ordinal) per row.
3. Validate the load:
   - The connector schema hash must match.
   - The row-count change must be within `maxShrink`. This protects against a connector that "succeeds" with an empty result because of an expired credential or a wrong filter.
4. Swap the dataset's current-table pointer in one catalog transaction (as in [§7.4](07-run-namespaces.md#74-publish-build-in-isolation-swap-atomically)).
5. Mark the old table `RETIRED` and keep it as a snapshot under the retention rules.

A failed run or a failed validation leaves the previous snapshot current and sets `REFRESH_FAILED`.

**INCREMENTAL (watermark merge)**

1. Read the stored watermark: the maximum `watermarkField` of the last successful refresh.
2. Run the wrapper graph with `watermarkParameter` set to that watermark, into a staged delta table.
3. `MERGE` the delta into the current table by key, in one transaction:
   - Rows flagged as deleted (`deletes.mode = TOMBSTONE`) become deletes.
   - New keys get the next `__src_row` values. Updated keys keep theirs.
4. Advance the watermark only when the merge commits.
5. Keep the staged delta table as **delta batch N** (a numbered snapshot of kind `DELTA`) instead of dropping it, under the snapshot retention rules ([§8.7](#87-retention)). This prepares the change feed (D29).

**Change feed (D29, decided for Q12).** Incremental replicas keep no SCD2 history and offer no "as of T" reads. What they do keep, from P3 on, is the delta batches, which hold the changed rows and the tombstones of each refresh. In P5 a reader can ask for **changes since batch N**:

- `StoreReader` gains `changesSince` (the last batch the consumer processed). It returns the rows of all later batches with an operation column (`UPSERT`, `DELETE`), plus the new last batch number for the consumer to store.
- REST adds `/replicas/{id}/changes?since=N`.
- Downstream jobs, for example a load into a warehouse, then process only what changed instead of the whole replica.
- **If batch N+1 has already been dropped by retention**, the call fails with an error telling the consumer to take a full read. Changes are never silently skipped.
- A FULL refresh (a `fullReconcile`, or a switch after `maxGap`) cannot be expressed as a delta. It starts a new batch sequence, and consumers that pass an older batch number get the same full-read error.

Batches cost almost nothing to keep, because the staged delta table already exists. Keeping them from P3 means the P5 feature needs no migration.

**Deletes.** The Salesforce components already support this: `SalesforceReader` and `SalesforceBulkReader` both have `includeDeleted`, which switches to `queryAll` and returns deleted records with `IsDeleted = true`.

- **Gap limit.** Salesforce returns deleted records only for a limited time (the recycle bin retention; the exact period needs checking). `maxGap` captures that limit: if the last successful incremental refresh is older than `maxGap`, the manager automatically runs a FULL refresh instead.
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
  RM->>RM: gap since last success > maxGap? then FULL else INCREMENTAL
  RM->>DB: create staged table DS_007001
  RM->>JE: run wrapper graph as svc_salesforce, MODIFIED_SINCE = watermark
  JE->>SF: queryAll (includeDeleted)
  JE->>DB: StoreWriter → DS_007001 (bulk)
  JE-->>RM: finished OK (rows, max SystemModstamp)
  RM->>DB: validate schema hash and row delta
  RM->>DB: MERGE delta into current (or swap pointer for FULL)
  RM->>DB: snapshot row (asOf, watermark, rows), state → FRESH
  RM->>RM: invalidate Wrangler preview caches for this connector and parameters
```

## 8.5 Freshness metadata

Stored per snapshot. Exposed through REST, MCP and the admin UI, in the Wrangler source badge, and in job tracking.

| Field | Meaning |
|---|---|
| `snapshotId` | Physical version that a consumer pins ([§8.6](#86-consumer-contract-and-snapshot-pinning)) |
| `asOf` | Source time the snapshot represents: refresh start for FULL, the watermark for INCREMENTAL |
| `loadedAt` | When the load committed |
| `refreshRunId` | Run of the refresh job, linking to its execution history and log |
| `rows`, `bytes` | Size |
| `watermark` | Incremental cursor |
| `batchNo` | Number of the delta batch for an incremental refresh (D29) |
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

**Pinning.** A consumer resolves the replica to a `snapshotId` once:

- a job, when it is converted or starts
- a SQL statement, when it is compiled

The whole run then reads that snapshot, even if a refresh swaps the pointer meanwhile. The run's tracking records the snapshot, which makes a run reproducible ("which data did this job see?"). Retention never drops a snapshot that a running job has pinned. Pins are released at run end, and the maintenance sweeper clears pins whose run is final.

## 8.7 Retention

Four independent rules, all enforced by `datastore_maintenance`:

1. **Snapshots.** Keep the last *N* snapshots and none older than *T* (pinned ones excepted). This supports rollback ("re-point to the previous snapshot") and diffs between refreshes. Delta batches (D29) are not counted against *N*. They are kept up to age *T*, so the change feed reaches back as far as the snapshots do.
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

---

← [7. Run-scoped namespaces](07-run-namespaces.md) · [Index](README.md) · [9. Wrangler integration](09-wrangler.md) →
