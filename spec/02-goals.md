# 2. Goals and non-goals


**Goals**

- G1. One logical catalog of typed tabular datasets, usable from graphs, jobflows, Wrangler, REST and MCP.
- G2. Isolated, self-cleaning working storage for a job tree.
- G3. Fast, repeatable access to data from sources that are slow, rate-limited or not random-access, with explicit freshness.
- G4. Bounded memory for Wrangler lookups and previews, with no change to Wrangler recipe semantics.
- G5. Policy-enforced reads (ACL plus row policy) on every platform path. Row policy is a hard boundary even against user code in workers ([§5.3](05-architecture.md#53-data-plane-how-records-travel)).

**Non-goals (this iteration)**

- Federated queries across different physical databases (Base §1).
- Change data capture (CDC) from sources. Incremental replicas use source-side watermarks only. The replica change feed (D29) is built from those watermark deltas, not from source logs.
- SCD2 history and "as of T" reads for incremental replicas (Q12).
- Human data stewardship workflows: approval, per-cell audit, staged edits. Those remain the Data Manager's job.
- A replacement for the Wrangler preview cache for plain file sources. It stays ([§9.8](09-wrangler.md#98-relation-to-the-existing-preview-cache)).
- SQL pushdown of Wrangler recipe steps (D27).

---

← [1. Summary](01-summary.md) · [Index](README.md) · [3. What exists today](03-existing-platform.md) →
