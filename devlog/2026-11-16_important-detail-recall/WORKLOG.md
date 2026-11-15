# WORKLOG - Reliable recall of important details

- **Task ID**: `2026-11-16_important-detail-recall`
- **Branch**: `2026-11-16_important-detail-recall`
- **Started**: 2026-11-16
- **Status**: Design draft

## Summary

Mapped the existing context search and session-identity recall paths. Proposed an evidence-backed, profile-scoped memory index and a delayed-recall benchmark. No implementation or tests have been run.

## Key files

- `src/decompress-shared.ts` — existing current/cross-session `search_context` behavior.
- `src/agent/conversation-recall.ts` — short-lived retry continuity, not durable user memory.
- `kernel/src/search/` — current lexical/hybrid search algorithms and optional host-supplied semantic search.

## Validation

- Read-only source inspection only.
- No tests run; this branch contains design documentation only.
