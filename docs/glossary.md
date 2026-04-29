# Glossary

Definitions for the terms used throughout the collabMEM architecture documentation. Terms are grouped by area. Cross-references link to the chapter where the term is developed in depth.

---

## Core concepts

**Cognitive memory layer.** The part of an LLM chat application that extracts, organizes, stores, activates, and reinforces structured memory from conversation turns. The subject of this documentation set. Its **primary purpose** is to keep the assembled prompt sent to the assistant at a bounded size well below the model's context-window limit, regardless of conversation length — see **context-window strain** below.

**Context window.** The fixed token limit of an LLM's input. The assistant can only see what fits here.

**Context window strain.** The compounding failure mode of stacking verbatim turns into the prompt as a conversation grows: rising cost per turn, attention dilution, lost-in-the-middle, and eventual truncation at the hard ceiling. Reducing context-window strain is the **primary purpose** of this architecture. See the README's ["context-window problem" section](../README.md#the-context-window-problem).

**Context footprint.** The size of the assembled prompt, measured in tokens. This architecture treats the footprint as a **bounded** quantity that should not grow with conversation length.

**Bounded prompt.** A prompt whose total size is capped by an explicit ceiling chosen by the product, much lower than the model's context-window limit, and stable across turn count. Producing a bounded prompt every turn is the primary job of the activation pass.

**Salient-history substitution.** The pattern of using compact salient digests in the prompt's recent-history section in place of verbatim prior turns. This is where the architecture earns most of its context-window savings.

**Cognitive cycle.** The five-stage loop that memory passes through: extract, reconcile, store, activate, reinforce. See [05-cognitive-cycle.md](05-cognitive-cycle.md).

**Asset.** A typed memory object stored by the cognitive layer. The six asset types are engrams, associations, salient digests, Meta-Vault entries, consensus reports, and extraction log entries. See [04-asset-taxonomy.md](04-asset-taxonomy.md).

**Provenance.** The source metadata attached to every asset: which conversation, which turn, which producer(s), when, and by what origin (extracted / manual / imported).

**Scope.** The breadth over which an asset is meaningful: user, workspace, project, or conversation. Used to prevent leakage between contexts that should stay separate.

**Vault.** A scope-isolated partition of the store. The smallest useful unit is one vault per user. See [08-storage.md](08-storage.md).

---

## Asset types

**Engram.** A durable memory unit: one concept, one content body, one confidence, one utility score. The primary recall target. Named after Richard Semon's 1904 term for a physical trace of experience. See [02-conceptual-foundation.md](02-conceptual-foundation.md) and [04-asset-taxonomy.md](04-asset-taxonomy.md#1-engrams).

**Association.** A weighted, typed edge between two engrams. Captures either an explicit semantic relationship (proposed by an extractor) or an emergent co-activation pattern (from usage). Named after Hebbian association in neuroscience. See [04-asset-taxonomy.md](04-asset-taxonomy.md#2-associations).

**Salient digest.** A compact, per-turn structured summary of a conversation exchange. Contains decisions, facts, open questions, action items, and domain preferences. Salient digests are what the activation pass uses in place of verbatim turn history when assembling the next prompt — they are the key mechanism by which this architecture avoids stacking transcripts into the context window. See [04-asset-taxonomy.md](04-asset-taxonomy.md#3-salient-digests) and [09-retrieval-activation.md](09-retrieval-activation.md#replacing-verbatim-history-with-salient-digests).

**Meta-Vault entry.** A durable cross-conversation pattern about the user, workspace, or collaboration style (preference, style, domain knowledge, workspace convention). Broader in scope than engrams. See [04-asset-taxonomy.md](04-asset-taxonomy.md#4-meta-vault).

**Consensus report.** The durable record of a multi-producer reconciliation pass: what was proposed, how proposals were grouped, agreement ratios, contradictions. Optional — only produced when reconciliation merges multiple proposals. See [04-asset-taxonomy.md](04-asset-taxonomy.md#5-consensus-reports-optional--only-when-reconciliation-produces-multiple-proposals).

**Extraction log entry.** Per-extraction-run telemetry: success/failure, duration, counts produced, errors. Optional but recommended. See [04-asset-taxonomy.md](04-asset-taxonomy.md#6-extraction-log-optional--operational-telemetry).

---

## Extraction

**Extraction.** The first stage of the cognitive cycle: turning a completed turn into structured memory candidates. See [06-extraction.md](06-extraction.md).

**Extractor.** The component that performs extraction. Typically an LLM with a structured-output prompt, possibly smaller and cheaper than the chat model.

**Inboard Agent.** The name used in the reference implementation for the dedicated extraction component. Used interchangeably with "extractor" in these docs.

**Extraction contract.** The five-category structured output every extractor must produce: salient digest, proposed engrams, relationships, meta-insights, extraction quality. See [06-extraction.md](06-extraction.md#output-contract).

**Decoupled stream.** The architectural pattern where extraction runs on a separate stream or background job from chat, so chat latency never depends on memory work. See [06-extraction.md](06-extraction.md#the-decoupled-stream-pattern).

**Fallback extractor.** A secondary model used when the primary extractor fails. Usually in a different family for independence.

**Minimal fallback digest.** The minimum acceptable extraction output when all attempts fail — typically just a summary of the turn. Never leave a turn with zero memory output.

---

## Reconciliation

**Reconciliation.** The optional-as-a-distinct-layer-but-conceptually-required stage between raw extraction output and durable memory. Validates, deduplicates, merges, detects contradictions, routes to review. See [07-reconciliation.md](07-reconciliation.md).

**Reconciliation mode.** One of the six supported approaches: human-in-the-loop review, verifier pass, rule-based validation, deferred correction via mutation, multi-producer agreement, ensembled same-model agreement. Modes are composable. See [07-reconciliation.md](07-reconciliation.md#the-six-reconciliation-modes).

**Verifier pass.** A reconciliation mode that uses a second LLM to critique and accept/reject/revise each proposal.

**Multi-producer agreement.** A reconciliation mode that treats independent agreement across multiple producers (for example, multiple models in parallel) as a confidence signal.

**Ensembled same-model agreement.** A reconciliation mode that calls the same extractor multiple times and treats the results as if they came from independent producers.

**Deferred correction.** A reconciliation mode that stores proposals with minimal validation and relies on the user to correct memory over time via the mutation surface.

**Agreement ratio.** In multi-producer reconciliation: the number of producers that proposed a given (merged) engram divided by the total number of participating producers.

**Contradiction.** A detected conflict between proposals with highly similar concept labels but semantically opposed content. Preserved as a typed `contradicts` association, not silently resolved.

**Representative proposal.** When multiple proposals are merged into one engram, the representative is the single proposal chosen to carry forward. Tie-breaks include confidence, length, and producer priority. Implementation choice.

---

## Storage

**Local-first.** The architectural principle that the primary memory store lives close to the user (browser, device, personal workspace). Cloud is a mirror, not a source of truth. See [08-storage.md](08-storage.md).

**Sync batch.** A bounded set of recent changes exported from the local store and applied to the cloud mirror, or vice versa. Idempotent, dependency-ordered. See [08-storage.md](08-storage.md#sync-batch-structure).

**State.** An asset's current lifecycle status: active, archived, superseded, pending, suppressed, or deleted. Explicit states are preferred over silent deletions.

**Active.** An asset's state when it is available for activation.

**Archived.** An asset's state when it is retained for history but excluded from normal activation.

**Suppressed.** An asset's state when it is retained but never injected into prompts. An explicit user choice; absolute.

**Pinned.** A user override that raises an asset's priority, causing it to bypass the relevance floor during activation.

**Pending.** An asset's state when it has been proposed but not yet approved for activation (typical in human-in-the-loop reconciliation).

---

## Activation

**Activation.** The stage that brings stored memory back into the prompt for the next turn. Preferred over "retrieval" because activation also reinforces what it touches. See [09-retrieval-activation.md](09-retrieval-activation.md).

**Activation engine.** The component that performs activation. Runs synchronously in the chat path.

**Query classification.** Pattern-based or model-based categorization of the user's turn into query types (temporal, factual, associative, general) to adjust scoring weights.

**Scoring axes.** The multiple signals used to score an engram against a query: text match, recency, temporal proximity, utility, confidence, Hebbian boost, scope match, user priority.

**Seed.** A small number of top-scoring engrams selected during activation. Hebbian boost is computed relative to seeds.

**Hebbian boost.** An addition to an engram's activation score equal to the summed weights of associations connecting it to any seed. The mechanism by which associations pull neighbors into the prompt.

**Priming.** Informal term for the effect of an engram's activation raising the likelihood of its associated engrams being activated. The Hebbian boost formalizes priming in this architecture.

**Pathway A.** Semantic association creation: the extractor proposes an edge between engrams explicitly. See [02-conceptual-foundation.md](02-conceptual-foundation.md#how-associations-are-created).

**Pathway B.** Usage-driven association creation and reinforcement: every time two engrams are co-activated, their association's weight is nudged and its co-activation counter is incremented. See [02-conceptual-foundation.md](02-conceptual-foundation.md#how-associations-are-created).

**Utility score.** An engram's learned usefulness, updated based on whether activations of it contributed to useful turns. High utility = likely to be activated again.

**Co-activation.** An event in which two engrams are activated in the same turn. Each co-activation reinforces the association between them.

**Relevance floor.** The minimum activation score required for an engram to be included in the prompt, even if the token budget has room.

**Token budget.** The cap on how much of the assembled prompt can be memory context. Divided across sections (engrams, salient history, verbatim recent turns, Meta-Vault). The token budget is the mechanism that enforces a **bounded prompt** — it is a hard ceiling chosen by the product, set well below the model's context-window limit, and stable across conversation length. Implementation choice for the specific numbers.

**Greedy trimming.** The pattern of dropping the lowest-scoring items first when a budget section overflows. Trims at the asset boundary, never in the middle of text.

---

## Reinforcement

**Reinforcement.** The fifth stage of the cognitive cycle: updating memory based on how activation went. Nudges utility scores and association weights. See [05-cognitive-cycle.md](05-cognitive-cycle.md#stage-5-reinforce) and [09-retrieval-activation.md](09-retrieval-activation.md#7-the-feedback-loop).

**Decay.** The gradual lowering of an un-accessed engram's utility score. Prevents stale memory from dominating activation. Rate is an implementation choice; gentle over weeks is typical.

---

## Transparency

**Memory Inspector.** The UI surface that answers "what does the system remember?" Shows every stored asset with full provenance and mutation controls. See [10-transparency-mutability.md](10-transparency-mutability.md#memory-inspector-what-it-must-show).

**Payload Inspector.** The UI surface that answers "what did the assistant actually receive?" Shows the full assembled prompt for a given turn, sectioned and audit-linked. See [10-transparency-mutability.md](10-transparency-mutability.md#payload-inspector-what-it-must-show).

**Dual-surface pattern.** The architectural requirement that both a Memory Inspector and a Payload Inspector exist as peer surfaces. Neither replaces the other.

**Mutability.** The requirement that users can edit, delete, archive, suppress, pin, approve, reject, or restore memory. A core requirement, not a polish item. See [10-transparency-mutability.md](10-transparency-mutability.md#mutability-users-are-first-class-editors).

**Activity timeline.** A chronological record of extraction, reconciliation, activation, mutation, and sync events. Surfaced in the Memory Inspector's Activity tab. Append-only.

---

## Historical figures (for the biology and cognitive-science inspiration)

**Richard Semon (1859–1918).** German biologist who coined the term "engram" in 1904 to describe a hypothetical physical trace of experience. See [02-conceptual-foundation.md](02-conceptual-foundation.md#1-engrams--memory-traces-in-silicon).

**Donald Hebb (1904–1985).** Canadian neuroscientist who proposed in 1949 that "neurons that fire together wire together," formulating the principle behind associative memory. See [02-conceptual-foundation.md](02-conceptual-foundation.md#2-hebbian-associations--wiring-ideas-together).

**John R. Anderson (b. 1947).** American cognitive psychologist at Carnegie Mellon University; architect of the ACT-R framework (Adaptive Control of Thought—Rational, 1976 onward). ACT-R's base-level activation formula (recency and frequency of past access) grounds the temporal-priority and utility axes in this architecture. See [09-retrieval-activation.md](09-retrieval-activation.md#2-scoring-axes).

**Allan M. Collins** and **Elizabeth F. Loftus.** Cognitive psychologists whose 1975 paper "A Spreading-Activation Theory of Semantic Processing" formalized the model where activation propagates along weighted edges to neighboring concepts. Grounds the seed-plus-Hebbian-boost pattern. See [09-retrieval-activation.md](09-retrieval-activation.md#3-seed-selection-and-hebbian-boost).

**Hermann Ebbinghaus (1850–1909).** German psychologist whose 1885 experimental work on the forgetting curve grounds the utility-decay rule used to prevent stale memory from dominating activation. See [05-cognitive-cycle.md](05-cognitive-cycle.md#stage-5-reinforce).

---

## Academic references

A consolidated list of the classical and contemporary sources cited across these docs lives in the [README's Academic references section](../README.md#academic-references). Individual chapters that lean on specific sources include chapter-appropriate subsets at the end of each chapter:

- [02-conceptual-foundation.md](02-conceptual-foundation.md#academic-references) — Semon (1904), Hebb (1949), Collins & Loftus (1975)
- [05-cognitive-cycle.md](05-cognitive-cycle.md#academic-references) — Hebb (1949), Anderson & Lebiere (1998), Ebbinghaus (1885)
- [07-reconciliation.md](07-reconciliation.md#academic-references) — Fellegi & Sunter (1969), Condorcet (1785), Dietterich (2000)
- [09-retrieval-activation.md](09-retrieval-activation.md#academic-references) — Anderson & Lebiere (1998), Anderson & Schooler (1991), Collins & Loftus (1975), Hebb (1949), Liu et al. (2024), Rocchio (1971)
- [01-philosophy.md](01-philosophy.md#academic-references) — Liu et al. (2024)

---

## Related but distinct

**Chat history.** The full record of user and assistant messages. Distinct from memory — memory is extracted from chat history, not the same thing as it.

**Vector embedding.** A numerical representation of text used for similarity search. This architecture is compatible with embedding-based similarity as one activation axis but does not require it.

**RAG (Retrieval-Augmented Generation).** A broader pattern of pulling external content into a prompt before generation. This architecture is a specific, structured, memory-focused instance of that broader pattern.

**Context window (related concept).** Defined above in Core concepts. Listed here as well because "context window" is sometimes used loosely to mean "everything the model has seen"; in this architecture it means strictly the token limit of the model's input and the bounded workspace the activation pass assembles within that limit.
