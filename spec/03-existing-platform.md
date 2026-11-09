# 3. What exists today


Verified against the code on branch `CLO-37527-twin-null-handling`.

| Area | What exists | Where |
|---|---|---|
| Managed data DB | Data Manager uses its **own** datasource (`datamanager.datasource.type` = JDBC / JNDI / remote / empty), separate from the system DB. Data is supported only on PostgreSQL, SQL Server and Oracle. | `S/datamanager/DataManagerConfiguration`, `S/datasource/DataManagerDataSource`, `web/WEB-INF/applicationContext-dataManagerPersistence.xml`, `web/WEB-INF/config.properties` |
| Per-dialect DDL | `SchemaHandler` with Postgre, Mssql and Oracle subclasses. It covers create and drop tables, add and remove columns and indexes, truncate, row limit, quoting and temp tables. `LayoutDiff` handles schema evolution, and each dialect has an exception translator. | `S/datamanager/schema/*` |
| Schema patches | Separate patch stream for the DM database | `S/dbschema/DataManagerDbSchemaUpdate`, `web/WEB-INF/dbpatches/data_manager_db/` |
| DM model | Data set metadata is stored in the system DB (`data_set`, SINGLE_TABLE inheritance with types TRANSACTIONAL and REFERENCE). Data lives in the DM DB, one table per data set (`<code>_data`, `_audit`, `_change`). | `S/persistent/DataSet.java`, `S/datamanager/LocalDataManagerPersistence` (≈4.5k LOC) |
| DM types | BOOLEAN, DATE, DECIMAL(32,10), LONG, STRING, URL | `S/datamanager/DataSetLayout.ColumnMetadataType` |
| DM security | Global `Operation` tree (`data_sets_*`, `data_manager`). Per data set ACL READ_ONLY < EDITOR < APPROVER < ADMIN. **No row-level security.** | `S/facade/api/misc/Operation`, `S/persistent/DataSetPermission*` |
| Graph ↔ server data | DM readers and writers call `IAuthorityProxy` dataset methods. On a worker these go over gRPC to the core, which runs JDBC. | `E/graph/runtime/IAuthorityProxy`, `S/graph/runtime/ServerAuthorityProxy`, `S/worker/ServerAuthorityProxyApiImpl` |
| Design-time metadata | DM readers implement `MetadataProvider`, and `dataSetCode` propagates metadata on update. | `cloveretl.component.commercial/plugin.xml`, `AbstractDataSetReader` |
| SQL libraries | Calcite core/file/linq4j 1.41.0, Avatica, JSqlParser 5.3 | `cloveretl.server/build.gradle`, `version.properties`. Used today in `S/mcp/tool/utils/PerformanceLogsManager` and the MCP DB query validator. |
| Run tree | `run_record.root_run_id` and `parent_run_id` are indexed. `ROOT_RUN_ID` and `PARENT_RUN_ID` are automatic graph parameters in every descendant. `RunRecordFilter.setRootRunId` queries a whole tree. | `S/persistent/RunRecord`, `cloveretl.commons/.../execution/AdditionalJobProperties`, `ExecutionHelper` |
| Job end hooks | `TransformationLifeCycleStrategy.doReleaseEnvironment` (per run). Events `JobFinished`, `JobError`, `JobAborted`, `JobUnknown`. Crash recovery through `solveUnresolvedGraphs` (→ UNKNOWN). **No "tree finished" signal.** Daemon children can outlive the root. | `S/job/graph/`, `S/events/`, `S/graph/LocalExecutionManager` |
| Temp space | One directory per run per node. Not shared by a tree or across nodes. | `S/tempmgr/TempFileManager` |
| Scheduling | Quartz schedules running `Task.TaskType`, including `data_manager_maintenance` → `TaskProcessorDataManager` (retention purge) | `S/persistent/Task`, `S/tasks/TaskProcessorDataManager` |
| Wrangler sources | `.wsrc`: file, Excel, connector (a library `.sgrf` subgraph with one output) or DM data set. Converted to graph components by `W*Source.convert*`. | `E/wrangler/job/model/source/*`, `S/library/LibraryManager` |
| Wrangler cache | Preview only. Binary files in `<workspace>/.cache/` keyed by source id + **parameter hash** + record limit. Max age `wrangler.sourceCache.maxAge=3600`, swept by `GarbageManJob`. Full runs never use it. | `E/wrangler/SourceCacheProvider`, `S/wrangler/WorkspaceCacheProvider`, `S/timing/GarbageManJob` |
| Wrangler lookup | The `WLookup` step compiles to `EXT_HASH_JOIN` (left outer). The whole lookup side is held in memory, with no spill. | `E/wrangler/job/model/step/WLookup`, `cloveretl.component/.../HashJoin` |
| Wrangler preview | Synchronous hidden graph run, read back through edge debug data. `numRecords` only, with no offset or cursor. The UI virtualises a fully loaded set (1k–100k rows). | `S/wrangler/WranglerManager`, `S/facade/api/resp/WranglerDataset` |
| Lookup tables | `simpleLookup`, `dbLookup` (with `maxCached`), `rangeLookup`, `persistentLookup` (JDBM), `referenceLookup` (DM) | `cloveretl.lookup*`, `ReferenceLookupTable` |
| JDBC dialects (engine) | `JdbcSpecific`: type mapping, DDL type strings, quoting. No pagination, MERGE or full CREATE TABLE. PostgreSQL inherits the default type mapping. | `E/database/sql/JdbcSpecific`, `cloveretl.jdbc/.../specific/impl/*` |
| Bulk load | Only external tools (`psql \copy`, `bcp`, `sqlldr`). No JDBC bulk APIs (`CopyManager`, `SQLServerBulkCopy`) anywhere. | `cloveretl.bulkloader` |

---

← [2. Goals and non-goals](02-goals.md) · [Index](README.md) · [4. Placement relative to the Data Manager](04-placement.md) →
