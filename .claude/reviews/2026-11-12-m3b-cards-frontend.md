# Review: M3b — cards frontend (`feat/m3b-cards-frontend`)

## Reviewer report — round 1
**Verdict:** CHANGES_REQUESTED
**Checks:** lint PASS, typecheck PASS, test PASS (shared 484, web 313, api 478)

### BLOCKER
- None.

### MAJOR
- `apps/web/src/features/cards/CardDetailsDialog.tsx:213-223` with `apps/web/src/stores/board.ts:567-587`: the due date is saved on every `onChange` of `<input type="date">`, and `updateCard` does nothing about request order. In Chromium, typing the year with the keyboard fires `input` once per digit as soon as all segments are filled. That sends `0002-…`, `0020-…`, `0202-…` and `2026-…` as 4 parallel PATCHes (`z.iso.date()` accepts all of them). The responses, and the server writes, can land in any order. A stale response then overwrites the newer optimistic value through `replaceCard(cards, card)`, and the wrong date can end up stored. The same race hits rapid changes of any select (priority/assignee). A failed older request also rolls the field back over a newer successful one, because `restoreFields` restores `previous`. Fix:
  1. Save the due date on blur / Enter (commit pattern like `TitleField`), not on every change.
  2. In `updateCard`, keep a per-card (or per card+field) sequence number. Apply a success response or a rollback only if it belongs to the latest update of those fields.
  3. Add a store test with two overlapping updates where responses resolve in reverse order.
- `apps/web/src/stores/board.ts:485-499, 545-555` with `apps/web/src/features/cards/cardErrorMessage.ts:12-13`: a column deleted by another user (`404 COLUMN_NOT_FOUND`) is not handled.
  - `createCard` never reloads, yet the composer shows "This column no longer exists. The cards have been refreshed.", which is not true.
  - `moveCard` reloads only the cards. The deleted column stays on screen (empty), so every further drop or create there fails again.
  - Fix: on `COLUMN_NOT_FOUND`, in both `createCard` (catch, then rethrow) and `moveCard`, also call `reloadColumns()` (and `reloadCards()`). Keep the message truthful. Add a test for both paths.

### MINOR
- `apps/web/src/stores/board.ts:485-595`: only `moveCard` invalidates a list request already in flight. `createCard`, `updateCard` and `deleteCard` do not. A list request started earlier can therefore make a just-created card disappear (the composer is usable while cards still show skeletons on first load), or bring a just-deleted card back. Simply bumping `cardsGeneration` would leave `cardsStatus` stuck on `'loading'` during the first load. Instead, re-run `loadCards` after the mutation when a list request is in flight, or merge the mutation into the late result.
- `apps/web/src/stores/board.ts:579-586`: `updateCard` failing with 404 (the card was deleted elsewhere) shows the toast, but the card stays on the board. Remove it locally, or call `reloadCards()`.
- `apps/web/src/features/cards/CardDetailsRoute.tsx:235-237`: `close` pushes `boardPath` as a new history entry, so browser Back reopens the modal you just closed. Consider `navigate(boardPath(boardId), { replace: true })` when the modal was opened from the board, or `navigate(-1)` when there is an in-app history entry.
- `apps/web/src/stores/board.ts:241-245` with `apps/web/src/features/boards/BoardPage.tsx:185-192`: on the first load, a failed card load is reported twice, as a notice toast and as the inline "Could not load the cards." banner. Set the notice only when cards were already `'ready'`.
- `apps/web/src/features/cards/CardDetailsDialog.tsx:224-227`: for a VIEWER, `<Label htmlFor>` points to a `<p>`, which is not labelable. Use plain text in `dt`, or `aria-labelledby`.
- `apps/web/src/features/boards/BoardColumn.tsx:181-221`: no dashed placeholder at the drop target. DESIGN §3 asks for "przerywany prostokąt". The tinted list is an acceptable interim solution, but it is a design deviation and should be agreed or recorded.
- `apps/web/src/components/ui/checkbox.tsx:11`: `rounded-[4px]` → `rounded-md` has the same value (4px) and is unrelated to the task. Revert it, or mention it in the commit as a tidy-up.
- `apps/web/src/features/boards/BoardColumn.tsx:243`: a card created while a filter is active may not match the filter and disappears from view right away. Consider a short hint, or briefly highlighting the new card regardless of the filter.

### Done well
- The store protocol is careful:
  - Moves send both `afterId`/`beforeId`, and only one move runs at a time (`isMovingCard`).
  - Rollback restores only the moved card (`restoreCardPosition` keeps edits made meanwhile).
  - After `INVALID_POSITION`/404 the cards reload.
  - Rebalance is detected with strict `<` through `rankFitsAt` (never `localeCompare`), and `insertCardByRank` does the same.
  - A list requested before a move is dropped.
  - Board and session generation guards cover every card request.
- All input goes through the shared Zod schemas (`createCardSchema`, `updateCardSchema`, `cardKeySchema`). Only changed fields are sent.
- The card route handles a card not yet in the store (fetch by key after cards settle), a malformed key without a request, a deleted card, and retry on errors.
- Escape on a dirty field discards instead of closing the dialog.
- Permission gating is consistent: VIEWER gets no "+", no drag, read-only details and no Delete. Copy link stays available. Tests cover this.
- Tests check behaviour, not just rendering:
  - keyboard drag within and between columns with a fake layout;
  - rollback toast, overlapping reloads, locale-independent rank comparison;
  - partial rollback on `ASSIGNEE_NOT_MEMBER`;
  - `COLUMN_NOT_EMPTY` and `WIP_LIMIT_EXCEEDED` messages.
- Design tokens are used throughout: priority colours via `text-success/warning/danger/destructive`, WIP highlight `border-danger bg-danger-subtle`, `shadow-card`/`shadow-raised`, 2-line clamp, 24 px avatar.
- No `any`. No mentions of development tooling, `docs/` or ADR numbers in tracked files.

## Reviewer report — round 2
**Verdict:** APPROVE
**Checks:** lint PASS, typecheck PASS, test PASS (shared 484, web 326, api 478)

### Round-1 items
- MAJOR 1 (out-of-order edits, due date on every keystroke): fixed.
  - `DueDateField` saves only on Enter or blur. Escape discards via the dirty attribute.
  - `updateCard` numbers every edit per `<cardId>:<field>`.
  - `mergeEditResponse` keeps local values for fields that have a newer edit, or an edit still in flight. Column and rank stay local.
  - A rollback restores only the fields whose latest edit failed.
  - Covered by tests: responses in reverse order, an older failure after a newer success, another field still in flight, and due date typing/Escape.
- MAJOR 2 (column deleted elsewhere): fixed. `isColumnGoneError` makes `createCard` set the notice, reload columns and cards, and rethrow. `moveCard` also reloads columns. The message is now accurate. Store and UI tests added.
- Minors:
  - Fixed: list reread when a load was in flight (`pendingCardsLoad` + `reloadIfListInFlight`; on first load the status correctly ends `ready`), 404 on edit removes the card, close goes back / replaces history, no double report of a failed first load, plain `dt` for read-only values, checkbox reverted, hint for a card hidden by the filter.
  - Accepted as an agreed design deviation: the tinted drop target.

### BLOCKER
- None.

### MAJOR
- None.

### MINOR
- `apps/web/src/stores/board.ts` (`updateCard` catch): if two overlapping edits of the same field both fail, the screen and the server can disagree. Example: A sets title=X, then B sets title=Y.
  - B fails first. It is the latest edit, so the title goes back to B's snapshot, which is X (A's optimistic value).
  - A fails next. It is no longer the latest, so nothing is restored.
  - Result: X stays on screen while the server still has the original title. This is rare (two consecutive failures). A possible fix: when a non-latest edit fails and no newer edit of that field is still in flight or has succeeded, restore the field, or simply `reloadCards()`.
- `apps/web/src/stores/board.ts`: `latestCardEdits` is never pruned, not even on board change or reset. The growth is negligible (one entry per edited card field), but it could be cleared in `resetBoardStore` / `forgetOpenedBoard`. The sequence numbers only ever increase, so clearing is safe.
- `apps/web/src/features/cards/CardDetailsRoute.tsx:61-66`: there is no test for the close behaviour (`navigate(-1)` when opened from the board, `replace` for a direct link). Add one RTL test with a memory router that checks the history entries.

### Done well
- The sequence approach is clean, well commented and type-safe. Server timestamps are merged sensibly (`updatedAt` keeps the newer one).
- The fix for lists already in flight avoids the `'loading'` deadlock flagged in round 1.
- The new tests target exactly the races that were reported, not only the happy path.
- There is no `any`, and no tracked file mentions development tooling, `docs/` or ADR numbers.
