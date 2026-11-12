# Review: M3a — cards backend (WF-CARD-01..04, ADR-020)

Branch: `feat/m3a-cards-backend` (uncommitted changes)

## Reviewer report — round 1
**Verdict:** APPROVE
**Checks:** lint PASS, typecheck PASS, test PASS (shared 479, api 472, web 219; DB-backed api tests ran, none skipped)

### BLOCKER
- none

### MAJOR
- none

### MINOR
- `apps/api/prisma/migrations/20261112014500_cards_and_board_keys/migration.sql` (backfill, `DELETE FROM "Board" ... NOT EXISTS member` and the `UPDATE "BoardMember" SET "role" = 'OWNER'`). Answer to the question in the brief: I do not count this as a defect, because those rows can only come from direct DB writes, the project has no production data, and the test covers it. Still, a migration that silently deletes data and silently grants OWNER (which includes `board:delete`) is a poor default. The fix is to fail fast instead: `RAISE EXCEPTION` with the offending board ids, so a person fixes the data and reruns. If you keep the repair, at least `RAISE NOTICE` the number of deleted boards and promoted memberships so the operator sees it in the migration log.
- `apps/api/src/services/card.service.ts:157-171`: `list` runs three independent queries in parallel on different connections with no shared snapshot. If a column is created and a card is moved into it between `listColumnIds` and `listForBoard`, that card is silently dropped (`byColumn.get(...)?.push`). Fix: run the reads in one `REPEATABLE READ` read-only transaction, or use a single query (`column.findMany({ include: { cards: { orderBy: rank } } })` plus the board key).
- `apps/api/prisma/schema.prisma` (`CardTransition`): there is no index on `fromColumnId` / `toColumnId`. `ON DELETE SET NULL` on column delete therefore does a sequential scan of the whole transition table, and it does so while holding the exclusive board lock. Fix: add `@@index([fromColumnId])` and `@@index([toColumnId])`. They are cheap and will also help the flow analytics queries in M5.
- `apps/api/src/repositories/card.repository.ts` (`delete`) and `apps/api/src/repositories/board-member.repository.ts` (`removeUnlessOwner`): these writes bypass `createBoardTransactionRunner`, so they get no `lock_timeout` and no 503 `BOARD_BUSY` mapping. If a card row is held by a slow move or rebalance, the request waits without a bound. Fix: run them through the runner (or set `lock_timeout` the same way) for consistency.
- `apps/api/src/services/card.service.ts:225-228`: `getByKey` reads the board key twice (`boardKeyOf` followed by `withKey`). Reuse the key you already have: `toCard(card, boardKey)`.
- `packages/shared/src/schemas/card.ts:24`: `cardDescriptionSchema` does not trim, so a whitespace-only description is stored as is. This is already listed as known debt. Consider `.trim()` and mapping `''` to `null`.
- `migration.sql` (the `replace(txt, ...)` block): the indentation is inconsistent with the rest of the `DO` block. Cosmetic only.

### Done well
- **Locking and deadlock freedom.** I checked every pair of paths and found no lock cycle:
  - The order is always board, then card, then column.
  - Create takes the exclusive board lock, which correctly avoids the `FOR SHARE` vs `UPDATE ... RETURNING` upgrade deadlock on the counter.
  - Moves hold at most one column lock (the target).
  - FK checks take `FOR KEY SHARE`, which is compatible with `FOR NO KEY UPDATE`.
  - Assign and member removal both lock in the order membership, then card.
  - PATCH and DELETE take no board lock and touch neither ranks nor membership counts, so they cannot form a cycle.
- **WIP is race-free.** Settings change only under the exclusive lock, so they stay stable while a move holds the shared lock. Create is serialized by the exclusive lock. Moves entering a column lock the column row before counting, and under READ COMMITTED the count statement sees the committed winner. Reordering inside a column skips the check. The parallel tests (move/move and create/move, queued behind a held board lock) really exercise the race.
- **Transition atomicity.** The transition is written in the same transaction as create and move, and only when the column changes. No-op moves and reorders write nothing. The injected-failure test proves the move and the transition roll back together.
- **Authorization across boards.** `lockCard`, `findColumn`, `update`, `updatePosition` and `delete` are all scoped by `boardId`. Neighbours resolve only within the target column, and the assignee is checked against the membership of this board. The role matrix test covers OWNER/ADMIN/MEMBER/VIEWER/OUTSIDER, and forbidden requests leave the data unchanged.
- **Member removal.** It deletes the membership first, then unassigns, in one transaction. Together with the `FOR SHARE` membership lock on assign, a card can never end up assigned to a non-member. The concurrent test proves it.
- **Card numbers.** Numbers come from an atomic counter inside the create transaction, so a rollback returns the number and a deleted card's number is never reused. Concurrent creates are tested.
- **Column refactor.** Extracting into `board-transaction.ts` and `ranked-write.ts` is behaviour-preserving: the column tests are unchanged and pass, and the re-exports keep existing imports working. `COLUMN_NOT_EMPTY` is enforced under the exclusive lock and backed by a `NO ACTION` FK. Board delete still cascades correctly with cards (tested).
- **Migration.** The rank column is hand-written with `COLLATE "C"` and a DEFERRABLE unique constraint, consistent with Column. The 224-character `translate()` maps match one-to-one. The backfill test runs the exact SQL block from the migration file inside a rolled-back transaction.
- **Shared code.** All validation goes through Zod schemas in `packages/shared` (strict objects, `refine` for "at least one field"). Ordering uses LexoRank only. There is no `any`, no references to `docs/` or ADR numbers, and no development-tooling mentions. The frontend changes are minimal and compile-only, as the brief required.

## Reviewer report — round 2
**Verdict:** APPROVE
**Checks:** lint PASS, typecheck PASS, test PASS (shared 484, api 478, web 219)

Scope: only the delta for the round-1 minor items.

### BLOCKER
- none

### MAJOR
- none

### MINOR
- `apps/api/src/services/card.service.ts` (`list`): `let board;` is declared without a type and relies on TypeScript inferring it from later assignments. Write `let board: BoardCards | null;` so the type is explicit, matching `let deleted: boolean;` in `delete`. Cosmetic only.
- `migration.sql`: the indexes and notices were added by editing the existing migration `20261112014500_cards_and_board_keys` in place. That is fine while the migration exists only on this unmerged branch. The second developer must reset or re-apply it if they ran an earlier version locally (Prisma will report a checksum mismatch). Once it is merged, never edit it again.
- The snapshot consistency of `listBoard` is covered only indirectly: the runner test asserts the isolation level, but no test interleaves a write with the list read. This is acceptable, because REPEATABLE READ gives that guarantee by construction.

### Round-1 items: status
1. List snapshot: **resolved.** `listBoard` reads the key, columns and cards in one REPEATABLE READ transaction through the runner. The snapshot is taken at the first statement (`set_config`), so all three reads share it. Busy maps to 503 and a missing board to 404.
2. CardTransition indexes: **resolved.** `@@index([fromColumnId])` and `@@index([toColumnId])` are in the schema and in the migration SQL.
3. Lock timeout on delete and member removal: **resolved.** Both go through `createBoardTransactionRunner` (rank constraint `null` for members), and `BoardBusyError` maps to 503 `BOARD_BUSY` in both services. A route test checks 503 and that the card is kept while its row is locked.
4. `getByKey` double read: **resolved.** The key is read once and reused.
5. Description trimming: **resolved.** The schema trims, applies the limit after trimming, and stores blank as `null`. Tested on both create and PATCH.
6. Backfill: **resolved as I suggested.** The repair is kept, `GET DIAGNOSTICS` + `RAISE NOTICE` reports the deleted and promoted counts, and the indentation is fixed.

### Done well
- The runner extension is small and type-safe: per-call `isolationLevel`, and an optional rank constraint. It reuses the same timeout and error-mapping path instead of copying it.
- The new `board-transaction.test.ts` checks the actual isolation level and that `lock_timeout` stays local to the transaction, against the real database.
