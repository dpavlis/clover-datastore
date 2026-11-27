# 10. Components and APIs: delta to Base §4–§8


| Element | Purpose | Notes |
|---|---|---|
| `StoreReader` | Structured read of `ns.dataset` (Base §4) | Metadata from the catalog (`MetadataProvider`), freshness options for replicas, output of freshness info as dictionary or graph properties |
| `StoreWriter` | Append, upsert or replace into `ns.dataset` (Base §8) | `createIfMissing` with lifetime, expected schema version, batch and commit size, rejected-record port |
| `StoreQuery` | Dataset SQL (Base §5, [§10.2](#102-sql-scope-d42)): a `SELECT` with output records, or an `INSERT INTO … SELECT` with a row count | Result metadata derived from the compiler at design time through the server |
| `StoreOperation` | Jobflow-style control: CREATE, DROP, TRUNCATE, ALTER, PUBLISH, REFRESH_REPLICA, WAIT_FRESH, plus CREATE_NAMESPACE, EXTEND_TTL and DROP_NAMESPACE for scratch namespaces (D20) | One operation per input record, like the file operation components |
| `storeLookup` | Lookup table over a lookup index | [§9.5](09-wrangler.md#95-lookups) |
| CTL functions (D28) | `boolean storeExists(string name)`, `long storeRowCount(string name)`, `date storeAsOf(string name)` (null unless a replica), `string storeState(string name)` (the replica state, or `ACTIVE`) | Read-only metadata for jobflow conditions, in `cloveretl.ctlfunction.commercial`. They resolve `run.*` within the calling run and need `READ`. Server runs only, like the components. No data access or SQL from CTL: rows are read through components, or through `storeLookup` with CTL's existing `lookup(name).get(key)` syntax. |
| REST | `/api/rest/data-store/v1/namespaces`, `/datasets/{ns}.{name}/schema`, `/records`, `/query`, `/replicas`, `/replicas/{id}`, `/replicas/{id}/refresh`, `/purge`, `/pause`, `/resume` (D41), `/replicas/{id}/snapshots`, `/replicas/{id}/changes?since=N` (P5, D29) | Public, OpenAPI-first like the DM public API |
| MCP | `store_list`, `store_schema`, `store_read`, `store_query`, `store_replica_status`, `store_replica_refresh` | Read tools first. Write tools gated in the Designer like `ToolConfirmation.GATED`. |

## 10.1 REST for applications (D34)

The table above lists routes. An application outside CloverDX needs more than routes: an identity that is not a person, pages that stay consistent, and a safe way to load data. Base §4 and §8 define the read and write operations themselves; this section adds what the API needs around them.

**D34 (proposed). The public REST API is a full client of the store, read and write, with these rules.**

**Identity.** An application calls with an **API token**, not a user's password.

- A token belongs to a Server user, typically a service account, and acts with that user's grants and row policy attributes. It can be narrowed further to named namespaces and to `READ` or `WRITE`.
- Tokens are created and revoked by the user or by a `data_store_admin`, have an expiry, and are shown once. The store keeps a hash.
- Whether the token is a new store object or reuses an existing Server mechanism is open (Q24).
- Every call is recorded in the store audit with the token's id.

**Reading.**

| Rule | Detail |
|---|---|
| Paging | `GET /datasets/{ns}.{name}/records?limit=…` returns a `next` cursor. The cursor is a keyset position (the last key, or `__src_row`), not an offset, so deep pages cost the same as the first. |
| Consistency | The first page pins the dataset's current physical table and returns a **snapshot token** inside the cursor ([§6.2](06-namespaces.md#62-catalog-model)). Later pages read the same table, even after a publish or a refresh. The token expires after `datastore.rest.snapshotTtl` without use, and an expired one gets `410 Gone`, which tells the client to start again. |
| Filter, sort, fields | As Base §4. Sorting by anything but the key needs an index on the dataset, or the request is refused above a row limit. |
| Freshness | For a replica, the response carries `asOf` and the replica's state. The request may state `maxStaleness` and `onStale` ([§8.6](08-replicas.md#86-consumer-contract-and-snapshot-pinning)), except `LIVE`. |
| Schema | `GET …/schema` returns the fields and the schema version, also as JSON Schema. |

**Writing.**

| Rule | Detail |
|---|---|
| Small writes | `POST …/records` with a JSON array or NDJSON: append, or upsert by the dataset's key. Limited to `datastore.rest.maxInlineBytes` per request. One request is one transaction. |
| Idempotency | A write may carry an `Idempotency-Key` header. A repeat of the same key within 24 hours returns the first result and writes nothing, so a client can retry after a timeout without duplicating rows. |
| Schema | A write states the schema version it was built for. A mismatch is refused with the current version (`409 Conflict`). |
| Rejected records | The response lists rejected records with their position and reason, up to a limit. The request chooses whether one rejection fails all. |
| Bulk loads | An **upload session**, the REST form of build-then-publish ([§7.4](07-run-namespaces.md#74-publish-build-in-isolation-swap-atomically)): `POST /datasets/{ns}.{name}/uploads` opens it and returns an id; the client sends any number of chunks (NDJSON or CSV); `POST …/uploads/{id}/commit` publishes with mode `REPLACE`, `APPEND` or `MERGE`; `DELETE` abandons it. Readers see nothing until the commit. |
| Session storage | A session builds a publishable table (D33) owned by the session. An abandoned session is dropped after `datastore.rest.uploadTtl`. Its storage counts against the caller's quota. |

**Long operations.** A query or a commit that may outlast an HTTP request can be sent with `Prefer: respond-async`. The reply is `202 Accepted` with an operation resource (`/operations/{id}`) to poll for state and result. `datastore.query.timeout` applies to both forms.

**Limits and errors.** Each token has a request rate and a daily volume limit, with defaults in configuration. Errors use one JSON shape with a stable code (`DATASET_NOT_FOUND`, `SCHEMA_MISMATCH`, `QUOTA_EXCEEDED`, `REPLICA_EXPIRED`, `SNAPSHOT_GONE`, …), in the manner of the Data Manager public API.

**Data plane.** REST calls run in the core, never through a worker lease. They are meant for application traffic and moderate loads. Loads of tens of millions of rows belong in a graph with `StoreWriter`.

Additions to [§5.7](05-architecture.md#57-configuration-proposed-keys):

```properties
datastore.rest.snapshotTtl=PT15M
datastore.rest.maxInlineBytes=
datastore.rest.uploadTtl=PT24H
datastore.rest.defaultRateLimit=
```

## 10.2 SQL scope (D42)

**D42 (proposed, [issue #3](https://github.com/dpavlis/clover-datastore/issues/3)). Dataset SQL is full SQL over store datasets, not a restricted query form.** A developer who knows SQL on PostgreSQL, Oracle or SQL Server should be able to write the query they would write there. The same language is accepted by `StoreQuery`, REST `/query` and MCP `store_query`.

**What is supported**

| Area | Supported |
|---|---|
| Joins | Inner, left, right and full outer, cross; any number of datasets, across namespaces, including a dataset joined to itself |
| Grouping | `GROUP BY`, `HAVING`, the standard aggregates, `DISTINCT`, `GROUPING SETS` / `ROLLUP` / `CUBE` |
| Sub-selects | Scalar subqueries, `IN` / `EXISTS` / `ANY`, correlated subqueries, derived tables in `FROM`, common table expressions (`WITH`) |
| Set operations | `UNION [ALL]`, `INTERSECT`, `EXCEPT` |
| Window functions | `OVER (PARTITION BY … ORDER BY …)` with ranking and aggregate functions |
| Ordering and limits | `ORDER BY`, `OFFSET` / `FETCH` (and `LIMIT`) |
| Expressions | `CASE`, `CAST`, arithmetic, string, date and numeric functions |
| Writing | `INSERT INTO ns.dataset [(fields)] SELECT …` and `INSERT INTO … VALUES …` |
| Parameters | Named bind parameters, never string substitution |

**How it stays inside the store's rules.** The statement is not sent to the database as written. As Base §5 describes, the core parses and validates it with Calcite against the catalog, then generates SQL in the database's dialect. In that step:

- every `namespace.dataset` is replaced by its pinned physical table ([§6.2](06-namespaces.md#62-catalog-model)), so the statement reads one version of each dataset
- `READ` is checked on every dataset read, and `WRITE` on the target of an `INSERT`
- the row policy of each dataset is placed below its scan, so a join, a subquery or an aggregate can never see a row the caller may not read
- names that are not store datasets are refused. A statement cannot reach catalog tables, other schemas or database functions with side effects.
- `datastore.query.timeout` and `datastore.query.maxRows` apply

**`INSERT INTO … SELECT`** runs entirely inside the database. No row passes through the engine, so it is the fast way to derive one dataset from others.

- The target may be a shared or scratch dataset, or `run.x`. A replica is never a valid target.
- It is one transaction, coordinated through the lease registry like any other write (D32), and it counts against quotas.
- The selected columns must match the target's fields by name, or by position when a field list is given. Types follow the store's type mapping ([§5.5](05-architecture.md#55-type-model)).
- `StoreQuery` then has no output records. It returns the number of rows written, on an optional output port and in the job log.
- Through MCP an `INSERT` is a write tool, gated like the other write tools. Through REST it needs a token with `WRITE` (D34).

**Functions and portability.** The three databases do not have the same functions.

- A **portable set** behaves the same on all three: the functions Calcite translates for each dialect. It is listed in the documentation, and a statement that uses only these runs unchanged on any store database.
- **Functions of the store's own database** are also accepted, provided they are on an allow-list of side-effect-free functions for that dialect. The validator marks such a statement *not portable*, and Designer shows the mark. This gives developers what the database offers without tying every deployment to the smallest common set.
- Whether the allow-list approach, or the portable set alone, is the right rule is open (Q28).

**Not supported in the first release:** `UPDATE`, `DELETE` and `MERGE` statements, and DDL such as `CREATE TABLE … AS SELECT`. Rows are changed through `StoreWriter` (upsert, replace) and datasets are created through `StoreOperation`. Whether these statements should follow is Q27.

---

← [9. Wrangler integration](09-wrangler.md) · [Index](README.md) · [11. Phasing](11-phasing.md) →
