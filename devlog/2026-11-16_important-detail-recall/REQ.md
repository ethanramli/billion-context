# REQ - Reliable recall of important details

- Task ID: `2026-11-16_important-detail-recall`
- Home Repo: `billion-context`
- Created: 2026-11-16
- Status: Draft
- Priority: P2
- Owner: Ethan
- References: https://github.com/ranxianglei/billion-context

## 1. Background & Problem Statement

- **Context**: A very long conversation can fit through compression while still losing the practical ability to retrieve a small but important user detail later.
- **Current behavior (symptom)**: `search_context` can search block summaries and, when originals are available, folded original text. It is model-invoked and scoped to the current session by default; a historical session must be named explicitly. `conversation-recall.ts` is a short-lived retry/session-identity bridge, not user-memory recall.
- **Expected behavior**: When a user asks about a detail from days earlier, the agent finds the supporting original statement, reports when and where it came from, and does not invent an answer when evidence is missing or ambiguous.
- **Impact**: Preserves useful personal and task context without keeping the full transcript in every model request.

## 2. Reproduction (if applicable)

- **Environment**: Design task; no runtime reproduction yet.
- **Minimal reproduction steps**:
  1. Put a small specific fact in a long conversation, then allow many turns and compression to occur.
  2. Ask a natural-language question about that fact days later, without supplying its exact wording or session id.
  3. Check whether the answer is grounded in the original statement and cites its source.
- **Relevant configuration**: Existing session persistence and `search_context` behavior.

## 3. Constraints & Non-Goals

- **Constraints**:
  - Preserve source message, timestamp, and session provenance for every recalled fact.
  - Retrieved conversation text is untrusted data. Never promote it into a system/developer instruction.
  - Keep memory local to the user's configured profile by default; prevent cross-profile or cross-session leakage.
  - Retrieval must be bounded in latency and injected token count.
- **Non-Goals**:
  - Putting the entire transcript back into the model context.
  - Training or fine-tuning a model.
  - Claiming perfect recall when the source was deleted, never stored, or cannot be searched.

## 4. Acceptance Criteria (must be testable)

- **Correctness**:
  - [ ] A delayed question retrieves a held-out exact detail even when that detail is absent from the compression summary.
  - [ ] Every returned memory includes its original evidence and timestamp/session provenance.
  - [ ] A missing or conflicting detail yields an uncertainty/no-evidence result rather than a fabricated fact.
  - [ ] A correction supersedes the old value without deleting its history or presenting it as current.
  - [ ] Adversarial text in a recalled message remains data and cannot become a privileged instruction.
- **Performance / Stability**:
  - [ ] Index and retrieval storage are bounded and survive restart.
  - [ ] Cross-session retrieval is opt-in and isolated by profile.
  - [ ] A benchmark records precision, recall, false-memory rate, token overhead, and latency.
- **Regression**:
  - [ ] Existing compression and `search_context` behavior remains available.
  - [ ] New/modified test cases are added during implementation; none run for this design-only draft.

## 5. Proposed Approach

- **Affected modules & entry files**:
  - `src/decompress-shared.ts` and the search tool path: reuse current search rather than build a competing index.
  - `src/persist.ts` / `src/storage/`: durable, profile-scoped memory records with source refs.
  - `src/server/` or the host/plugin retrieval path: bounded pre-answer recall when a query points to prior context.
  - `benchmarks/`: delayed-recall cases with distractors and known answers.
- **Risks**: Automatic extraction may omit facts or over-save sensitive details; retrieval may surface stale values or prompt-injection text. Evidence, timestamps, correction handling, isolation, and user-visible controls are required.
- **Rollback strategy**: Gate the feature off by default until its benchmark and isolation checks pass; retain existing `search_context` as the fallback.
