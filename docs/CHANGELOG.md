# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/pl/1.1.0/).

## [Unreleased]
### Zmieniono
- `error-handler` mapuje `RankError` na odpowiedzi HTTP: `INVALID_RANK` i `INVALID_ARGUMENT` → 400, `INVALID_RANK_ORDER` → 409.
- `POST /boards` tworzy tablicę razem z domyślnymi kolumnami.

### Naprawiono
- Kolizja klucza jest rozpoznawana tylko dla P2002 na ograniczeniu `Column_boardId_rank_key` (wcześniej każdy P2002 w repozytorium kolumn), z obsługą adaptera pg bez `meta.target`.
- JSDoc w `packages/shared/src/schemas/column.ts` opisuje złagodzone umieszczanie po kolizji; test nie zostawia nieobsłużonej obietnicy.

### Dodano
- Kolumny tablicy (M2c, WF-COL-01..04): `GET/POST /boards/:boardId/columns`, `PATCH .../:columnId` (nazwa), `PATCH .../:columnId/move` (pozycja względem sąsiadów `afterId`/`beforeId`), `PATCH .../:columnId/settings` (kategoria, limit WIP, tryb ścisły), `DELETE .../:columnId`; ostatnia kolumna `DONE` nie może być usunięta ani przekategoryzowana.
- Model `Column`, enum `ColumnCategory` i migracja `20261111151836_columns` (`rank` z `COLLATE "C"`, odraczalne ograniczenie unikalności `(boardId, rank)`).
- Nowa tablica dostaje 4 domyślne kolumny (Backlog, To Do, In Progress, Done).
- Przypisywanie `rank` po stronie serwera: ponowienia przy kolizji, rebalansowanie w jednej transakcji, blokady wiersza tablicy; nowe kody błędów `NEIGHBOR_NOT_FOUND`, `INVALID_POSITION`, `RANK_CONFLICT`, `LAST_DONE_COLUMN`.
- `packages/shared`: schematy kolumn, `COLUMN_CATEGORIES`, `COLUMN_ERROR_CODES`, `DEFAULT_BOARD_COLUMNS`.
- Jawne limity czasu transakcji kolumn (`lock_timeout` 2 s, `maxWait` 2 s, `timeout` 5 s); przekroczenie daje 503 `BOARD_BUSY` (nowy kod w `COLUMN_ERROR_CODES`), bez ponowień po stronie serwera.
- Testy: shared 368, api 342, web 77 (nowe: `column.repository.test.ts`, `prisma-errors.test.ts`).
- Tablice, członkowie i role (M2b, WF-BOARD-01..03): `GET/POST /boards`, `GET/PATCH/DELETE /boards/:boardId`, `GET/POST /boards/:boardId/members`, `PATCH/DELETE /boards/:boardId/members/:userId` (usunięcie siebie = opuszczenie tablicy).
- Modele `Board`, `BoardMember`, enum `BoardRole` i migracja `20261111143833_boards_members` z częściowym indeksem unikalnym (najwyżej jeden `OWNER` na tablicę).
- `packages/shared`: schematy tablic i członków, `BOARD_ERROR_CODES`, macierz uprawnień (`BOARD_PERMISSIONS`, `hasBoardPermission`, `canManageMember`, `canLeaveBoard`).
- API: `ForbiddenError`, middleware `requireBoardPermission`, serwisy `board` i `board-access`, repozytoria `board` i `board-member`.
- Testy: shared 316, api 193, web 77.
- Własna implementacja LexoRank w `packages/shared` (M2a): `rankFirst`, `rankBetween`, `rankAfter`, `rankBefore`, `rankSequenceBetween`, `isValidRank`, `assertValidRank`, `rankSchema`, `RankError`, `RANK_MAX_LENGTH` (32), `needsRebalance`, `rebalanceRanks`.
- 198 nowych testów jednostkowych (shared: 226 łącznie), w tym zgodność z wyrocznią BigInt i kolejnością bajtową (`COLLATE "C"`).
- Benchmark `pnpm --filter @streamline/shared bench`: LexoRank vs reindeksacja całkowitoliczbowa O(n).
- Uwierzytelnianie frontendu (M1b): strony `/login` i `/register`, `authSlice` (Zustand, access token tylko w pamięci), klient axios z automatycznym odświeżaniem sesji (single-flight, ponowienie po `REFRESH_TOKEN_ROTATED`), `ProtectedRoute`, `GuestRoute`, przywracanie sesji po załadowaniu strony, przycisk wylogowania.
- Zależności `apps/web`: zustand 5.1.0, axios 1.21.1, react-hook-form 7.91.0, @hookform/resolvers 7.0.5, zod 4.6.7.
- Testy komponentów i klienta HTTP (testy: shared 28, api 77, web 77).
- Uwierzytelnianie backendu (M1a): `POST /auth/register`, `/auth/login`, `/auth/refresh`, `/auth/logout`, `GET /auth/me`; hasła bcryptjs, access token JWT (15 min), refresh token (7 dni) w ciasteczku httpOnly z rotacją, wykrywaniem ponownego użycia i oknem łaski.
- Modele Prisma `User`, `RefreshToken` i migracja `20261111010926_auth_users_refresh_tokens`.
- Middleware `requireAuth`, ograniczanie liczby żądań na `/auth/*`, redakcja nagłówków i pól wrażliwych w logach.
- Schematy Zod uwierzytelniania w `packages/shared`.
- Zmienne środowiskowe: `JWT_ACCESS_SECRET` (wymagana), `ACCESS_TOKEN_TTL_SECONDS`, `REFRESH_TOKEN_TTL_SECONDS`, `REFRESH_REUSE_GRACE_SECONDS`, `REFRESH_COOKIE_PATH`, `BCRYPT_COST`, `AUTH_RATE_LIMIT_*`, `DATABASE_URL_TEST`.
- Testy integracyjne Supertest na osobnej bazie `streamline_test` (testy: shared 28, api 77, web 5).
- Szkielet monorepo pnpm: `apps/web`, `apps/api`, `packages/shared` (M0).
- API: Express 5, endpoint `GET /health`, konfiguracja Zod, logowanie pino, obsługa błędów w jednolitym formacie.
- Pusty schemat Prisma 7 z adapterem `@prisma/adapter-pg` i `docker-compose.yml` z PostgreSQL 18.
- Frontend: Vite, React 19, Tailwind 4, shadcn/ui, strona startowa sprawdzająca `/health`.
- Pakiet `shared`: schematy odpowiedzi, robocza lista zdarzeń WebSocket, `boardRoom()`.
- ESLint, Prettier, Husky + lint-staged, Vitest (testy: shared 5, api 15, web 5).
- GitHub Actions CI.
- Test dymny `@hello-pangea/dnd` z React 19.

### Zmieniono
- `NotFoundError` i `ConflictError` przyjmują opcjonalny kod domenowy (domyślnie `NOT_FOUND` / `CONFLICT`).
- Routing frontendu: `/` jest trasą chronioną w układzie `AppLayout`; `/login` i `/register` dostępne tylko dla niezalogowanych.
- `createApp` przyjmuje `{ config, logger, db }` — klient bazy jest wstrzykiwany z `server.ts`.
- CI przekazuje `DATABASE_URL_TEST` zamiast `DATABASE_URL`.
