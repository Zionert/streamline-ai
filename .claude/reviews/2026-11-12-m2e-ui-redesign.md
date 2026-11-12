# Review: M2e UI redesign of apps/web (`feat/m2d-boards-frontend`, uncommitted)

## Reviewer report, round 1
**Verdict:** CHANGES_REQUESTED
**Checks:** lint PASS, typecheck PASS, test PASS (web 21 files, 212 tests), build PASS (825.87 kB JS, Vite >500 kB chunk warning)

Scope: I reviewed only the M2e presentation changes. The M2d work in the same tree was approved earlier. `apps/api/package.json` was ignored as requested.

Store/API logic check: `stores/board.ts`, `stores/board.test.ts`, `boardsApi.ts`, `columnsApi.ts`, `columnOrder.ts`, `columnDrag.ts`, `columnSettings.ts` and `boardErrorMessage.ts` all have mtimes from 23:41–23:59, which is before M2e started (about 00:28). Their content matches the M2d review with its follow-ups applied (`createColumn` appends and reloads, best-effort comment in `moveColumn`). `renameColumn` already rethrew after the rollback, so the new toast in `BoardColumn.handleRename` is presentation only. Nothing outside `apps/web` changed, apart from `pnpm-lock.yaml` for the 3 new dependencies.

### BLOCKER
- none

### MAJOR
1. **(blocking)** `apps/web/src/features/boards/BoardColumn.tsx:72-101`: the column header is the drag handle. `dragHandleProps` carries `role="button"` (confirmed in `@hello-pangea/dnd` dist, line 6378), and the element also gets `aria-label="Drag column X"`. In the accessibility tree a button's children are presentational and its name comes from `aria-label`. For every user with `column:write` (MEMBER and up), screen readers therefore lose:
   - the `<h2>` column heading (heading navigation no longer works);
   - the category (`CategoryDot` sr-only text);
   - the card counter and the **"limit exceeded"** state (`ColumnCounter`). This is the main §2 signal, and it is now visual only.
   - the strict-limit marker.

   While renaming, the `<input>` of `ColumnTitleEditor` also sits inside `role="button"`, which is an interactive element nested in a button. RTL tests do not model presentational children, so `columnTitles()` and `getByLabelText('WIP 3 of 2, limit exceeded')` pass even though real assistive tech never reaches these elements. This breaks DESIGN §9 / WNF-05 (WAI-ARIA conformance). It is a regression introduced by putting the heading inside the handle.

   **Fix (any one of these):**
   - (a) Spread `dragHandleProps` on the header but override the role after the spread (e.g. `role="group"` with `aria-roledescription="draggable column"`). Replace `aria-label` with `aria-labelledby` pointing at the `<h2>` id. Children then stay exposed, and focus plus the space/arrow keys still work.
   - (b) Keep the header as the mouse drag area, but put the `<h2>`, counter and lock outside the element that carries `role="button"`.

   In both cases, render `ColumnTitleEditor` outside the handle element (or drop the handle props) while `isEditingTitle` is true. Add an assertion that cannot pass with presentational children, e.g. the handle has no `aria-label` that overrides its content, or the `h2` is not a descendant of `[role="button"]`.

### MINOR
2. (minor) **Focus lost after a dialog opened from a dropdown.** Affected: `BoardsPage.tsx:231-257` (Rename / Delete board), `BoardColumn.tsx:172-219` (Settings / Delete column).
   - The dialog's FocusScope records the *menu item* as the element to return to. That item is unmounted when the menu closes, so after Esc/Cancel focus most likely falls to `<body>`. This is not verified in a browser, and no test checks focus.
   - `modal={false}` correctly avoids the leftover `pointer-events: none` on body, so keep it.
   - Fix: keep a ref to the menu trigger and, in the dialog's `onCloseAutoFocus`, call `event.preventDefault(); triggerRef.current?.focus()`. Add a test with `toHaveFocus()` after Escape.
3. (minor) `components/layout/ShortcutsDialog.tsx:14-19`: the dialog lists `/` ("Focus the card filter") and `?` ("Show keyboard shortcuts") as if they work, but neither is bound. The description only explains card shortcuts. Either bind `?` (open this dialog) and `/` (focus the filter input) with a document keydown listener that ignores events from inputs and editable elements, or mark the unbound rows "Soon" the same way the sidebar does.
4. (minor) `features/boards/sidebarState.ts` / `BoardPage.tsx:102-103`: `drawerOpen` is not reset when the viewport leaves the mobile breakpoint. Open the drawer, widen the window past 767 px, then narrow it again: the modal drawer reappears and traps focus without any user action. Reset `drawerOpen` to `false` when `isMobile` becomes false (e.g. in the effect or event that reacts to the change, or derive `drawerOpen && isMobile`).
5. (minor) `features/boards/BoardPage.tsx:109-114`: the notice-to-toast effect is correct.
   - The fixed `id` makes the StrictMode double effect run update one toast instead of adding a second.
   - `dismissNotice()` lets a repeated identical message toast again.
   - Board switches reset `notice`.

   However, no test runs it under StrictMode. Add one `renderApp(BOARD, { strict: true })` variant of "rolls back and shows an error…" that asserts exactly one toast (`getAllByText(/busy/i)` has length 1).
6. (minor) `apps/web/index.html:8-28` repeats `THEME_STORAGE_KEY`, the default and the valid values from `src/lib/theme.ts`. Both are consistent today ('streamline-theme', default `light`, light/dark/system), but nothing protects against drift. Add a small test that reads `index.html` (`fs.readFileSync`) and asserts it contains `THEME_STORAGE_KEY` and the same preference literals.
7. (minor) Bundle grew from 711 to 826 kB, and Vite warns about it. As in the M2d review, `lazy()` for `BoardPage` (dnd + dialogs) in `routes.tsx` would split it. This can be a follow-up.

### Accepted deviations (no action)
- `--destructive` = `#AE2E24` (light) for red text and filled destructive buttons, with `--danger` kept for borders and fills. This is better for AA than DESIGN's `--danger` text (`#E34935` on white is below 4.5:1).
- No board description in the sidebar and no member avatars on board cards: the data does not exist in the API yet.
- "Create board" opens a dialog. Focus returns to the trigger, and tests check the request body and validation.
- TODO dot uses `--primary` (blue) and BACKLOG uses `--text-subtle` (grey), which matches the colours in §2.

### Done well
- Tokens follow DESIGN §6 exactly in both themes and are mapped onto every shadcn token through one `:root, .dark` block. I spot-checked contrast: text-subtle on column, primary on primary-subtle, destructive on danger-subtle, palette colours under white text, and dark primary buttons with `#1D2125` text. All are at or above 4.5:1.
- No-flash theme: the inline script, `readStoredTheme` and `initTheme` agree. The system listener is registered once and returns its unsubscribe. `useMediaQuery` uses `useSyncExternalStore` with a correct subscribe/cleanup. The test `matchMedia` stub really dispatches changes and is reset after each test.
- Docked sidebar: `inert` + `aria-hidden` when collapsed, and `aria-expanded`/`aria-controls` on the edge toggle, which stays outside the inert region so focus is never trapped inside. The mobile drawer is a real Radix dialog with a title, Escape/backdrop close and focus return. Storage failures are handled.
- Icon-only buttons all have `aria-label`s (help, theme, account menu, column actions, add card, sidebar toggles). Breadcrumbs use `nav` + `aria-current`. Skeletons are `aria-hidden` inside `role="status"` with sr-only text.
- Tests check behaviour, not just rendering:
  - theme persistence and `<html>` class;
  - OS theme following and ignoring;
  - sidebar persistence, narrow default and breakpoint switch;
  - drawer close paths;
  - toast shown once, store cleared, toast dismissable;
  - optimistic rename rollback;
  - menus driven by the keyboard (`test/menu.ts`).

  Updated auth tests keep their original assertions (request bodies, redirects, refresh counts) and only change selectors and paths.
- No `any`, no eslint-disable, no hard-coded colours outside the avatar palette and the overlay. No mentions of development tools, AI, `docs/` or ADRs anywhere in `apps/web`.

## Reviewer report, round 2
**Verdict:** APPROVE
**Checks:** lint PASS, typecheck PASS, test PASS (web 21 files, 219 tests), build PASS (Vite >500 kB chunk warning, deferred item 7)

Logic check repeated: `stores/board.ts`, the API modules and the ordering/drag helpers still have their pre-M2e mtimes (23:41–23:59). Round 2 changed presentation and tests only. No new untracked files appeared outside the expected set.

### Round 1 items
1. **Fixed.** `BoardColumn.tsx`: `dragHandleProps` now sits on a dedicated `GripVertical` span labelled "Drag column X". The category dot, `<h2>`, counter and strict marker are siblings outside it. While renaming, `ColumnTitleEditor` replaces that block and also sits outside the handle. Tests assert that no `[role="button"]` is an ancestor of the heading, the WIP counter or the title input (`BoardColumn.test.tsx:55`, `BoardPage.test.tsx:256-258, 320-321`). VIEWERs get no handle, and the block gets `pl-2` instead.
2. **Fixed.** `returnFocusRef` in `ConfirmDialog`, `ColumnSettingsDialog` and `RenameBoardDialog` uses `onCloseAutoFocus` → `preventDefault()` + focus on the "⋯" trigger. When the ref is null (e.g. after a successful delete has unmounted the trigger), the default behaviour applies. Focus tests cover the board menu (Rename + Escape, Delete + Cancel) and the column menu (`BoardPage.test.tsx:405, 413`). Rename keeps the `keepFocus` path to the inline input.
3. **Fixed.** `ShortcutsDialog`: Shift+N, `/` and `?` are marked "Soon", and the description says so. Esc is the only shortcut shown as available, and it works through Radix.
4. **Fixed.** `useSidebarState(isMobile)` uses React's "adjust state when a prop changes" pattern (`wasMobile` state) to close the drawer when the layout leaves mobile. This is correct and needs no effect. Test: `BoardSidebar.test.tsx:176` covers mobile → wide → mobile and checks that the drawer stays closed with `aria-expanded="false"`.
5. **Fixed.** A StrictMode variant of the rollback test (`BoardPage.test.tsx:548-558`) asserts exactly one toast.
6. **Fixed.** `theme.test.ts` imports `index.html?raw` and checks the storage key, all three preferences, the `light` default and the media query against the constants in `theme.ts`.
7. **Deferred** (bundle size). Acceptable as a follow-up.

### Stray edit in `BoardsPage.test.tsx`
The file is clean: 216 lines, one `describe`, 11 uniquely named tests, a single import block, and no duplicated test bodies or leftover fragments. The new focus test (`:161`) is well formed.

### BLOCKER
- none

### MAJOR
- none

### MINOR
- (minor, deviation to document) A mouse drag now starts only from the grip, not from the whole header as in DESIGN §2 ("Przeciąganie kolumny za nagłówek"). Keyboard dragging is unchanged. This is a reasonable accessibility trade-off. Ask the documenter to update DESIGN §2 (in the private docs repo) so the spec matches the implementation.
- (minor, optional) `badges.tsx` `ColumnCounter`: `aria-label` on a plain `<span>` without a role is ignored by assistive tech. The sr-only text already provides the description, so the attribute is redundant. Tests use `getByLabelText` on it, so remove it only together with those queries. No action required.
- (minor) Bundle size, item 7 carried over.

### Done well
- Each round 1 fix comes with a test that would have failed before the fix: the ancestor-role assertions, `toHaveFocus` after dialogs close, the resize sequence, the StrictMode single toast, and the drift check against `index.html`.
- The fixes stay small and presentation-only, and the earlier behavioural assertions are untouched.
- Privacy is still clean: no mentions of development tools, AI, `docs/` or ADRs in `apps/web`.
