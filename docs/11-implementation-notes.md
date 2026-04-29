# 11. Implementation Notes

Practical guidance for teams adopting this architecture. This chapter is about **trade-offs**, not prescriptions. It assumes you have read the earlier chapters.

---

## Adoption paths

The right entry point depends on your product shape.

### Single-model chat app

You have one user and one assistant per turn.

**Required chapters:** [01](01-philosophy.md), [03](03-architecture.md), [04](04-asset-taxonomy.md), [05](05-cognitive-cycle.md), [06](06-extraction.md), [08](08-storage.md), [09](09-retrieval-activation.md), [10](10-transparency-mutability.md).

**Recommended reconciliation mode:** start with mode 4 (deferred correction) plus mode 3 (rule-based validation) as a prefilter. Add mode 2 (verifier pass) later if quality needs more help.

**What to skip first pass:** the consensus report asset and associated UI. You can add it later if you adopt multi-producer reconciliation.

**Minimum viable implementation:**

- One extractor LLM call per turn
- IndexedDB or equivalent local store
- Memory Inspector with the 4 required tabs (Engrams, Associations, Salient Digests, Meta-Vault) plus Activity
- Mutation: edit, delete, suppress, pin (even minimal forms)
- Activation with text-match + recency + utility; skip embeddings initially

### Multi-model parallel chat app

You have one user and N assistants responding to the same turn.

**Required chapters:** all of them.

**Recommended reconciliation mode:** mode 5 (multi-producer agreement) as primary, mode 3 (rule-based validation) as prefilter, mode 1 (human review) for low-confidence items.

**What to add over single-model:**

- Multi-producer extraction fan-out with a collector and bounded timeout
- Consensus report asset and its Memory Inspector tab
- Producer attribution ("N models agreed") on engrams, associations, salient digests

### Multi-agent system

You have one user and multiple named agents with distinct roles (planner, researcher, critic, tool agent).

**Required chapters:** all of them.

**Recommended reconciliation mode:** mode 5, but with agent-role awareness. Treat agents as producers for the purpose of agreement counting, but:

- track agent role in provenance
- separate observed facts from agent recommendations in the asset schema
- require review for memory created by autonomous or tool-using agents

**What to add:**

- Agent-role field in provenance
- Review gates for agent-authored memory
- Separate scopes for different agent namespaces if your product demands it

---

## Tuning knobs

Every place this architecture leaves a value unspecified is a knob you will tune. Here are the main ones, what they control, and how to think about them.

### Extraction

| Knob                          | Controls                                    | How to think about it                                                  |
| ----------------------------- | ------------------------------------------- | ---------------------------------------------------------------------- |
| Extractor model choice        | Quality vs cost per turn                    | Smaller models are often fine; verify with your own eval data          |
| Fallback model                | Robustness under primary failure            | Should be in a different family than the primary                       |
| Retry count                   | Cost ceiling per turn                       | 2–3 is typical; more is rarely worth the cost                          |
| Retry delay                   | Throughput vs backoff                       | Exponential; cap at a few seconds                                      |
| Extraction token limit        | Output size                                 | Tight limits force concise proposals; too tight and useful detail is lost |

### Reconciliation

| Knob                          | Controls                                    | How to think about it                                                  |
| ----------------------------- | ------------------------------------------- | ---------------------------------------------------------------------- |
| Similarity threshold (merge)  | How aggressively proposals are merged       | Lower = more merging, more risk of merging distinct concepts           |
| Contradiction threshold       | How different content must be to conflict   | Lower = more items flagged as contradictions; good for safety         |
| Auto-approve confidence floor | What moves to active without review         | Higher = more review queue work                                        |
| Multi-producer timeout        | How long to wait for all producers          | Trade latency for completeness                                         |
| Agreement-confidence adjustment | How much multi-producer agreement lifts confidence | Higher = agreement dominates; lower = single-producer confidence still matters |

### Activation

| Knob                          | Controls                                    | How to think about it                                                  |
| ----------------------------- | ------------------------------------------- | ---------------------------------------------------------------------- |
| Axis weights per query type   | What signals dominate in scoring            | Tune empirically; query classification is only as useful as the weights |
| Seed count                    | How local Hebbian boost stays               | More seeds = broader boost; fewer = more focused                       |
| Memory token budget           | Context-window fraction reserved for memory  | Trade context-window cost against memory coverage                      |
| Section ratios                | Engrams vs salient history vs verbatim       | Depends on whether your product cares more about depth or freshness    |
| Relevance floor               | Minimum score to include                    | Higher = cleaner context, more risk of missing relevant memory          |
| Decay rate                    | How quickly un-accessed items lose utility   | Gentle; over weeks, not minutes                                        |

### Storage

| Knob                          | Controls                                    | How to think about it                                                  |
| ----------------------------- | ------------------------------------------- | ---------------------------------------------------------------------- |
| Store size per vault          | When memory crowds the device/store         | Have a pruning strategy; archive before deleting                       |
| Sync batch size               | Network efficiency vs latency               | Cap at a reasonable upper bound                                        |
| Sync frequency                | Freshness across devices                    | Opportunistic; manual trigger always available                         |
| Retention for archived items  | When to hard-delete                         | Depends on product; be explicit in privacy policy                      |

---

## Cost considerations

A cognitive layer adds **latency and spend** to each turn. It also — and this is the point — **removes** the cost that would otherwise come from stacking transcripts into a growing context window. Net cost depends on conversation length: short conversations pay slightly more (the cost of running the extractor), long conversations pay dramatically less (because the prompt sent to the chat model does not grow with history).

Rough mental model for per-turn cost in a naive stacked-transcript app vs a cognitive-layer app, as conversation length grows:

```text
Naive app (stack transcripts):
  turn 1 cost:    1x chat
  turn 50 cost:   5x chat  (window largely full of prior turns)
  turn 200 cost:  truncation crisis or forced new conversation

Cognitive-layer app:
  turn 1 cost:    1x chat + 1x extractor
  turn 50 cost:   ~1x chat (bounded prompt) + 1x extractor
  turn 200 cost:  ~1x chat (bounded prompt) + 1x extractor
```

The crossover point — where the extractor's added cost is repaid by the chat model's bounded prompt — is typically reached within the first few dozen turns of a real conversation. Long-running conversations run at a small fraction of a naive app's cost.

### Added LLM calls per turn

In the minimum viable implementation:

- **+1 extraction call** (always)

With verifier reconciliation (mode 2):

- **+1 verifier call** per turn (or per proposal if you verify individually)

With multi-producer mode in a 3-model app:

- **+3 extraction calls** per turn

With ensembled same-model mode (N=3):

- **+3 extraction calls** per turn

### Budgeting spend

The extractor should be **cheaper** than the chat model. Typical pattern: a small, fast model at ~1/5th to 1/20th the cost of the chat model. Spend on extraction is still real — at scale, it is the difference between an affordable cognitive layer and a costly one.

The verifier, if used, is usually the same tier as the extractor.

### Budgeting latency

Extraction is async, so its latency does not directly affect chat UI. But:

- activation is inline and its latency budget is tight
- reconciliation blocking on slow extractions can delay when memory becomes visible
- users notice if memory appears "late" relative to the turn they just finished

Aim for: activation under ~100ms (excluding storage I/O), extraction-to-visibility under a few seconds.

### Storage growth

A modestly active user will generate:

- ~0 to 2 engrams per turn
- ~0 to 3 associations per turn
- ~1 salient digest per turn per producer
- ~0 to 1 Meta-Vault entry per session

Scale from there. A year of regular use per user easily accumulates thousands of engrams. Plan pruning and archiving.

---

## Common pitfalls

The specific mistakes teams make when implementing this architecture. Each one will cost you a week if you do not notice it.

### 1. Coupling extraction to chat latency

The most common failure. If chat waits on extraction, you have regressed. Always use a separate stream or background job, and verify with a stopwatch.

### 2. Missing provenance

Storing memory without "where did this come from" means the user can never trust or audit it. Provenance is not optional. Treat any asset without it as a bug.

### 3. Inflexible schema

Locking in specific field names early, then discovering you need new ones, is painful. Design your schema to tolerate unknown fields and gracefully extend. Version the schema.

### 4. No user-facing mutation path

Shipping a Memory Inspector that only shows, never lets you edit, is a trap. Users will try to correct something, fail, and lose trust. Implement mutation from Day 1, even minimally.

### 5. Treating the extraction prompt as permanent

Your first extraction prompt will be wrong. Budget for prompt iteration. Version the prompt, and record which version produced which extraction in the extraction log.

### 6. Single-score confidence

A single confidence number the user cannot interpret is worse than none. Expose contributors (extraction confidence, agreement, usage, explicit user approval).

### 7. Over-activating

Sending too much memory to the assistant degrades answers. The temptation is to cast a wide net; the discipline is to cast a narrow one.

### 8. Under-activating

Sending too little memory makes the assistant feel forgetful. The temptation is to trust the extractor's confidence; the discipline is to use all the axes and tune thresholds.

### 9. Never decaying

Memory that never loses utility becomes noise. Implement a gentle decay from the start, even if it is just "halve the utility score of any engram untouched for 30 days."

### 10. Skipping reconciliation

"I'll just store raw extractions and clean up later" is the fastest way to poison the store. Pick a reconciliation mode from Day 1, even if it is mode 4 (deferred correction) with only basic validation.

### 11. Silently suppressing memory

If the system decides to skip a user's memory, the user should be able to see that it was skipped and why. Invisible suppression destroys trust.

### 12. Making the chat UI a control panel

Resist the urge to render memory controls inline in the conversation. Inline hints are fine; inline controls distract from the conversation.

### 13. Forgetting that bounded context is the primary purpose

The most consequential pitfall. Teams build extraction, reconciliation, storage, and a Memory Inspector, then quietly stack verbatim turns into the prompt "just to be safe." The cognitive layer becomes a parallel decoration that the actual chat path ignores. Symptoms: prompt size climbs with conversation length, cost per turn climbs, and the Memory Inspector shows plenty of content that is never actually injected.

Mitigation: make bounded prompt size a production SLO with a dashboard. Treat any regression as a bug, not a tuning opportunity.

---

## Evaluation

How do you know your cognitive layer is working?

> **What this chapter does not publish.** The reference implementation includes an internal evaluation suite (for example, for compression quality). Its specific metrics and thresholds are not architectural. What follows is a menu of signals you should consider tracking, not a prescribed metric set.

### Turn-level signals

- **Memory recall.** Did the assistant recall a relevant stored fact when asked?
- **Memory accuracy.** Was the recalled fact correct?
- **Memory restraint.** Did the assistant inject memory only when useful, not on every turn?
- **Contradiction rate.** How often does the system surface contradicting memory?

### Context-window signals (the primary SLOs)

These are the metrics that confirm the architecture's primary purpose is being met.

- **Prompt size by turn index.** Plot the assembled prompt's token count against the turn number of a conversation. The curve should be **flat**. Any upward slope is a regression of the whole architecture.
- **Prompt size P50/P95.** The tail matters. If P50 is stable but P95 climbs, you have outlier conversations that are leaking unbounded history.
- **Fraction of prompt consumed by salient history vs verbatim history.** Salient history should dominate. If verbatim is growing, digest replacement is not happening.
- **Prompt-to-window ratio.** Average and worst-case prompt size as a fraction of the chat model's context window. This should sit well below 1.0 (typical targets: 0.2–0.5) and should not drift up over time.
- **Max conversation length before degradation.** Can users run conversations of arbitrary length without a quality cliff? If the answer is "only up to ~N turns," bounded context is not actually bounded.
- **Cost per turn by turn index.** Should be approximately flat, modulo extraction cost.

### Store-level signals

- **Growth rate.** Engrams per active user per week. Expect a curve that flattens, not one that keeps growing linearly.
- **Utility distribution.** What fraction of engrams have non-trivial utility scores? If nearly all are at zero, activation is too broad.
- **Mutation rate.** How often do users edit or suppress memory? High edit rate = useful; high suppress rate = the system is adding noise.
- **Activation selection rate.** What fraction of stored memory gets activated in a typical week? Very low = memory is dead; very high = activation is not being selective.

### User-level signals

- **Memory Inspector engagement.** Do users visit it at all? Do they return to it?
- **Approval rate in review queues.** If most pending items get approved, auto-approval is probably too strict. If most get rejected, extraction is too loose.
- **"Was this helpful?" on memory-injected responses.** Easy to collect; useful to compare memory-on vs memory-off turns.
- **Privacy signals.** Do users export their memory? Do they delete their vault? Both indicate they take it seriously.

### What to monitor for regressions

- Extraction success rate (from the extraction log).
- Reconciliation rejection rate.
- Activation latency P50 / P95.
- Sync failure rate (if sync is enabled).

---

## Decisions that will change over time

Early adopters should expect to revisit:

1. **Extractor model choice.** Cheaper, better small models appear regularly.
2. **Reconciliation thresholds.** The right similarity and confidence thresholds only emerge from real data.
3. **Activation weights per query type.** Query classification is only as useful as its tuning.
4. **Memory token budget.** Context windows are changing fast; the right budget today is not the right budget next year.
5. **Asset taxonomy.** You may discover your product needs a new asset type. Add it deliberately; do not crowd it into an existing one.

Treat all of these as **products in their own right**, with owners, metrics, and iteration loops.

---

## What this architecture will NOT solve for you

- **Hallucinations in the chat assistant.** A cognitive layer reduces them by bringing in stored facts, but the assistant can still hallucinate.
- **Bad extractors.** Garbage in, garbage in memory.
- **Privacy policy.** Storing memory is a privacy obligation. Write an explicit policy; do not let this architecture substitute for one.
- **Compliance.** GDPR/CCPA/sector-specific obligations are on you; local-first helps but does not automatically satisfy them.
- **Adversarial inputs.** Users who deliberately attempt to poison memory are a product and security problem this architecture does not address.
- **UX quality.** Good IA and visible affordances are everything. A well-architected cognitive layer wrapped in bad UI feels worse than no memory at all.

---

## What to read next

- [glossary.md](glossary.md) — the terminology reference
- Return to any chapter that affected a design decision you are working through.
