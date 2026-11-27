# CloverDX Data Store — Placement, Run Scopes, Replicas and Wrangler Integration

**Status:** Design agreed · 9 November 2026. All 29 decisions in [§12](12-decisions.md#12-decision-register) are agreed with the product owner, and every question raised has a decision ([§13](13-questions.md#13-open-questions)). Ready for the P0 proof of concept.
**Under review · 27 November 2026.** A design review against the product source ([findings](../review/2026-11-27-design-review.md)) added decisions D30–D38, all *proposed* and not yet agreed, and open questions Q20–Q25. D30–D33 affect the P0 and P1 design.
**Builds on:** *CloverDX Dataset Store — Technical Specification* (8 November 2026), cited throughout as **Base §n**. It is not part of this repository.
**Nature:** Design specification. Nothing here is implemented. Names, routes and property keys are proposals.
**Naming:** The Base spec calls the feature *Dataset Store*. It is renamed **Data Store** (D3). Its objects are still called *datasets*.

Conventions:

- **Dn**: a decision. [§12](12-decisions.md#12-decision-register) marks each one *decided* (agreed with the product owner) or *proposed* (the author's recommendation, still to be confirmed).
- **Qn**: a question raised during design. [§13](13-questions.md#13-open-questions) shows which decision answered it.
- Code references point into the CloverDX product source repository, not this one. `S/` = `cloveretl.server/src/main/java/com/cloveretl/server/`, `E/` = `cloveretl.engine/src/org/jetel/`.

## Chapters

- [1. Summary](01-summary.md)
- [2. Goals and non-goals](02-goals.md)
- [3. What exists today](03-existing-platform.md)
- [4. Placement relative to the Data Manager](04-placement.md)
- [5. Architecture](05-architecture.md)
- [6. Namespaces](06-namespaces.md)
- [7. Run-scoped namespaces](07-run-namespaces.md)
- [8. Replicas (shadow datasets)](08-replicas.md)
- [9. Wrangler integration](09-wrangler.md)
- [10. Components and APIs: delta to Base §4–§8](10-components-and-apis.md)
- [11. Phasing](11-phasing.md)
- [12. Decision register](12-decisions.md)
- [13. Open questions](13-questions.md)
- [Appendix A: code references](appendix-a-code-references.md)
