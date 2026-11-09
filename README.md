# CloverDX Data Store: design specification

This repository holds the design specification for the **CloverDX Data Store**, a planned CloverDX Server subsystem that provides:

- managed tabular datasets
- run-scoped working storage for job trees
- replicas of slow sources, such as Salesforce
- Wrangler integration

Nothing here is implemented yet. The spec is versioned here, apart from the product source, so that it can be discussed and changed through pull requests.

**Start here:** [spec/README.md](spec/README.md), which has the status, the conventions and the chapter list.

## How to read it

1. [Summary](spec/01-summary.md) gives the whole design on one page.
2. The [decision register](spec/12-decisions.md) lists every decision (`D1`, `D2`, …) with its status.
3. Each chapter explains the reasons behind its decisions and links to the others.

Diagrams are written in Mermaid inside the Markdown and render directly on GitHub.

## How to contribute

**Questions and objections go in issues:**

- Use **Open question** for something the spec does not answer yet.
- Use **Challenge a decision** to argue that a decision `Dn` should change. Bring evidence, such as a code reference, a measurement or a customer case.

**Changes to the text go in pull requests:**

- Make one topic per pull request, so that a review stays focused.
- Comment on lines in the pull request for wording. Use the pull request conversation for substance.
- When a pull request changes a decision, update the chapter **and** the [decision register](spec/12-decisions.md) in the same pull request. When it answers a question, also update the [questions](spec/13-questions.md) table.

**Numbering rules:**

- `D` and `Q` numbers are permanent. They are never reused or renumbered.
- A new decision takes the next free `D` number; a new question takes the next free `Q` number.
- A decision that is replaced is not deleted. Its register row is struck through and points to the decision that replaces it.
- A decision is marked *decided* only after the product owner agrees in the pull request. Until then it is *proposed*.

**Writing conventions:**

- Cross-references to other sections are links, written `[§7.3](07-run-namespaces.md#73-…)`. Use the GitHub preview to check that the anchor works.
- `Base §n` refers to the earlier *CloverDX Dataset Store — Technical Specification*, which is not in this repository.
- Code paths are written `S/…` (Server) and `E/…` (engine), relative to the CloverDX product source repository. When you cite code, check the path against the current source.
