# 10. Components and APIs: delta to Base §4–§8


| Element | Purpose | Notes |
|---|---|---|
| `StoreReader` | Structured read of `ns.dataset` (Base §4) | Metadata from the catalog (`MetadataProvider`), freshness options for replicas, output of freshness info as dictionary or graph properties |
| `StoreWriter` | Append, upsert or replace into `ns.dataset` (Base §8) | `createIfMissing` with lifetime, expected schema version, batch and commit size, rejected-record port |
| `StoreQuery` | Dataset SQL (Base §5) | Result metadata derived from the compiler at design time through the server |
| `StoreOperation` | Jobflow-style control: CREATE, DROP, TRUNCATE, ALTER, PUBLISH, REFRESH_REPLICA, WAIT_FRESH, plus CREATE_NAMESPACE, EXTEND_TTL and DROP_NAMESPACE for scratch namespaces (D20) | One operation per input record, like the file operation components |
| `storeLookup` | Lookup table over a lookup index | [§9.5](09-wrangler.md#95-lookups) |
| CTL functions (D28) | `boolean storeExists(string name)`, `long storeRowCount(string name)`, `date storeAsOf(string name)` (null unless a replica), `string storeState(string name)` (the replica state, or `ACTIVE`) | Read-only metadata for jobflow conditions, in `cloveretl.ctlfunction.commercial`. They resolve `run.*` within the calling run and need `READ`. Server runs only, like the components. No data access or SQL from CTL: rows are read through components, or through `storeLookup` with CTL's existing `lookup(name).get(key)` syntax. |
| REST | `/api/rest/data-store/v1/namespaces`, `/datasets/{ns}.{name}/schema`, `/records`, `/query`, `/replicas/{id}`, `/replicas/{id}/refresh`, `/replicas/{id}/snapshots`, `/replicas/{id}/changes?since=N` (P5, D29) | Public, OpenAPI-first like the DM public API |
| MCP | `store_list`, `store_schema`, `store_read`, `store_query`, `store_replica_status`, `store_replica_refresh` | Read tools first. Write tools gated in the Designer like `ToolConfirmation.GATED`. |

---

← [9. Wrangler integration](09-wrangler.md) · [Index](README.md) · [11. Phasing](11-phasing.md) →
