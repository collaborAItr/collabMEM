# 01. Philosophy

This chapter sets expectations for every other chapter. If you read only one document before deciding whether to adopt the architecture, read this one.

## What this documentation set is

It is an **architectural reference** for building a cognitive memory layer into an LLM chat application. It describes:

- the typed memory objects that make up the store
- the lifecycle those objects pass through
- the extraction, reconciliation, storage, and activation patterns that move data through that lifecycle
- the UI obligations that come with storing memory that shapes future prompts
- the trade-offs each of those decisions imposes

It is written to be consumed by both human developers and LLM coding assistants. Chapters are self-contained enough that any one can be pasted into a model's context window as reference material for a specific implementation task.

## What this documentation set is NOT

- **It is not a runtime.** There is no binary, server, daemon, or hosted service.
- **It is not a library, SDK, or package.** There is no `npm install`, no `pip install`, no `cargo add`.
- **It is not source code.** No module in this repository can be imported or called.
- **It is not a product.** It is the design rationale for one.
- **It is not a prompt pack.** The extraction prompt is an implementation detail; this repository defines only the output contract the extraction must satisfy.
- **It is not a specific database recipe.** The architecture is compatible with relational, document, graph, vector, and hybrid stores. None of them is architectural.
- **It is not a privacy, security, or compliance policy.** Those are your responsibility and depend on your jurisdiction, user base, and product surface.
- **It is not a guarantee.** Adopting this architecture will not automatically make your chat app good. It will, if followed, prevent a long list of predictable failures.

## Model-agnostic by design

The same architecture supports:

- **Single-model chat apps.** One user, one assistant. Extraction produces one set of memory candidates per turn. Reconciliation, if used, is a single-producer pattern (human review, rule-based validation, or a second-pass verifier).
- **Multi-model chat apps.** One user, several assistants running in parallel on the same turn. Each assistant's response can produce its own extraction. Reconciliation merges proposals and treats independent agreement as a confidence signal.
- **Agentic systems.** One user, many roles (planner, researcher, critic, tool agent). Reconciliation tracks the agent role in provenance and can require review for memory produced by autonomous agents.

No chapter in this set assumes a specific number of models or agents. Where a detail depends on that choice, the chapter calls it out explicitly.

## The primary purpose: reduce context-window strain

Before the principles, state the purpose that motivates them.

The cognitive layer exists to keep the prompt sent to the assistant at a **stable, bounded size** that sits well below the model's maximum context window, regardless of how long the conversation has run. It does this by replacing the default pattern of stacking verbatim turns with a pattern of stacking **compact, selected memory**:

- verbatim recent turns are replaced by **salient digests** that preserve the important content in a fraction of the tokens
- long-term knowledge is moved out of the prompt and into **typed memory objects** that are injected only when relevant
- the **activation engine** enforces a token budget and selects what enters the prompt; the prompt does not grow just because the conversation does

This is the single biggest practical benefit of adopting this architecture. Every principle below either enables this behavior directly or protects it from the design regressions that would erode it.

## Core principles

These are the invariants. An implementation may vary in every other respect, but if it violates these, it is no longer an instance of this architecture.

### 1. Memory is product state, not a side effect

Memory that shapes future prompts is not a hidden internal cache. It is durable product state, on the same tier as saved preferences, documents, or project settings. It deserves the same treatment:

- explicit types
- versioned fields
- provenance on every record
- a visible state (active, archived, pending, suppressed, deleted)
- a user-facing surface that exposes and mutates it

### 2. Transparency comes before automation

Before adding any deeper behavior — confidence scoring, auto-merging, reinforcement, decay — make sure the user can see what exists. A cognitive layer that silently accumulates memory is worse than one that accumulates none.

At minimum, an implementation must expose:

- what memory objects exist
- where each one came from
- whether it was injected into a specific prompt
- when it was last updated or reinforced
- whether it is uncertain, stale, or user-edited

### 3. Manual mutability is a core requirement

The user must be able to change memory. Not in a future release. At v1.

The minimum mutation surface is:

- edit content
- delete
- archive
- suppress (keep but never inject)
- pin (raise activation priority)
- approve or reject pending memory

User edits always override automated extraction. If an extraction pipeline later proposes a change to a user-edited item, the pipeline must ask for approval or preserve the user value.

### 4. Memory is typed

A flat pile of notes is easy to build and hard to use. Typed memory lets each kind of knowledge have its own extraction rules, activation behavior, display, and review policy.

The six asset types in [chapter 04](04-asset-taxonomy.md) are the minimum useful taxonomy. An implementation can add more; it should not collapse these into one.

### 5. Retrieval should be selective and bounded

The goal is not to send more context to the assistant. It is to send **better** context, and to send it within a **bounded token budget** that does not grow with the conversation. "Better" means a compact, relevant set of typed memory selected by an activation engine that considers text match, recency, temporal fit, utility, confidence, associations, and user controls. "Bounded" means the prompt sent at turn 200 of a long conversation is approximately the same size as the prompt sent at turn 10 — just a different selection.

Sending the whole store, the whole history, or the top-K by cosine similarity alone does not count as activation. Letting the prompt grow linearly with the conversation does not count either. See [chapter 09](09-retrieval-activation.md).

### 6. Local-first where possible

The primary store should live close to the user. Optional cloud sync can be added deliberately. This order matters:

- local-first gives privacy, offline use, and user ownership for free
- cloud-first gives convenience and forces an explicit privacy story

Most personal chat apps should default to local-first storage with explicit, auditable sync. See [chapter 08](08-storage.md).

### 7. Decoupled extraction

Extraction runs **after** the assistant has responded, in a separate stream or background job. Chat UI latency must not depend on extraction latency. The production reference implementation uses a dedicated secondary stream (SSE) for extraction results so the chat surface stays responsive.

An implementation that makes chat wait for extraction is not this architecture.

### 8. Reconciliation is configurable

Memory should become durable only after some organization or review. "Some" ranges from:

- schema validation + user review (simplest single-model app)
- a verifier LLM pass (single-model with model-assisted quality control)
- rule-based validation (deterministic gating)
- deferred correction (store first, let the user fix later)
- multi-producer agreement (several models propose; agreement boosts confidence)
- ensembled same-model agreement (same model called multiple times)

Consensus across independent producers is a special case, not the general pattern. See [chapter 07](07-reconciliation.md).

### 9. Prompts are auditable

When memory influences a prompt, that influence is visible. The user can see the final dynamic-memory block, can trace which stored items contributed to it, can see the approximate size and cost, and can suppress any item they do not want injected.

The reference implementation calls this surface the Payload Inspector and treats it as a peer of the Memory Inspector. See [chapter 10](10-transparency-mutability.md).

### 10. Context windows are a constraint, not a memory medium

Modern LLMs offer large context windows. That is not a reason to use them as memory. Treating the context window as storage produces the failure mode this architecture exists to fix: rising cost, diluted attention, lost-in-the-middle, and an unavoidable hard ceiling.

Instead, this architecture treats the context window as a **bounded-capacity workspace** that is rebuilt on every turn from:

- a small, stable block of static instructions
- a carefully activated block of typed memory (engrams, Meta-Vault, relevant associations)
- a compact block of salient digests standing in for verbatim history
- the current user message
- optional tool or attachment context

The total footprint stays below a chosen maximum threshold — much lower than the model's advertised context limit. If your implementation's prompt grows when the conversation does, something is wrong. Fix activation, budgeting, or digest generation before you fix anything else.

### 11. Biology is inspiration, not a claim

The terminology — engram, association, activation, reinforcement, priming, Hebbian boost — is chosen because the metaphors hold and they signal the right product behavior: durable traces, associative recall, recency, use-based reinforcement. They are not claims of biological equivalence, and a reader should not expect the system to replicate biological memory in any mechanistic sense.

## How to read these docs

- Every chapter is standalone. You can read them out of order if you know what you are looking for.
- Diagrams are inline Mermaid. They render on GitHub and in most Markdown viewers.
- Object shapes are shown as TypeScript-style type sketches because that syntax is widely legible. The sketches are illustrative; your actual fields will depend on your dev stack.
- Production specifics from the reference implementation are not published here. Where a concept has a tuned value in production — a threshold, a weight, a ratio, a timeout — this documentation describes the shape of the knob and leaves the value to the implementer.
- The glossary in [docs/glossary.md](glossary.md) is the single source of truth for terminology.

## What to read next

- If you want the mental model: [02-conceptual-foundation.md](02-conceptual-foundation.md).
- If you want to know what the system stores: [04-asset-taxonomy.md](04-asset-taxonomy.md).
- If you want to know how one turn flows end-to-end: [05-cognitive-cycle.md](05-cognitive-cycle.md).

---

## Academic references

One recent empirical source grounds the "lost-in-the-middle" failure mode that this chapter's *Context windows are a constraint, not a memory medium* principle reacts to. For the broader bibliography, see the [README's Academic references section](../README.md#academic-references).

- **Liu, N. F., Lin, K., Hewitt, J., Paranjape, A., Bevilacqua, M., Petroni, F., & Liang, P. (2024).** Lost in the Middle: How Language Models Use Long Contexts. *Transactions of the Association for Computational Linguistics,* 12, 157–173. DOI: [10.1162/tacl_a_00638](https://doi.org/10.1162/tacl_a_00638). — *Empirical evidence that LLMs disproportionately ignore information placed in the middle of long prompts; one of the four compounding costs of treating the context window as memory.*
