---
date: 2026-07-15
project: orchestrator / orchestrator-ui-v1
tags: [session-log]
---

# 2026-07-15 — Tenant storage metric + chat-agents cursor-table crash

## What was done

### 1. Tenant "Storage Used" metric — analysis + implementation
- Backend feature at [[tenant-service]] (`orchestrator/libs/shared-modules/src/shared/tenants/services/tenant.service.ts`), method `getStorageUsedByTenantIds`, shipped same-day in commit `238272fb4e` ("storage BE").
- Data model reviewed: `KbDocument.gcsBytes` (raw file in GCS bucket, files only, 0 for websites), `extractedTextBytes` (parsed text, all source types), `contentBytes` (post-chunking, confirmed by user as **not relevant** to this metric).
- Found the original formula (`SUM(extractedTextBytes)` only) undercounted file uploads — it ignored the actual uploaded file bytes entirely — and was inconsistent with the pre-existing [[kb-source-storage-breakdown]] feature (`kb-source.service.ts#getStorageBreakdown`, which sums `gcs+content+vector` and explicitly excludes `extractedTextBytes`).
- **Decision (user-confirmed):** storage should reflect "what the user actually uploaded or parsed." Formula changed to a per-document effective size — `gcsBytes` when present (file sources), else `extractedTextBytes` (website/raw-text sources) — summed per tenant. Avoids double-counting since the two fields are mutually exclusive by source type.
- Implemented as `SUM(CASE WHEN d.gcs_bytes > 0 THEN d.gcs_bytes ELSE d.extracted_text_bytes END)` in a raw SQL query against `kb_documents` (via `KbPrismaService`, no caching — read fresh each call, unlike the cached `taskConsumed` metric on the same service).

### 2. Runtime bug: `operator does not exist: bigint = jsonb`
- Hit when loading the tenant list after the above change went live.
- Root cause: `Prisma.join(tenantIds)` inside an `IN (...)` clause with `bigint[]` params — a known Prisma raw-query param-typing gap where BigInt values inside a joined IN-list can get bound as `jsonb` instead of `bigint`. The sibling `kb-source.service.ts` queries never hit this because they use scalar `=` comparisons, not an array `IN`.
- Fix: explicit `::bigint` casts on `orgId` and on each element of the tenant-id list (built via `Prisma.sql` per element instead of a bare `Prisma.join`).

### 3. Fixed a batch of unrelated pre-existing typecheck errors (orchestrator)
Surfaced by `npm run typecheck`, not caused by the storage work:
- `tools.node.ts:253` — `creditService.addAgentTask(...)` for KB search wasn't destructuring its return, leaving `billableCount` undefined downstream. Added the destructure.
- `execution.orchestrator.ts` — `countVisibleAssistantReply` called `this.agentTaskService.checkAndAddAgentTask(...)` but `AgentTaskService` was never injected into the constructor (already a registered provider in `langgraph.module.ts`, just not wired). Added the injection.
- `execution.orchestrator.spec.ts` / `tools.node.spec.ts` — test mocks/destructures didn't match: missing `agentTaskService` mock in `buildOrchestrator()`, and two KB-search tests in `tools.node.spec.ts` referenced a nonexistent `agentTaskService` (copy-paste from the orchestrator spec) when `ToolsNode` actually uses `creditService.addAgentTask`. Corrected constructor args and assertions.
- `tenant.service.spec.ts` — constructor now takes `kbPrismaService` as 2nd param (from the storage feature); test wasn't passing it. Added mock.
- Noted but **not fixed**: a large separate set of pre-existing typecheck errors from what looks like Prisma client drift (missing models — `creditBalance`, `notificationLog`, `aiModelCredit`, `NotificationTrigger` — and a missing `extractedTextBytes` field on `KbDocument` in generated types). Likely needs `prisma generate`. User hasn't confirmed whether to chase this yet.

### 4. Chat-agents Evaluation/Activity tab — intermittent crash (reload fixes it)
- Route: `orchestrator-ui-v1/apps/ipaas/app/(builder)/chat-agents/[id]/page.tsx`. Both tabs (`packages/chat-agent/.../evaluations-list.tsx`, `packages/agent/.../activity.tsx`) share [[use-table-module]] (`packages/table/src/hooks/use-table-module.ts`) with `paginationType: "cursor"`.
- Confirmed mechanism: unguarded reads of `meta.cursorConfig.order` (an array) on raw, unvalidated API response data in three spots in `use-table-module.ts` — throws if a response's `cursorConfig` is present but missing/malformed `order`. Fixed defensively (`Array.isArray` guards on all three access points, lines ~372, ~411).
- **Not yet resolved:** user reported this does NOT happen on `tool-flow-activity.tsx`, a third production consumer of the same cursor-pagination hook — meaning the defensive guard fixes the crash mechanism but not the actual trigger, which must be something upstream/chat-agents-specific.
- Leading unconfirmed hypothesis: `activity.tsx` has a `setInterval` polling loop (every 5s while agent status is ACTIVE) with a stale-closure dependency array (`[agentId]` only, doesn't include `refetch`/`agentData`) — could race with a user's pagination click and produce a mismatched/malformed `cursorConfig` in the response the pagination logic reads. `tool-flow-activity.tsx` has no equivalent polling (not 100% confirmed).
- Also unconfirmed: whether `evaluations-list.tsx` has an analogous trigger via the recently-added `useGenerateInsights` optimistic-update logic in `chat-agent-provider.tsx` (commit `c277df69`, same day, touches the Evaluations area).
- A research agent was launched to compare all three cursor-pagination consumers (backend cursorConfig construction, polling/invalidation triggers, query-key structure) but was **killed by the user before finishing** — not yet re-run.

## Pending items
- [ ] Re-run (or resume) the comparison investigation between chat-agents Activity/Evaluations and `tool-flow-activity.tsx` to confirm the actual trigger (polling race vs. mutation-invalidation race vs. backend response shape) — the defensive guard is a mitigation, not a confirmed root-cause fix.
- [ ] Consider adding a route-scoped `error.tsx` under `chat-agents/[id]/` so a future uncaught error in one tab doesn't tear down the whole builder page (no route-scoped boundary exists today; falls through to the app-root `error.tsx`).
- [ ] Decide whether to chase the pre-existing Prisma-client-drift typecheck errors (likely `prisma generate`) — flagged to user, no decision yet.
- [ ] No test coverage exists yet for `getStorageUsedByTenantIds` / `storageUsed` field in `tenant.service.spec.ts` (unlike `taskConsumed`, which has assertions) — worth adding given it just changed formula twice in one day.

## Related
- [[tenant-service]]
- [[kb-source-storage-breakdown]]
- [[use-table-module]]
- [[chat-agents-cursor-pagination-bug]]
