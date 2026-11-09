# 1. Summary


The Base spec describes a server-hosted store of managed tabular datasets: logical names mapped to RDBMS tables, CloverDX metadata, row policy, REST and graph access, and analytical SQL compiled by Calcite. This document adds four things:

1. **Placement.** The Data Store is a **separate subsystem** from the Data Manager. It shares a refactored storage foundation and can share the Data Manager's database, but has its own model, API, permissions and licence module ([§4](04-placement.md#4-placement-relative-to-the-data-manager)).
2. **Namespaces and run scopes.** Every dataset lives in a *namespace*. A job tree (a root run and every descendant) gets its own isolated **run namespace**, keyed by `ROOT_RUN_ID`. Temporary datasets in it are dropped when the tree finishes. A dataset built there can be **published** atomically into a shared namespace ([§6](06-namespaces.md#6-namespaces), [§7](07-run-namespaces.md#7-run-scoped-namespaces)).
3. **Replicas.** A replica is a store dataset bound to a slow source, such as a Salesforce connector. The connector author declares what can be replicated in `library.json`, and the operator defines instances with a schedule, freshness target, retention and access. Refreshes are validated before they are swapped in, and consumers pin one snapshot per run ([§8](08-replicas.md#8-replicas-shadow-datasets)).
4. **Wrangler.** A Server-side hook in the existing conversion swaps the connector SUBGRAPH for a store reader, leaving the downstream graph unchanged. Source previews can page, sort and filter the full replica server-side. Large lookups run against an indexed store copy that keeps the current first-row-wins semantics, and Excel/CSV lookups are parsed once into a *staged dataset* ([§9](09-wrangler.md#9-wrangler-integration)).

The codebase already has most of the foundation:

- **Calcite 1.41.0 and JSqlParser 5.3 are already Server dependencies.** This removes the main dependency-footprint concern in Base §9.
- The Data Manager already runs a separate managed database (PostgreSQL, Oracle, SQL Server) with per-dialect DDL.
- `ROOT_RUN_ID` already reaches every descendant job.
- Wrangler already has a preview-only source cache keyed by parameter hash, and converts every job on the Server through a `ConversionContext` that a replica hook can join.
- The Salesforce readers already support `includeDeleted` (`queryAll`), which incremental replicas need to see deletes.

---

[Index](README.md) · [2. Goals and non-goals](02-goals.md) →
