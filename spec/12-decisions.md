# 12. Decision register


| # | Decision | Content | Status |
|---|---|---|---|
| D1 | Relation to Data Manager | Separate subsystem. `SchemaHandler` is split into a shared `ManagedSqlDialect`, exception translators move, the patch runner base is reused. DM behaviour unchanged. | decided |
| D2 | Store database | Own datasource config. May reuse the DM database (`type=datamanager`), only when DM is local. Own prefix and patch table. | decided |
| D3 | Naming (Q1) | Feature "Data Store", objects "datasets". Always qualified where DM data sets also appear. Machine names `store`/`datastore`, never `dataset`. | decided |
| D4 | Data plane | Direct lease from the start: core-compiled statements executed on a worker pool with a DML-only account. Row-policy datasets always proxied. `datastore.lease.mode` can restrict leases. | decided |
| D5 | Catalog location | In the store DB, next to the tables | decided |
| D6 | Type model | Scalar CloverDX types from P1, list/map/variant as JSON from P5. Versioned schemas, mapping owned by the storage foundation. | decided |
| D7 | Designer local runs | As the Data Manager: metadata from the Server at design time for Server projects, data only in Server runs. Run datasets take explicit (shared `.fmt`) metadata. | decided |
| D8 | Wrangler lookups | Staged file datasets plus `storeLookup` first, SQL join pushdown for lookups later (P5), all behind a parity suite (detailed by D13) | decided |
| D9 | Replica definitions | Capability in `library.json` (key, watermark, deletes, parameter classes, defaults). Instances in the store catalog. | decided |
| D10 | Wrangler hook | `SourceReplicaProvider` in `ConversionContext`, resolved on the Server once per conversion, snapshot pinned for the whole run | decided |
| D11 | Replica content | The connector's native output plus `__src_row`. Wrangler's converter and row filler stay in the graph. | decided |
| D12 | Replica access | Default `READ` = the connector's execute groups. Wider grants explicit. Connectors using per-user credentials not replicable at first. | decided |
| D13 | Large lookups | Lookup index (keys computed in the engine, first row per key by `__src_row`) plus `storeLookup` above a size threshold, in memory below it | decided |
| D14 | Wrangler browsing | Replica-fed previews and graph-free source browsing | decided |
| D15 | `${…}` in connector parameters (Q14) | Resolve against the sandbox's parameters before matching. If unresolvable, read live and warn. | decided |
| D16 | Replica creation from Wrangler (Q15) | Request from Wrangler, approval by a data store admin (`REQUESTED` state). Direct creation only by admins. | decided |
| D17 | Stale replica in unattended Wrangler runs (Q16) | `USE_WITH_WARNING` by default. Stricter behaviour is opted into per replica or per `.wsrc`. | decided |
| D18 | Staged file datasets (Q17) | All lookup sources, plus main sources above a size threshold. Main sources are staged only during full runs. | decided |
| D19 | Incremental refresh (Q18) | Ships with replicas in P3, together with FULL refresh, `maxGap` escalation and `fullReconcile`. | decided |
| D20 | Sharing outside the tree (Q5) | Scratch namespaces (TTL + ACL) now. Join token later, prepared by the `NAMESPACE_MEMBER` relation. | decided |
| D21 | Tree outcome (Q6) | Failed if any member failed. A failed tree keeps `TEMPORARY` data for `retainOnFailure`. Plus `keepOnFailure` per dataset and `datastore.run.keepNamespace` per job. | decided |
| D22 | Local store (Q7) | None. Store graphs run on the Server only. | decided |
| D23 | Data services (Q19) | `run.*` refused in `RESTJOB` trees. Use shared or scratch namespaces. | decided |
| D24 | Namespaces and sandboxes (Q3) | Own ACL plus an optional link to one sandbox or workspace that adds live-derived grants (never `ADMIN`). Names stay global. | decided |
| D25 | Licensing (Q2) | `dataStore` module flag, plus optional `maxDataStoreReplicas` and `maxDataStoreBytes` limits, enforced only when present | decided |
| D26 | Dates (Q4) | UTC in `TIMESTAMP(3)` / `DATETIME2(3)`, written and read with an explicit UTC calendar. Field `timeZone` kept for formatting. | decided |
| D27 | Wrangler step pushdown (Q10) | None. Step previews always run the recipe graph. | decided |
| D28 | CTL functions (Q11) | Read-only metadata functions: `storeExists`, `storeRowCount`, `storeAsOf`, `storeState`. No data or SQL from CTL. | decided |
| D29 | Incremental replica history (Q12) | No SCD2 or as-of reads. Delta batches kept from P3 under snapshot retention. A "changes since batch N" feed in P5, failing loudly when batches are gone. | decided |
| D31 | Worker pool (amends D4) | The store's worker pool is private to the lease executor and not bound in JNDI. With leases on, object ACL is advisory against user Java code in the worker; row policy stays a hard boundary. The default of `datastore.lease.mode` is open (Q20). | proposed |

---

← [11. Phasing](11-phasing.md) · [Index](README.md) · [13. Open questions](13-questions.md) →
