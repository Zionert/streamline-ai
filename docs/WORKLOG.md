# Dziennik prac

Najnowsze wpisy na górze.

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
