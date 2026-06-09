# 13. Evaluation

How collabMEM&trade; is measured. Without a regression suite every retrieval, scoring, or extraction change is a guess; this chapter documents the eight scenarios that gate every PR touching the cognitive memory layer.

The full implementation spec lives in [`98-APPENDIX-eval-suite.md`](../../../planning/collabMEM/planning/IMPL_PLAN/98-APPENDIX-eval-suite.md). This chapter summarizes the *what* and *why* — read the appendix for the *how*.

---

## Why an eval suite

collabMEM&trade; is a stack of probabilistic components: scoring, spreading activation, multi-model consensus, compression, decay. Each layer is correct in isolation but failure modes compound across layers. A change in the inclusion threshold (Phase A) silently shifts what reinforcement loops (Phase B) see. A new embedding model (Phase C) shifts what the consolidation worker (Phase F) clusters. Without per-PR measurement, the system drifts.

The eval suite makes drift visible. Each scenario produces a JSON report with pass/fail plus key numeric metrics; per-run reports are gitignored, but a single committed `baseline.json` is the contract every PR must honor or explicitly update.

---

## What gets measured

Eight baseline scenarios, all deterministic and all fast (the full suite runs in under 5 seconds):

| Scenario | Validates | Phase that owned it |
|----------|-----------|---------------------|
| `relevance-floor` | Inclusion threshold + pin bypass | A.1 + A.2 |
| `suppression` | Suppressed engrams never enter prompts | A.2 |
| `pinning` | Pinned engrams activate within scope regardless of score | A.2 |
| `prompt-flatness` | Prompt size remains roughly flat as turns grow | A.11 + bounded-context invariant |
| `retrieval-recall` | Recall@1, @5, @10 across exact / synonym / paraphrase / abbreviation | B.10 baseline; tightened by B.1 |
| `over-activation` | Off-topic queries don't over-activate weakly related memories | A.1 baseline; tightened by B.1 |
| `digest-compression` | Digest preserves expected decisions / facts / preferences keywords | A.9 |
| `consensus` | Multi-model agreement / contradiction / single-model branches all behave correctly | A.10 + B.5 |

Later phases append scenarios — `lexical-prefilter-noregress` (B.10.5), `semantic-recall` (B.1), `spread-recall` (A.5), `entity-resolution` (B.3), `diversity` (MMR), `scope-isolation` (B.2), `sync-conflict` (B.4), `episodic-recall` (B.6), `dexie-migration` (cross-phase). Each new scenario follows the same `recordScenarioResult({ name, passed, metrics, failureMessages })` contract so reports aggregate cleanly.

---

## Pass criteria evolve with the system

The baseline is a moving target — by design.

When B.1 ships semantic embeddings, `recallAt5` jumps from ~0.4 (lexical-only) to ≥ 0.85. The eval suite's `retrieval-recall` scenario records both; the **pre-B.1 hard pass criterion** is "exact-match recall@1 = 1.0 AND recallAt5 ≥ 0.30 AND recallAt10 ≥ 0.40" — tight enough to catch real regressions, loose enough to pass on the lexical baseline. The B.1 PR description tightens these thresholds to the appendix-spec post-B.1 values and copies the new run report over `baseline.json`.

The same shape applies to `over-activation` (current pre-B.1 cap: ≤ 7 false positives; post-B.1 target: 0) and to `digest-compression` (current placeholder compressor returns ratio ≈ 0.95; post-A.9-frontend-wiring target: ≤ 0.55).

The CI compare script (`scripts/compare-eval-baseline.js`) hard-fails when a previously-passing scenario fails and warns when any numeric metric drops more than 5%. PR authors choose either to fix the regression or to update the baseline with a justification.

---

## Determinism is the foundation

The suite runs in jsdom against `fake-indexeddb`. Every Dexie database is uniquely named per scenario (`collaborAItr-collabmem-eval<scenario>`) and torn down in `afterAll`. Engram IDs are seeded from labels (`seedId('ts-strict')` always returns the same 32-char hex) so per-run reports diff cleanly. Timestamps in fixtures are derived from a fixed `FIXTURE_EPOCH` so decay-sensitive scenarios produce identical scores across runs and machines.

The suite never makes a live LLM call. Pre-computed embeddings are stored in `fixtures/embeddings.bin` (a flat little-endian Float32Array) — the format is specified in the appendix §8.1 so B.1's eval scenarios can load them without round-tripping `@huggingface/transformers`.

---

## Surfaces

- **Local:** `npm run test:collabmem-eval` (frontend/). Writes the latest report to `frontend/.collabmem-eval/<timestamp>.json`. The summary line on the way out tells you whether the suite passed.
- **CI gate:** `npm run test:collabmem-eval:compare-baseline` reads the latest run + the committed baseline, prints a markdown summary, exits non-zero on hard fails.
- **Baseline updates:** copy the latest run report over `.collabmem-eval/baseline.json`, commit with a message that explains why the baseline moved (e.g., "collabmem-eval: update baseline after B.1 (recall@5 0.4 → 0.92)").

The suite is dev tooling — it does not ship to users and does not execute in production. Its only output is the report on disk, the CI exit code, and the historical baseline trail in git.

---

## What this chapter does NOT cover

- **Production telemetry.** Live SLOs (prompt-size, latency) are surfaced via the admin Prompt-size SLO dashboard ([Chapter 11](11-implementation-notes.md), section "Prompt-size SLO"), not the eval suite.
- **A/B experiments.** The suite is a regression net, not a controlled experiment harness.
- **User satisfaction.** Implicit and explicit feedback loops (chapter [05](05-cognitive-cycle.md)) drive the reinforcement signal that the eval suite then validates.
