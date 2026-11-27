# 13. Open questions


| # | Question |
|---|---|
| Q1 | ~~User-facing noun: *dataset* or something that does not collide with the Data Manager?~~ Decided: D3. |
| Q2 | ~~Licensing model: module flag, capacity or per replica?~~ Decided: D25. |
| Q3 | ~~Should shared namespaces be bound to sandboxes (default namespace, permission inheritance)?~~ Decided: D24. |
| Q4 | ~~Time-zone policy for `date`?~~ Decided: D26. |
| Q5 | ~~Can a job outside the ExecuteGraph tree join a run namespace with a root-issued token?~~ Decided: D20. |
| Q6 | ~~Tree outcome for retention: the root's status or the worst member status?~~ Decided: D21. |
| Q7 | ~~Embedded local store for offline Designer work?~~ Decided: D22. |
| Q8 | ~~Parameter-to-predicate pushdown in the first replica release?~~ Proposed: no, identity-only matching first ([§8.3](08-replicas.md#83-identity-and-parameters)). |
| Q9 | ~~Where are replicas defined?~~ Proposed: D9. |
| Q10 | ~~Should recipe step prefixes be pushed into SQL for Wrangler step previews?~~ Decided: D27 (no). |
| Q11 | ~~Are CTL functions over the store wanted?~~ Decided: D28. |
| Q12 | ~~Should `SCD2` history for incremental replicas ("as of" reads) be in scope at all?~~ Decided: D29 (no SCD2. A change feed instead.) |
| Q13 | ~~Per-principal replicas?~~ Proposed: not in the first release. Connectors using per-user credentials stay live ([§8.8](08-replicas.md#88-security)). |
| Q14 | ~~A connector parameter that still holds a `${…}` reference at conversion time: resolve it against the sandbox's parameters for matching (proposed), or always read such a source live?~~ Decided: D15. |
| Q15 | ~~Self-service replicas from Wrangler: may any Wrangler user holding `data_store_create` create an instance, or does an administrator approve it (API quota, storage)?~~ Decided: D16. |
| Q16 | ~~Default for unattended Wrangler runs over a stale replica: `USE_WITH_WARNING` (proposed, matches previews) or `REFRESH_AND_WAIT`?~~ Decided: D17. |
| Q17 | ~~Staged file datasets for main Excel/CSV sources as well, or only for lookup sources (proposed for the first release)?~~ Decided: D18. |
| Q18 | ~~Should incremental refresh move from P5 into P3? FULL refresh already covers the nightly-unload scenario, and the Salesforce components already support `includeDeleted`.~~ Decided: D19. |
| Q19 | ~~Data service requests using `run.*`?~~ Decided: D23. |
| Q20 | Default of `datastore.lease.mode`: `ALL` (as D4 has it), or `SYSTEM_JOBS`, so that object ACL is a hard boundary unless a deployment opts out? The P0 throughput comparison of leased and proxied loads should inform this ([§5.3](05-architecture.md#53-data-plane-how-records-travel), D31). |
| Q21 | Above what table size should an admin choose `IN_PLACE` for an incremental replica, given the cost of one in-database copy per refresh on each of the three databases? To be measured in P3 (D35). |
| Q24 | API tokens for applications: a new store object, or an existing Server mechanism (D34)? |
| Q25 | Should restarting a failed job let the new tree adopt the kept run namespace of the tree it came from? A restart always starts a new root today, so the restarted job cannot address the old tree's `run.*` data except read-only by name ([§7.3](07-run-namespaces.md#73-dataset-lifetimes-inside-a-run-namespace)). |

---

← [12. Decision register](12-decisions.md) · [Index](README.md) · [Appendix A: code references](appendix-a-code-references.md) →
