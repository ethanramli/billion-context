# DESIGN - Reliable recall of important details

- Task ID: `2026-11-16_important-detail-recall`
- Home Repo: `billion-context`
- Created: 2026-11-16
- Status: Draft

## 1. Goals & Non-Goals

- **Goals**: Let the agent retrieve a small, meaningful detail from an earlier conversation days later, with evidence and provenance. Keep ordinary requests lean by retrieving only a few relevant items.
- **Non-Goals**: Replace context compression, retain every raw conversation indefinitely, or answer from unsupported summaries alone.

## 2. Background & Motivation

The product promise should be useful recall, not a large token counter. A memory should survive compression because the system can return to its source, not because every detail is copied into a growing prompt.

## 3. Current Architecture (as-is)

- `search_context` manually searches compressed block topics/summaries; it can additionally search folded original text when a resident session still has that source.
- Historical-session search accepts a `conversation_id`. A persisted session without a resident `Session` degrades to summary-only search because the folded-original provider is unavailable there.
- The search engine already exposes BM25, fuzzy, hybrid, and optional host-supplied semantic algorithms. The current `executeSearchContext` path deliberately uses exact substring matching for predictable no-match behavior.
- `src/agent/conversation-recall.ts` stores a bounded, ten-minute lane-to-session mapping for retries. It is not a durable memory index.

## 4. Proposed Design (to-be)

1. **Create evidence-backed memory records.** Store a concise candidate fact plus the original message ref, exact supporting text, timestamp, profile/session scope, and a status such as current, superseded, or user-pinned. Do not discard source text when summarizing it.
2. **Separate importance from recency.** Rank candidates using query relevance first, then importance and recency. Explicit user requests to remember something outrank inferred salience. Keep extraction and ranking explainable enough to inspect.
3. **Retrieve before answering.** Use the user’s current question to query the memory index across the allowed scope. Return only a small number of evidence snippets. Prefer existing exact/hybrid search first; add semantic retrieval only if the benchmark shows a clear miss class.
4. **Keep evidence low-privilege.** Present retrieved text as quoted historical evidence with date and source. It must never be inserted as a system/developer instruction. If evidence conflicts, surface both dates and prefer the corrected/latest value only when the correction is explicit.
5. **Make recall honest and controllable.** If no adequate source is found, say so. Let the user inspect, correct, pin, or forget saved memories. Cross-session scope stays opt-in and profile-isolated.
6. **Prove it with delayed-recall evaluation.** Build conversations containing held-out facts, irrelevant detail, later corrections, paraphrased queries, ambiguous queries, and adversarial quoted text. Measure precision/recall, false-memory rate, correction handling, cross-profile leakage, token cost, and latency.

### Initial implementation sequence

1. Build a small local benchmark and identify where current `search_context` fails.
2. Add a provenance-preserving memory record/index behind an off-by-default flag.
3. Add bounded retrieval and user controls.
4. Compare against the baseline before enabling automatic recall by default.

## 5. Alternatives Considered

| Option | Pros | Cons | Decision |
|--------|------|------|----------|
| Only increase context window | Simple mental model | Still does not guarantee a small detail survives summarization or gets found | Reject |
| Rely on manual `search_context` only | Reuses existing code; low cost | Depends on the model deciding to search and knowing which session to search | Keep as fallback, insufficient alone |
| Store every transcript verbatim and inject it | Easy to explain | High token cost, poor privacy/latency, repeats the original context-growth problem | Reject |
| Evidence-backed memory index with bounded retrieval | Retains source, limits prompt cost, supports explicit uncertainty | Requires extraction, privacy boundaries, and evaluation | Proposed |

## 6. Risks & Trade-offs

- **Backward compatibility**: Additive and initially off by default.
- **Performance**: Keep retrieval bounded; do not re-embed every transcript on every turn.
- **Cross-platform**: Use the existing Node/storage abstractions; avoid OS-specific memory paths.
- **Security**: Memory text is user-controlled history and must stay untrusted. Indexes and references must be scoped so one profile/session cannot read another's private content.

## 7. Open Questions

- Should the first version save only user-pinned facts, or also infer important facts automatically?
- Should cross-session memory be scoped per agent profile, per project, or both?
- What is the retention/forget policy for source text and derived memories?
