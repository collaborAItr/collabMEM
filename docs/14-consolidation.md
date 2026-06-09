# 14. Background Consolidation

The engram graph grows monotonically by default. Every conversation adds new engrams; reconciliation handles deduplication of *near-identical* concepts but doesn't address two slower forms of bloat:

1. **Conceptual fragmentation** — many engrams that, if read together, imply a more general fact that no individual engram states.
2. **Redundant clusters** — many engrams that say substantially the same thing in different words, escape per-extraction dedup because no single pair crosses the merge threshold, but as a group are clearly one idea.

Background consolidation is the periodic process that *proposes* fixes for both. It is the system's analogue of sleep-time replay and abstraction in biological memory — hence the colloquial label "dreaming."

---

## Design principles

- **Idle-time only.** Consolidation must never block a user-facing turn or contend for the same compute. Run when the document is hidden or the user is otherwise idle, and throttle aggressively (a once-per-day cadence is plenty for most stores).
- **Local first.** The engram graph already lives on the client. Cluster locally in a Web Worker so the main thread stays responsive and so the policy can adapt to per-user data without round-tripping to a server.
- **Propose, don't commit.** Consolidation produces *proposals*, not silent rewrites. Every consolidation lands as a row in a pending queue surfaced in the Memory Inspector. The user accepts or rejects each one. Auto-merge is a trust violation: the architecture's contract is that the user owns the memory.
- **Cheap clustering, expensive synthesis.** The clustering step (group engrams by embedding similarity) should be free. Producing a synthesized "macro-fact" or "redundant-collapse" headline can use a small LLM call, but only after a cluster passes a cohesion threshold.
- **No new edges from clustering alone.** Clustering produces *proposals*, not associations. Hebbian and explicit edges remain the only sources of associations; consolidation doesn't sneak edges into the graph through the side door.

---

## Pipeline

### 1. Idle detection

A worker wakes on visibility-change to `hidden`, on a long-press of an explicit "consolidate now" button, or on the daily throttle expiring while the page is loaded. It does nothing if any of those conditions fail.

### 2. Candidate selection

Operate on engrams within a single scope (typically `user` and `workspace`; `conversation` is too narrow to benefit). Limit to a recent or high-utility window — consolidating ten thousand engrams every night is wasteful when only a few hundred have changed.

### 3. Clustering

Greedy single-link clustering on engram embeddings, with two thresholds:

- A **macro-fact** threshold — looser, because the goal is to find groups of related-but-distinct engrams that imply a generalization.
- A **redundant-collapse** threshold — tighter, because the goal is to find near-duplicates that survived per-pair dedup.

Each cluster carries a cohesion score (mean intra-cluster similarity) used to rank proposals later.

### 4. Synthesis

For each surviving cluster, propose:

- For **macro-fact** clusters: a new engram whose `concept` is a short generalization and whose `content` references the cluster members.
- For **redundant-collapse** clusters: a single canonical engram that subsumes the others; the source engrams are slated for archival on acceptance.

Synthesis can be heuristic (longest common subject, weighted average sentiment) or LLM-assisted. Heuristic-only is fine for v1; LLM-assisted produces nicer headlines.

### 5. Persistence

Each proposal is written to the [Consolidation Proposals](04-asset-taxonomy.md#9-consolidation-proposals-optional--background-dreaming) store with `status: "pending"`. Existing engrams are not touched.

### 6. User review

The Memory Inspector surfaces pending proposals in a dedicated tab. Each proposal shows:

- The proposed concept and content.
- The source engrams that contributed (each clickable to its full card).
- The cohesion score and the kind (`macro_fact` vs `redundant_collapse`).
- Accept and Reject affordances.

On acceptance, the new engram is created and (for `redundant_collapse`) the source engrams transition to `archived` with a `superseded_by` provenance link. On rejection, the proposal is dismissed; nothing in the graph changes.

---

## What consolidation is *not*

- **Not a contradiction resolver.** Contradictions go through reconciliation. Consolidation operates on engrams that *agree* — clusters of related or duplicative facts, not opposing ones.
- **Not a garbage collector.** Stale engrams age out via the lifecycle / utility model; consolidation doesn't archive on its own outside the `redundant_collapse` flow, and even there it only proposes archival pending user approval.
- **Not a re-extractor.** Consolidation operates on *existing engrams*. It never re-reads the source conversations to extract new ones. That's the extractor's job.

---

## Failure modes to avoid

- **Over-consolidation.** Tighten the cohesion threshold before loosening it. A graph full of vague macro-facts ("user has opinions about software") is worse than no consolidation at all.
- **Hidden auto-merges.** Any code path that mutates engrams without an explicit user accept is a bug. The proposal queue is the contract.
- **Runaway worker.** Cap per-run clustering compute (max engrams considered, max proposals produced) so a long-idle session can't burn through the user's CPU budget.
- **Drift from reconciliation policy.** If reconciliation considers two engrams distinct, consolidation should not silently merge them through `redundant_collapse`. Use the same canonicalization rules.

---

## What to read next

- [04-asset-taxonomy.md §9](04-asset-taxonomy.md#9-consolidation-proposals-optional--background-dreaming) — the proposal asset shape
- [07-reconciliation.md](07-reconciliation.md) — the contradiction / dedup pathway that consolidation is *not* a substitute for
- [10-transparency-mutability.md](10-transparency-mutability.md) — the inspector surface where proposals are reviewed
