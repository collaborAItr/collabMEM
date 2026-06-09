# 15. Federated Workspace DataBANK

The DataBANK stores per-user sentiment data — durable subject/sentiment summaries derived from the DataTag stream (e.g. `(subject: "tabs over spaces", sentimentAvg: +0.7, observations: 14)`). Each entry is private by default. A user can mark an entry **shareable**, which opts the underlying observations into a workspace-level rollup the rest of the workspace can see.

This chapter covers the shape and policy of that workspace-level rollup.

---

## What problem this solves

A workspace (an org, team, family, or any multi-user collaboration scope) has emergent shared opinions. "Most of the team prefers Postgres over MySQL." "We've all been positive on the new design system." Surfacing that without violating individual privacy is the federation problem.

The naïve solution — concatenate every member's DataBANK into one big shared bag — is wrong on two counts:

1. **Privacy.** Some entries are private and must never appear in shared surfaces.
2. **Statistical noise.** A subject with one strongly positive contributor reads as `+0.9 average`. A subject with twelve mildly positive contributors reads as `+0.3 average`. Naïve averaging ranks the noisy single-vote subject above the broad-consensus subject. That's a bug.

Federated DataBANK aggregates address both.

---

## What gets aggregated

For each unique `(workspace, category, subcategory, subject)` tuple, the aggregator computes:

- `aggregateSentimentMean` — Bayesian-shrunk mean (see below) across every contributing member's `sentimentAvg`.
- `aggregateSentimentVariance` — sample variance of the contributors' means; surfaces disagreement.
- `contributorCount` — distinct member count.
- `contributorIds` — the contributing user ids (used for back-attribution if a member later opts out).
- `shareableEntryIds` — pointer back to the source DataBANK rows.

Only entries with `shareable: true` enter the rollup. Entries with `shareable: false` are invisible to the aggregator.

---

## Bayesian shrinkage

The Bayesian-shrinkage formula is:

```
adjustedMean = (n * sampleMean + α * priorMean) / (n + α)
```

where:

- `n` is the contributor count.
- `sampleMean` is the unweighted mean of the contributors' `sentimentAvg` values.
- `priorMean` is the architectural prior — `0` (psychographically neutral) is the right default; the workspace has no opinion until evidence arrives.
- `α` is the prior strength. A value around 5 works well: it pulls single-contributor subjects strongly toward neutral, treats a small handful of contributors as moderately reliable, and lets large-`n` subjects pass through nearly untouched.

The intuition: a subject with one `+0.9` contributor produces an adjusted mean of about `+0.15`, because one voice isn't enough to claim the workspace agrees. A subject with twelve `+0.3` contributors produces an adjusted mean of about `+0.21`, because the consensus is broad enough to count even though no individual voice is loud. Sorting by `adjustedMean` (or by `contributorCount` desc, `|adjustedMean|` desc as a useful secondary order) surfaces real consensus instead of loudest-single-voice noise.

---

## What the federation surface is *not*

- **Not a cross-user gossip channel.** Aggregates show *what the workspace as a whole leans toward*, not who said what. `contributorIds` exists for back-attribution on opt-out, not for "who else thinks X?" queries from peers.
- **Not a vote.** A user with strongly held minority views is still entitled to their private DataBANK entry. The aggregate represents the shared frame, not the canonical truth.
- **Not retroactive.** Toggling `shareable` from false to true contributes from that moment onward; the aggregate doesn't dig into prior state. Toggling from true to false removes the contributor from the next recompute.
- **Not used by extraction.** The extractor reads the user's *own* DataBANK and any explicitly-injected workspace context, never the federated aggregate. Aggregates are an inspection / dashboard surface; injecting them into the prompt without explicit consent is a privacy regression.

---

## Recompute cadence

Aggregates are pure functions of the underlying DataBANK rows, so recompute is idempotent. Two reasonable cadences:

- **Nightly background job** for steady-state. Cheap; everyone wakes up to fresh aggregates.
- **On-demand recompute** triggered from the Memory Inspector. Useful when a member just toggled sharing and wants to see the workspace view update without waiting for the next cycle.

The reference implementation supports both. A minimum useful version ships with on-demand only; the nightly job is a layer on top.

---

## What lives where

- **Per-user DataBANK rows** stay in the per-user store, gated on the per-row `shareable` flag.
- **Workspace aggregates** live in a workspace-scoped store keyed by `(workspaceId, category, subcategory, subject)`. They are derived data; they can always be regenerated from the source rows.

The aggregate store is **not** a memory asset in the sense of [04-asset-taxonomy.md](04-asset-taxonomy.md). It's a query convenience derived from existing assets. Treat it the way you'd treat a materialized view, not the way you'd treat an engram.

---

## What to read next

- [04-asset-taxonomy.md §10](04-asset-taxonomy.md#10-workspace-databank-aggregates-optional--federated) — the aggregate row shape
- [12-sync-and-trust.md](12-sync-and-trust.md) — the privacy contract that gates `shareable`
- [13-evaluation.md](13-evaluation.md) — measuring whether federated surfaces actually help workspace decisions
