# 4. Placement relative to the Data Manager


## 4.1 Two different products

| Concern | Data Manager | Data Store |
|---|---|---|
| Primary user | A data steward editing rows in a UI | Jobs, Wrangler, REST and AI agents |
| Unit of work | A row or cell, with status workflow and approval | A set: bulk load, query, swap |
| Volumes | Thousands to low millions, human-reviewable | Millions and up (replicas of CRM objects, staging) |
| Audit | Per-cell audit table for every data set | Per-operation audit (who loaded or queried what). Optional row audit columns. |
| Schema | Six column types. Layout stored as a serialized Java object (`layout_binary`). | Full CloverDX type system, versioned schemas |
| Lifetime | Long-lived, with row retention | Long-lived, run-scoped or replica-managed |
| Access control | Per data set role (READ_ONLY…ADMIN) | Per namespace and dataset ACL **plus row policy** |
| Query | Filter, sort and page on one data set (`RowFilter.toSqlWhereClause`) | Structured reads plus analytical SQL with joins |
| Licensing | Data Manager seats (`getMaxDataManagerUsers`) | `dataStore` module, with optional replica and storage limits, not seats ([§5.8](05-architecture.md#58-licensing)) |

## 4.2 Options

| Option | Description | For | Against |
|---|---|---|---|
| **A. Third DM type** | Add `DataSetType.MANAGED` to the DM model | Reuses the UI, ACL, REST, MCP tools and components | 74 `instanceof` checks and 19 `switch` branches on the two existing types. The persistence interface is split by type. Every SQL statement assumes DM system columns. The narrow type model, seat licensing and remote mode (DM data on another server) do not fit run namespaces. Risk to an established product. |
| **B. Separate subsystem, shared foundation** *(recommended)* | New `datastore` subsystem. The general parts of the DM's dialect code, patching, exception translation and datasource plumbing are split out into a shared storage layer that both use. | Clean model. No regression risk to DM. Reuses the proven parts. Same database possible. | Needs one refactoring to extract the shared layer. Two catalogs to present coherently. |
| **C. Fully independent** | New subsystem with its own storage code | No coupling at all | Duplicates `SchemaHandler`, patches and translators. A second managed DB to operate. |

**D1 (decided). Option B.** The Data Store has its own model, API, components, permissions and licence module. It shares a storage foundation with the Data Manager. The code shows what that foundation is, and that it is a **split, not a move**:

| Today (DM) | Shared foundation (`S/managedstorage`) | Stays in DM |
|---|---|---|
| `schema/SchemaHandler` (≈1,000 lines) plus `Postgre`/`Oracle`/`Mssql` subclasses (≈640 lines) | `ManagedSqlDialect`, the general helpers: `canRollbackDDL`, `quote`, `limitRowsStatement` / `rowLimitingClause`, `runWithLockTimeout`, `toTimestamp`, `escapeStringForLike`, boolean literals, temp table DDL, `batchInsertReturningIds`, identifier length, and drop with Oracle `PURGE` (which DM already does) | `SchemaHandler`, now built on `ManagedSqlDialect`: every method that takes a `DataSet`, plus the system, audit, change, lookup and error column layouts (`SystemColumn`, `AuditColumn`, …) |
| `Postgre`/`Oracle`/`MssqlExceptionTranslator` | Moved unchanged | |
| `dbschema/DbSchemaUpdate` (abstract: patch path, dialect, patch table) | Already generic. The store adds `StoreDbSchemaUpdate` with its own patch folder and patch table. | `DataManagerDbSchemaUpdate` |
| `datasource/DataManagerDataSource(Factory)` with retries (`datamanager.datasource.tries`) | A parameterised managed datasource factory | DM configuration keys unchanged |
| `DataSetLayout.ColumnMetadataType` (6 types) | Not shared. The store has its own type model (D6). | Unchanged |

The split lands as the first change of P0, reviewable on its own. **DM behaviour does not change**, as shown by the existing DM unit and integration tests run before and after the split.

**D2 (decided). Own configuration; may share the DM database.** The store has its own datasource configuration (`datastore.datasource.*`). The value `datastore.datasource.type=datamanager` reuses the DM pool and database:

- Store tables use the `DS_` prefix (and `DSR_` for run tables), optionally in their own DB schema.
- The store has its own patch table, so the two patch streams never interfere.
- Sharing is valid only when the Data Manager is configured locally (`DataManagerLocation.LOCAL`). In `REMOTE` mode the DM data lives on another Server and there is no local database to share. A store configured with `type=datamanager` then refuses to start and logs the reason.
- Co-location is what makes [§4.3.1](#43-integration-points-with-the-data-manager) possible, because the query compiler executes in a single database (Base §1). With separate databases, everything except joins with DM data still works.

## 4.3 Integration points with the Data Manager

These keep the two subsystems loosely coupled while looking like one product:

1. **DM reference data sets as read-only foreign datasets** (later phase, needs D2 co-location; referred to as [§4.3.1](#43-integration-points-with-the-data-manager)). When DM and the store share a database, the catalog can expose a reference data set as `dm.<code>`: a logical dataset whose physical scan is the DM `_data` table under a fixed policy (`_enabled` rows within validity). Dataset SQL can then join store data with stewarded reference data. Writes stay in DM.
2. **Data Catalog.** `S/datacatalog/DataCatalog` (the Wrangler source and target catalog) gets a third provider, *store datasets*, next to connectors and DM data sets.
3. **Publishing into DM.** A job can still move store data into a transactional DM data set for human review using today's DM writer. No new coupling is needed.

**Naming.** "Data set" is already taken by the Data Manager everywhere:

- the `DataSet` entity
- `/data-sets` routes
- the *Data Sets* web app
- the `dataset_*` MCP tools (CLO-37003 onwards)
- the `DataSet*Reader/Writer` components

DM data sets also already appear as Wrangler sources and targets. A second, unqualified meaning would confuse users, support staff and AI agents.

**D3 (decided, Q1).** The feature is called **Data Store**. Its objects keep the name **datasets**. These rules keep the two apart:

| Where | Rule |
|---|---|
| Any UI, doc or message where both kinds can appear (Wrangler source and target picker, Data Catalog, MCP tool descriptions, error messages) | Always qualified: **"Data Store dataset"** versus **"Data Manager data set"**, listed under separate headings. Spelling alone (*dataset* versus *data set*) is never relied on. |
| Store-only screens and docs | "dataset" alone is fine |
| Machine names | Always the `store` / `datastore` prefix, never `dataset`: package `datastore`, REST `/api/rest/data-store/v1`, permissions `data_store_*`, MCP `store_*`, components `StoreReader` / `StoreWriter` / `StoreQuery` / `StoreOperation` (type ids `STORE_READER`, …), lookup type `storeLookup`, config `datastore.*` |
| Glossary | One entry for each term, cross-referenced, in the user guide and the knowledge base the AI tools read |

All of these machine names are free today. The only "data store" in the codebase is the profiler's internal `LargeDataStoreProvider`.

---

← [3. What exists today](03-existing-platform.md) · [Index](README.md) · [5. Architecture](05-architecture.md) →
