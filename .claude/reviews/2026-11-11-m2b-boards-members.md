# Review — M2b boards & members (branch `feat/m2b-boards-members`)

## Reviewer report — round 1
**Verdict:** APPROVE
**Checks:** lint PASS, typecheck PASS, test PASS (shared 316, api 193, web 77 — rerun by reviewer, API tests ran against the real test DB)

Extra check: `prisma migrate diff --from-config-datasource --to-schema prisma/schema.prisma` → empty migration, so the hand-written partial index `BoardMember_one_owner_per_board` causes no drift and will not be dropped by a future `prisma migrate dev`.

### Matrix verification (REQUIREMENTS section 2, cell by cell)
| Row | Implementation | Test |
|---|---|---|
| View board (all roles) | `GET /:boardId`, `GET /:boardId/members` → `board:view` = EVERYONE | matrix rows 1, 4 |
| Members management, OWNER/ADMIN, not on OWNER | `members:manage` = MANAGERS; OWNER target protected by conditional `role <> OWNER` update/delete → 403; OWNER not assignable (Zod `assignableBoardRoleSchema` → 400) | matrix rows 5–7, `forbids %s to change the OWNER's role`, `forbids an ADMIN to remove the OWNER`, `never assigns OWNER`, `rejects OWNER as the new role` |
| Rename board OWNER/ADMIN | `board:rename` = MANAGERS | matrix row 2 |
| Delete board OWNER only | `board:delete` = OWNER_ONLY | matrix row 3 |
| Transfer ownership OWNER only | `board:transfer` = OWNER_ONLY (endpoint out of scope) | shared permissions test |
| Cards/columns/AI/column settings | `card:write`/`column:write`/`ai:use` = CONTRIBUTORS, `column:settings` = MANAGERS — match the matrix, ready for M3+ | shared permissions test |
| Outsider → 404, member without role → 403 | `BoardAccessService.require` (404 before 403, role checked before body validation) | every matrix row has OUTSIDER = 404; `checks the role before validating the body`; nonexistent vs foreign board give identical response |
| Creator becomes OWNER, exactly one OWNER | nested create (single transaction) + partial unique index | `creates a board owned by the creator`, `database invariants` |
| Every member except OWNER may leave | `DELETE /members/:self` guarded by `board:view`, `canLeaveBoard` → 409 `OWNER_CANNOT_LEAVE` | `lets %s leave the board`, `does not let the OWNER leave` |

All cells match. Coder claims verified against the diff.

### BLOCKER
- none

### MAJOR
- none

### MINOR
- `apps/api/prisma/schema.prisma:61-63` — the doc comment says "Each board has exactly one OWNER, which is additionally enforced by the partial unique index". The index only enforces *at most* one; "at least one" is the service's responsibility (the migration comment says this correctly). → Reword to match the migration comment.
- `packages/shared/src/permissions.ts:56` / `apps/api/src/services/board.service.ts` — `canManageMember` is exported and tested but not used by the API; the "not the OWNER" rule lives separately in the repository's `role: { not: 'OWNER' }` filter. Acceptable (the SQL condition is the race-safe source of truth and the helper is useful for the web client), but add a one-line note on `canManageMember` that the API enforces the same rule atomically in the repository, so the two are kept in sync.
- `apps/api/src/middleware/require-board-permission.ts` / `board.service.ts` — known TOCTOU: a member demoted (or removed) between the middleware check and the mutation can still complete one in-flight request. Low impact and already flagged by the coder; keep it in the debt list (fix later by doing check + mutation in one transaction, or a conditional mutation that joins on the actor's current role).
- `apps/api/src/controllers/board.controller.ts:18-24` — `:userId` route param is only checked for non-emptiness, not via a Zod schema. Harmless (Prisma parametrizes, unknown ids → 404), optional for consistency with "validation only via Zod".

### Done well
- Clean, reusable authorization: `requireBoardPermission(action, { param })` + `getBoardMembership(res)` with typed `res.locals`, plus a defensive re-check in the service. Ready for nested column/card routers (`mergeParams` documented).
- Race safety where it matters: conditional `updateMany`/`deleteMany` on `role <> OWNER`, P2002 → 409 `ALREADY_MEMBER` for concurrent adds (tested with parallel requests), P2003 → 404 for a board deleted mid-request, DB-level single-OWNER invariant.
- Non-disclosure done right: 404 for outsiders on every board route, role check before body validation, same response for nonexistent and foreign boards, odd ids (long, unicode, SQL-like) tested.
- Excellent matrix test: data-driven 7 routes x 5 actors, plus a state-unchanged-after-403 check and auth checks for every route.
- Layering respected: Prisma only in repositories (new `prisma-errors.ts` helper also refactored into `user.repository.ts`), Zod schemas and types in `packages/shared`, no socket/column/LexoRank scope creep.
- Strict TS, no `any`; exhaustive `switch` over the add result union.
- Privacy: no mentions of development tools, `docs/` or ADR numbers in any changed/new file; the "AI assistant" wording in `permissions.ts` refers to the product feature (allowed).
