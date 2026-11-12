# Dziennik prac

Najnowsze wpisy na górze.

## 2026-11-12 — Nazar — M3a: karty (backend)
- Prisma: `Card`, `CardTransition`, enum `CardPriority`, rozszerzenie `Board` (`key`, `ownerId`, `nextCardNumber`); migracja `20261112014500_cards_and_board_keys` (ręczny SQL: `rank COLLATE "C"`, odraczalne `Card_columnId_rank_key`, blok PL/pgSQL uzupełniający klucze i właścicieli istniejących tablic, indeksy `CardTransition`).
- API: `routes/card.routes.ts`, `controllers/card.controller.ts`, `services/card.service.ts`, `services/ranked-write.ts`, `repositories/card.repository.ts`, `repositories/board-transaction.ts`; zmienione: `board.*` (klucz, `PATCH` z `key`, odpinanie kart przy usunięciu członka), `column.*` (wspólne pomocniki, 409 `COLUMN_NOT_EMPTY`), `board-member.repository.ts`, `errors/app-error.ts` (`BadRequestError`), `app.ts`, `test/helpers.ts`.
- `packages/shared`: `board-key.ts`, `schemas/card.ts`, `schemas/position.ts`, zmiany w `schemas/board.ts` i `schemas/column.ts`, `index.ts`.
- Frontend: tylko poprawki kompilacji (`BoardsPage.tsx`, `test/fixtures.ts`).
- Testy: shared 368 → 484, api 342 → 478, web 219. `lint`, `typecheck`, `test` przechodzą (testy API na prawdziwej bazie).
- Przegląd kodu: 2 rundy, zatwierdzone. Po rundzie 1 poprawiono: spójną migawkę listy kart (`REPEATABLE READ`), indeksy `fromColumnId`/`toColumnId`, limit czasu blokad przy usunięciu karty i członka, podwójny odczyt klucza w `getByKey`, przycinanie opisu, komunikaty migracji.
- Uwaga: migrację poprawiano w miejscu na niezmergowanej gałęzi — osoba z wcześniejszym szkicem w lokalnej bazie musi wykonać `prisma migrate reset`.
- Dokumentacja: ADR-043–049, API, DATABASE, ARCHITECTURE, PROJECT_STATE (M2d + M2e i PR #10 oznaczone jako zmergowane), CHANGELOG, notatki do rozdz. 3–5. Zmiany niezacommitowane.

## 2026-11-12 — Nazar — M2e: przebudowa interfejsu wg DESIGN
- `apps/web`: `index.css` (tokeny, cienie, narzędzia Tailwind, Inter), `index.html` (skrypt motywu), `lib/theme.ts`, `lib/avatar.ts`, `stores/theme.ts`, `hooks/useMediaQuery.ts`, `components/layout/` (`AppHeader`, `AppToaster`, `Breadcrumbs`, `Logo`, `ShortcutsDialog`, `ThemeSwitcher`, zmieniony `AppLayout`), `components/ui/` (`avatar`, `dropdown-menu`, `skeleton`, `sonner`, zmienione `button` i inne), `features/boards/` (m.in. `BoardSidebar`, `BoardColumn`, `BoardsPage`, `BoardPage`), `features/auth/` (`AuthCard`, `AuthField`, strony), testy; zależności w `apps/web/package.json` i `pnpm-lock.yaml`.
- Bez zmian logiki, API, bazy i zmiennych środowiskowych.
- Testy: web 172 → 219. `lint`, `typecheck`, `test`, `build`, `format:check` przechodzą; fragment 711 → 827 kB.
- Przegląd kodu: 2 rundy, zatwierdzone.
- Dokumentacja: ADR-038–042, DESIGN (§2 uchwyt, §6 `--destructive`), PROJECT_STATE, CHANGELOG, notatki do rozdz. 3–5. Nie sprawdzono w prawdziwej przeglądarce. Zmiany niezacommitowane.

## 2026-11-12 — Nazar — Poprawka: API nie startowało w `pnpm dev`
- Objaw: rejestracja/logowanie w przeglądarce kończyły się „Something went wrong”; proxy Vite zgłaszało `ECONNREFUSED` na `:4000`, a API nie wypisywało nic.
- Przyczyna: `tsx watch` (4.23.15, Windows) zawiesza się przed startem serwera, gdy stdin jest otwartym potokiem — tak uruchamia skrypty `pnpm -r --parallel --stream dev`. Bez trybu watch (i przy `pnpm --filter @streamline/api dev`) API startuje poprawnie.
- Zmiana: skrypt `dev` w `apps/api/package.json` używa wbudowanego trybu watch Node 24: `node --watch --import tsx --conditions=@streamline/source src/server.ts`. Sprawdzone: start przez `pnpm -r --parallel --stream`, `GET /health` = 200, restart po zmianie pliku w `packages/shared`.

## 2026-11-12 — Nazar — M2d: frontend tablic i kolumn
- `apps/web`: `src/stores/board.ts` (`useBoardStore`), `src/features/boards/` (`BoardsPage`, `BoardPage`, `BoardColumns`, `BoardColumn`, `ColumnTitleEditor`, `AddColumnForm`, `ColumnSettingsDialog`, `MembersDialog`, `ConfirmDialog`, `badges.tsx`, `columnOrder.ts`, `columnDrag.ts`, `columnSettings.ts`, `boardErrorMessage.ts`, `boardsApi.ts`, `columnsApi.ts`, `labels.ts`, `paths.ts`), nowe komponenty `components/ui/` (`dialog`, `alert-dialog`, `badge`, `checkbox`, `native-select`).
- Zmienione: `routes.tsx` (`/boards`, `/boards/:boardId`, `/` → `/boards`), `AppLayout.tsx`, `features/auth/redirect.ts` (`HOME_PATH`) i testy auth, `test/fixtures.ts`, `test/renderApp.tsx`, `test/setup.ts` (stub `ResizeObserver`). Usunięto `pages/HomePage.tsx`, `pages/HomePage.test.tsx`, `hooks/useApiHealth.ts`.
- Testy: web 77 → 172 (shared 368, api 342). `lint`, `typecheck`, `format:check`, `build` przechodzą.
- Przegląd kodu: 1 runda, zatwierdzone; uwagi drobne 1–5 poprawione. Uwaga 6 (fragment 537 → 711 kB, rozważyć `lazy()` dla `BoardPage`) pozostaje długiem technicznym.
- Dokumentacja: ADR-034–037, ARCHITECTURE, PROJECT_STATE (M2c oznaczone jako zmergowane, PR #8), notatki do rozdz. 3–5. Zmiany niezacommitowane.

## 2026-11-11 — Nazar — M2c: kolumny tablicy (backend)
- Prisma: `Column`, enum `ColumnCategory`; migracja `20261111151836_columns` z ręcznym SQL (`rank TEXT COLLATE "C"`, `UNIQUE (boardId, rank) DEFERRABLE INITIALLY IMMEDIATE`).
- `packages/shared`: `schemas/column.ts` (schematy, `COLUMN_ERROR_CODES`, `DEFAULT_BOARD_COLUMNS`) i testy `column.test.ts`; eksport w `index.ts`.
- API: `column.repository.ts`, `column.service.ts`, `column.controller.ts`, `column.routes.ts` (montowane w `board.routes.ts`), `app.ts`; `board.repository.ts` i `board.service.ts` tworzą 4 domyślne kolumny; `error-handler.ts` mapuje `RankError`; `test/helpers.ts`; testy: `column.service.test.ts`, `column.routes.test.ts`, uzupełnione `board.service.test.ts` i `error-handler.test.ts`.
- Przegląd kodu: 2 rundy, zatwierdzone. Poprawki po rundzie 1: JSDoc złagodzonego umieszczania, jawne limity transakcji kolumn (`lock_timeout` przez `set_config`, `maxWait`, `timeout`; `ColumnTransactionOptions` w `column.repository.ts`) z mapowaniem 55P03/P2028 na `BoardBusyError` → `ServiceUnavailableError` → 503 `BOARD_BUSY`, rozpoznawanie tylko P2002 na `Column_boardId_rank_key` (`isUniqueViolationOf`, `postgresErrorCode` w `prisma-errors.ts`), poprawka nieobsłużonej obietnicy w teście. Nowe testy: `column.repository.test.ts` (integracyjne), `prisma-errors.test.ts`. Testy: shared 368, api 342, web 77. Uwagi z rundy 2 w `docs/PROJECT_STATE.md`.
- Dokumentacja: API, DATABASE, ARCHITECTURE, ADR-028–033, notatki do rozdz. 3–5, stan projektu (M2b oznaczone jako zmergowane, PR #7). Zmiany niezacommitowane.

## 2026-11-11 — Nazar — M2b: tablice, członkowie i role
- Prisma: `Board`, `BoardMember`, enum `BoardRole`; migracja `20261111143833_boards_members` z ręcznym częściowym indeksem unikalnym `BoardMember_one_owner_per_board`.
- `packages/shared`: `schemas/board.ts`, `permissions.ts` (macierz uprawnień) wraz z testami; eksport w `index.ts`.
- API: `ForbiddenError` i kody domenowe w błędach (`errors/app-error.ts`), `repositories/prisma-errors.ts`, `board.repository.ts`, `board-member.repository.ts`, `user.repository.ts`, `board-access.service.ts`, `board.service.ts`, `require-board-permission.ts`, `board.controller.ts`, `board.routes.ts`, `app.ts`, `routes/index.ts`, `test/helpers.ts`; testy: `board.service.test.ts`, `require-board-permission.test.ts`, `board.routes.test.ts`.
- Przegląd kodu: 1 runda, zatwierdzone. Testy: shared 316, api 193, web 77.
- Dokumentacja: API, DATABASE, ARCHITECTURE, ADR-025–027, notatki do rozdz. 3–5, stan projektu (m.in. kontrakt użycia `rank` przeniesiony do M2c).
- Dług techniczny: patrz `docs/PROJECT_STATE.md`. Zmiany niezacommitowane.

## 2026-11-11 — Nazar — M2a: własny LexoRank i benchmark
- `packages/shared/src/rank/`: klucze base-62 (`keys.ts`), `rankBetween`/`rankAfter`/`rankBefore`/`rankSequenceBetween`/`rankFirst` (`lexorank.ts`), `needsRebalance`/`rebalanceRanks` (`rebalance.ts`), `index.ts`; eksport z `packages/shared/src/index.ts`.
- Testy: `keys/lexorank/rebalance/collation.test.ts` (198 nowych, shared 226); `packages/shared/test-utils/prng.ts` (mulberry32).
- Benchmark `packages/shared/bench/rank.bench.ts` (skrypt `bench`) vs reindeksacja całkowitoliczbowa O(n); wyniki w notatkach do rozdz. 5.3.
- Zmieniono też `packages/shared/package.json` i `tsconfig.json`.
- Przegląd kodu: 1 runda, zatwierdzone; po nim drobne poprawki (kontrakt użycia w JSDoc, pomiar wsadowy w benchmarku).
- ADR-022–024. Testy API nie uruchomione (brak Dockera) — do wykonania przed commitem.

## 2026-11-11 — Nazar — Wymagania i rozszerzony plan prac
- Nowy dokument `docs/REQUIREMENTS.md`: macierz uprawnień ról, wymagania funkcjonalne (MoSCoW) i niefunkcjonalne, zakres wyłączony.
- Wyróżnik projektu: analityka przepływu (lead/cycle time, przepustowość, CFD, aging WIP, prognoza Monte Carlo, prawo Little'a) oraz obecność użytkowników w czasie rzeczywistym.
- Opcjonalny asystent AI (podział zadania, kryteria akceptacji, podsumowanie tygodnia) za adapterem dostawcy.
- Model danych: kategoria i limit WIP kolumny, `CardTransition`, `AiUsage`; ADR-020, ADR-021.
- Plan prac rozszerzony do M9 (`docs/WORKFLOW.md`, sekcja 7); M2 podzielony na M2a–M2d.

## 2026-11-11 — Nazar — M1b: uwierzytelnianie (frontend)
- `authSlice` (Zustand) z access tokenem wyłącznie w pamięci; klient axios z interceptorem (401 → `/auth/refresh` → ponowienie, single-flight, jedno opóźnione ponowienie po `REFRESH_TOKEN_ROTATED`, wylogowanie po porażce); `sessionEpoch` odrzuca nieaktualne wyniki odświeżania.
- Strony `/login`, `/register` (react-hook-form + schematy `shared`), `ProtectedRoute`, `GuestRoute`, `AuthBootstrap` (przywracanie sesji, odporne na StrictMode), `AppLayout` z przyciskiem wylogowania, walidacja adresu przekierowania.
- Nowe zależności `apps/web`: zustand, axios, react-hook-form, @hookform/resolvers, zod. Testy z ręcznym adapterem axios (bez MSW). Testy: shared 28, api 77, web 77.
- Dokumentacja: ADR-016–019, ARCHITECTURE, TECH_STACK, PROJECT_STATE (M1a oznaczone jako zmergowane), notatki do rozdziałów 3, 4, 5.
- Przegląd kodu: zatwierdzony po 2 rundach (runda 1: wylogowanie w trakcie odświeżania — naprawione).
- Pliki: `apps/web/src/{stores,lib,features/auth,components/layout,components/ui,test,routes.tsx,pages}`, `apps/web/package.json`, `pnpm-lock.yaml`, `docs/*`.

## 2026-11-11 — Nazar — M1a: uwierzytelnianie (backend)
- Modele `User` i `RefreshToken`, migracja `20261111010926_auth_users_refresh_tokens`.
- Endpointy `/auth/register`, `/auth/login`, `/auth/refresh`, `/auth/logout`, `/auth/me`; bcryptjs, JWT (jose), refresh token z rotacją i wykrywaniem ponownego użycia (okno łaski), `requireAuth`, ograniczanie liczby żądań, redakcja logów.
- `createApp({ config, logger, db })` — ręczne wstrzykiwanie zależności; schematy Zod uwierzytelniania w `packages/shared`.
- Testy integracyjne na osobnej bazie `streamline_test` (globalSetup, TRUNCATE); CI używa `DATABASE_URL_TEST`. Testy: shared 28, api 77, web 5.
- Dokumentacja: ADR-010–015, aktualizacja API/DATABASE/TECH_STACK/PROJECT_STATE (M0 oznaczone jako zmergowane), notatki do rozdziałów 3 i 4.
- Przegląd kodu: zatwierdzony po 2 rundach.
- Pliki: `apps/api/prisma/*`, `apps/api/src/{app,server,config,errors,middleware,controllers,routes,services,repositories,test}`, `packages/shared/src/schemas/auth.ts`, `.github/workflows/ci.yml`, `docs/*`.

## 2026-11-11 — Nazar — M0: szkielet monorepo
- Monorepo pnpm, API Express 5 z `/health`, frontend Vite + React + Tailwind + shadcn/ui, pakiet `shared`.
- Prisma 7 (pusty schemat), `docker-compose.yml`, ESLint/Prettier/Husky, Vitest, GitHub Actions.
- Zweryfikowano `@hello-pangea/dnd` z React 19 (test dymny).
- Przegląd kodu: zatwierdzony w pierwszej rundzie.
- Pliki: `package.json`, `pnpm-workspace.yaml`, `eslint.config.js`, `docker-compose.yml`, `.github/workflows/ci.yml`, `apps/*`, `packages/shared`, `docs/*`.

## 2026-11-11 — Nazar — Przygotowanie projektu
- Dobór i opis stosu technologicznego (`docs/TECH_STACK.md`).
- Opis organizacji pracy zespołu (`docs/WORKFLOW.md`).
- Wstępne decyzje architektoniczne (`docs/DECISIONS.md`).
