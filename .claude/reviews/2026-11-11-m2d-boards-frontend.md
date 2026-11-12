# Review — M2d boards & columns frontend (`feat/m2d-boards-frontend`)

## Reviewer report — round 1
**Verdict:** APPROVE
**Checks:** lint PASS, typecheck PASS, test PASS (shared 368, api 342, web 163)

### BLOCKER
- none

### MAJOR
- none

### MINOR
- `apps/web/src/stores/board.ts` `createColumn` (`sortByRank([...state.columns, column])`) — the server can rebalance every rank of the board while creating a column (`column.service.ts` create → `setRanks`), and only the new column comes back. The fresh rank then gets sorted against stale local ranks, so the new column can land in the wrong place (e.g. before an old last column whose rank was long, close to `z...`). Fix: the column is always appended (no afterId/beforeId), so push it to the end without re-sorting. If its rank is not greater than the current last rank (`rankFitsAt(list, list.length - 1)` is false), call `reloadColumns()`, the same way `moveColumn` does.
- `apps/web/src/stores/board.ts` `moveColumn` rollback (`set({ columns: previousOrder, ... })`) — the whole list snapshot taken before the request gets restored. Anything changed while the move was in flight (column added through `AddColumnForm`, column deleted, inline rename, settings saved) is lost locally, and a deleted column comes back. Fix: roll back only the moved column (put it back at `fromIndex` in the *current* list), or keep the other column mutations disabled while `isMovingColumn` is true.
- `apps/web/src/stores/board.ts` `renameBoard` — unlike `createBoard`/`loadBoards`, it has no `boardsGeneration` guard. After a logout and login as another user, a late response could write the previous user's `role` into a board that both users share. Fix: capture `boardsGeneration`/`boardGeneration` before the request and skip the `set` when they changed.
- `apps/web/src/stores/board.ts` rebalance detection in `moveColumn` is a heuristic. A rebalanced rank that happens to fall between the stale neighbours goes unnoticed. That is acceptable for now (the order on screen stays correct and later moves send ids only), but the comment should say it is best effort. The first MINOR item makes the consequence harmless.
- `apps/web/src/features/boards/MembersDialog.tsx` "Remove" member — runs right away with no confirmation, while board and column deletion and "Leave board" all ask first. Suggest reusing `ConfirmDialog`.
- Bundle grew from 537 to 711 kB (Vite chunk warning). Consider `lazy()` for `BoardPage` (dnd + dialogs) in `routes.tsx`. This can be a follow-up.

### Done well
- Move protocol matches the API contract: `getMoveNeighbors` always sends both `afterId`/`beforeId` (null at the ends), and no ranks are computed on the client. Ranks are compared with `<`/`>` (`compareRanks`, `rankFitsBetween`), never `localeCompare`, and the reason is documented.
- Optimistic move, rollback with a notice, reload on `INVALID_POSITION`/`NEIGHBOR_NOT_FOUND`/404, and `isMovingColumn` blocking concurrent drags (`isDragDisabled`) are all correct. `finally` clears the flag only for the current board.
- Stale responses: generation counters on every board-scoped action, plus a reset on `sessionEpoch` or user change through `useAuthStore.subscribe`. Tests cover both (`drops the result of a board opened earlier`, `clears board data on logout...`).
- Drag tests use the real `@hello-pangea/dnd` keyboard sensor (space / arrows / space on the handle). They check the optimistic order before the server answers, the exact request body, rollback on 503 and reload on 409. They are not shallow "renders" tests.
- Permission gating follows `hasBoardPermission`/`canManageMember`/`canLeaveBoard` from shared. RTL tests cover VIEWER, MEMBER, ADMIN and OWNER, and the gating matches the role matrix in the requirements.
- All responses are parsed with shared Zod schemas, form input goes through shared schemas, and URLs are encoded.
- Accessibility: labelled regions per column, drag handles with accessible names, `role="alert"` for errors, `aria-invalid`/`aria-describedby` on fields, native selects, Radix dialogs with titles and descriptions.
- No `any`, no eslint-disable, no privacy violations (no development-tool or AI mentions, no `docs/`/ADR references), and only `apps/web` was touched.
