# Stan projektu StreamLine

_Aktualizacja: 2026-11-11_

## W skrócie
M0, M1a, M1b i M2a są zmergowane do `main` (ostatnio PR #3–#6). M2b (tablice, członkowie i role: WF-BOARD-01..03 ✅) jest zaimplementowane i zatwierdzone w przeglądzie kodu (1 runda); zmiany są na gałęzi `feat/m2b-boards-members` i nie są jeszcze zacommitowane. Następny etap: M2c (kolumny z kategorią i limitem WIP).

## Gotowe
- Dokumentacja stosu, organizacji pracy i decyzji (ADR-001–024).
- M0: monorepo pnpm, API Express 5 z `GET /health`, frontend Vite/React/Tailwind/shadcn, pakiet `shared`, `docker-compose.yml` (PostgreSQL 18), ESLint/Prettier/Husky/lint-staged, Vitest, GitHub Actions CI. `@hello-pangea/dnd` 18.0.1 potwierdzony z React 19.3.1.
- M1a (backend):
  - modele `User`, `RefreshToken` i migracja `20261111010926_auth_users_refresh_tokens` (`docs/DATABASE.md`),
  - `POST /auth/register`, `/auth/login`, `/auth/refresh`, `/auth/logout`, `GET /auth/me` (`docs/API.md`),
  - hasła: bcryptjs; access token JWT (jose, HS256, 15 min) w ciele odpowiedzi; refresh token nieprzezroczysty (7 dni), w bazie tylko skrót SHA-256, ciasteczko httpOnly `refresh_token`, rotacja z wykrywaniem ponownego użycia i oknem łaski (ADR-010–013),
  - middleware `requireAuth`, ograniczanie liczby żądań na `/auth/*`, redakcja danych wrażliwych w logach pino,
  - schematy Zod w `packages/shared` (`registerSchema`, `loginSchema`, `emailSchema`, `passwordSchema`, `publicUserSchema`, `authResponseSchema`, `meResponseSchema`),
  - `createApp({ config, logger, db })` z ręcznym wstrzykiwaniem zależności,
  - testy integracyjne Supertest na osobnej bazie `streamline_test` (ADR-015).
- M1b (frontend, `apps/web`):
  - `authSlice` (Zustand, `src/stores/auth.ts`): `user`, `accessToken` (tylko w pamięci), `status`, `sessionEpoch`,
  - klient axios (`src/lib/api.ts`): nagłówek Bearer, odświeżanie po 401 (single-flight), jedno opóźnione ponowienie po `REFRESH_TOKEN_ROTATED`, wylogowanie przy ostatecznej porażce (ADR-016, ADR-017),
  - strony `/login` i `/register` (react-hook-form + `zodResolver` ze schematami `loginSchema`/`registerSchema` z `shared`, komponenty shadcn/ui),
  - `ProtectedRoute` (przekierowanie do `/login` ze stanem `from`), `GuestRoute`, `AuthBootstrap` (przywracanie sesji przez `/auth/refresh`, odporne na StrictMode — ADR-018), przycisk „Log out” w `AppLayout`,
  - testy RTL z ręcznym adapterem axios zamiast MSW (ADR-019).
- M2a (`packages/shared/src/rank/`, ADR-002, ADR-022–024):
  - `rankFirst`, `rankBetween(a, b)` (`null` = granica otwarta), `rankAfter`, `rankBefore`, `rankSequenceBetween`, `isValidRank`, `assertValidRank`, `rankSchema`, `RankError` (kody `INVALID_RANK`, `INVALID_RANK_ORDER`, `INVALID_ARGUMENT`), `RANK_MAX_LENGTH` (32), `needsRebalance`, `rebalanceRanks`,
  - klucz: ułamek w systemie 62-cyfrowym (`0-9A-Za-z`), niepusty, nie kończy się na `0`; porządek leksykograficzny = liczbowy = `COLLATE "C"` w PostgreSQL,
  - testy: 198 nowych (`keys`, `lexorank`, `rebalance`, `collation`), w tym zgodność z wyrocznią BigInt i dowód różnicy względem `localeCompare`,
  - benchmark `pnpm --filter @streamline/shared bench` (`pnpm build && node bench/rank.bench.ts`, natywne usuwanie typów w Node 24, działa na `dist`, bez nowych zależności); wyniki: `docs/thesis-notes/rozdzial-5.md`, sekcja 5.3.
- M2b (tablice, członkowie, role; WF-BOARD-01..03 ✅; ADR-025–027), gałąź `feat/m2b-boards-members`, niezacommitowane:
  - modele `Board`, `BoardMember`, enum `BoardRole` (`OWNER`/`ADMIN`/`MEMBER`/`VIEWER`), migracja `20261111143833_boards_members` z ręcznym częściowym indeksem unikalnym `BoardMember_one_owner_per_board` (najwyżej jeden `OWNER` na tablicę; co najmniej jednego pilnuje serwis) — `docs/DATABASE.md`,
  - endpointy `GET/POST /boards`, `GET/PATCH/DELETE /boards/:boardId`, `GET/POST /boards/:boardId/members`, `PATCH/DELETE /boards/:boardId/members/:userId` (usunięcie siebie = opuszczenie tablicy) — `docs/API.md`,
  - `packages/shared`: `schemas/board.ts` (schematy, `BOARD_ERROR_CODES`: `USER_NOT_FOUND`, `ALREADY_MEMBER`, `OWNER_CANNOT_LEAVE`), `permissions.ts` (`BOARD_ACTIONS`, `BOARD_PERMISSIONS`, `hasBoardPermission`, `canManageMember`, `canLeaveBoard`),
  - API: `ForbiddenError`, opcjonalne kody domenowe w `NotFoundError`/`ConflictError`, `repositories/prisma-errors.ts`, repozytoria `board` i `board-member` (warunkowe `UPDATE`/`DELETE` z `role <> 'OWNER'`), `board-access.service`, `board.service` (ponownie sprawdza uprawnienia), middleware `requireBoardPermission(action, { param })` zapisujące `res.locals.boardMembership` — `docs/ARCHITECTURE.md`,
  - dwuwarstwowa kontrola ról (ADR-025), 404 dla nie-członków i 409 przy opuszczaniu przez właściciela (ADR-026), inwariant jednego właściciela (ADR-027); brak zdarzeń Socket.io,
  - nowe testy: jednostkowe i schematy w `shared`, serwis, middleware, integracyjne `board.routes.test.ts`.
- Sprawdzone lokalnie (M2b): `lint`, `typecheck`, `test`, `format:check` — shared 316, api 193, web 77 (powtórzone w przeglądzie kodu); testy API wymagają działającej bazy.
- Sprawdzone wcześniej (M1b): `build` web (ostrzeżenie: jeden fragment 537 kB).

## W toku
- M2b oczekuje na commit i PR (commit tylko na wyraźną prośbę).

## Kolejne kroki
Zakres i priorytety: `docs/REQUIREMENTS.md`; plan: `docs/WORKFLOW.md`, sekcja 7.
1. **Zatwierdzenie M2b**: commit i PR.
2. **M2c** — kolumny z kategorią (`BACKLOG`/`TODO`/`IN_PROGRESS`/`DONE`) i limitem WIP; trasy zagnieżdżone `/boards/:boardId/columns` z `Router({ mergeParams: true })` i `requireBoardPermission('column:write' | 'column:settings')`. Kontrakt użycia `rank` (dotyczy kolumn i kart; M2b nie używał `rank`, więc pozostaje do zrealizowania tutaj i w M3):
   - kolumna `rank` jako tekst z `COLLATE "C"` (inaczej `ORDER BY` w PostgreSQL da inną kolejność niż `<` w JS dla mieszanej wielkości liter),
   - unikalność (kontener, `rank`) + ponowienie przy kolizji albo blokada wierszy — `rankBetween` jest deterministyczne, więc równoległe wstawienia w tę samą lukę dają ten sam klucz,
   - gdy wynik dłuższy niż `RANK_MAX_LENGTH` — nie zapisywać, lecz przepisać całą listę `rebalanceRanks` w tej samej transakcji; do ustalenia: zbiorcze zdarzenie socket po rebalansowaniu,
   - `RankError` mapować na HTTP 400/409 (obsługa w `error-handler`),
   - API przyjmuje identyfikatory sąsiadów lub docelowy indeks, nie surowe rank; klucz liczy serwer z zapisanych sąsiadów,
   - jeden test integracyjny na prawdziwym PostgreSQL: `ORDER BY rank COLLATE "C"` zgodne z porządkiem z `shared`.
3. **M2d** — frontend tablic i kolumn (użyć `hasBoardPermission` z `shared` do ukrywania kontrolek).
4. Od M3 każde przeniesienie karty zapisuje `CardTransition` (ADR-020) — warunek analityki przepływu (M6).
5. WF-BOARD-04 (przekazanie własności): akcja `board:transfer` jest w macierzy uprawnień, ale endpointu jeszcze nie ma — do zrobienia przed możliwością opuszczenia tablicy przez właściciela.

### Uwagi dla kolejnych etapów
- Żądania chronione wykonywać przez instancję `api` z `apps/web/src/lib/api.ts` (obsługuje Bearer i odświeżanie); żądań do `/auth/*` nie opakowywać własnym odświeżaniem.
- Przeglądarka wywołuje `/api/auth/*` (proxy Vite); ciasteczko ma `Path=/api/auth`.

## Uruchamianie
1. Wymagane: Node 24 (`.nvmrc`), pnpm 10.32.0, Docker.
2. `pnpm install` (generuje klienta Prisma w `postinstall`).
3. Skopiować `.env.example` do `.env` (root, dla compose), `apps/api/.env.example` do `apps/api/.env`, `apps/web/.env.example` do `apps/web/.env`.
4. W `apps/api/.env` ustawić `JWT_ACCESS_SECRET` (wymagany, min. 32 znaki; API nie wystartuje bez niego). Przykład: `node -e "console.log(require('node:crypto').randomBytes(48).toString('base64url'))"`.
5. `pnpm db:up` (PostgreSQL w Dockerze).
6. `pnpm --filter @streamline/api db:migrate` — stosuje migracje (istnieją migracje `20261111010926_auth_users_refresh_tokens` i `20261111143833_boards_members`).
7. `pnpm dev` — API na `:4000`, web na `:5173` (proxy `/api` do API).

Kontrole: `pnpm lint && pnpm typecheck && pnpm test`, `pnpm format:check`, `pnpm build`. Testy API wymagają działającej bazy (`pnpm db:up`); baza `streamline_test` jest tworzona i migrowana automatycznie.

Zmienne środowiskowe uwierzytelniania (`apps/api/.env.example`): `JWT_ACCESS_SECRET`, `ACCESS_TOKEN_TTL_SECONDS` (900), `REFRESH_TOKEN_TTL_SECONDS` (604800), `REFRESH_REUSE_GRACE_SECONDS` (30), `REFRESH_COOKIE_PATH` (`/api/auth`), `BCRYPT_COST` (12), `AUTH_RATE_LIMIT_WINDOW_MS` (900000), `AUTH_RATE_LIMIT_MAX` (100), `AUTH_CREDENTIALS_RATE_LIMIT_MAX` (10), opcjonalnie `DATABASE_URL_TEST`.

## Znane problemy / dług techniczny
- Frontend: odświeżenie może rozpocząć się w trakcie `POST /auth/logout` (równoległe 401 w ciągu ok. 50 ms) i zrotować ciasteczko, zanim wylogowanie je unieważni; możliwe wzmocnienie: flaga „wylogowywanie” powodująca odrzucenie `refreshSession`.
- Frontend: `login`/`register` nie zwiększają `sessionEpoch` — odświeżenie w toku podczas logowania mogłoby nadpisać lub wyczyścić nową sesję (w obecnym UI nieosiągalne).
- Kody błędów (np. `REFRESH_TOKEN_ROTATED`) są zduplikowane w `apps/api` i `apps/web` — przenieść do `packages/shared`.
- Brak podziału kodu (jeden fragment 537 kB) — leniwe ładowanie tras w późniejszym etapie.
- Podwójna nawigacja po zalogowaniu/wylogowaniu (nieszkodliwa).
- Nazwy `WS_EVENTS` są robocze — do zatwierdzenia z funkcjonalnościami.
- `error-handler` mapuje wszystkie błędy 4xx parsera ciała na `BAD_REQUEST`; brak zabezpieczenia `headersSent`.
- Krok `prisma generate` w CI jest zbędny (robi to `postinstall`).
- Pliki w korzeniu (np. `eslint.config.js`) nie są objęte `pnpm -r lint`.
- TypeScript 6.0.3 — przejście na TS 7 po wsparciu w `typescript-eslint`.
- Brak ustawienia `trust proxy` — za reverse proxy limit żądań liczyłby adres proxy zamiast klienta.
- Access token bez claimów `iss`/`aud` — dodać przed uwierzytelnianiem Socket.io.
- Wygasłe i unieważnione wiersze `RefreshToken` nigdy nie są usuwane.
- `409 CONFLICT` przy rejestracji ujawnia istnienie konta (świadomy kompromis, ograniczony limitem żądań).
- Licznik ograniczania żądań jest w pamięci procesu (jedna instancja); okno łaski rotacji używa zegara aplikacji (również założenie jednej instancji).
- Okno łaski opóźnia wykrycie kradzieży, gdy atakujący odświeży token przed klientem (ADR-013).

- M2b: wyścig przy degradacji — rola jest sprawdzana w middleware, a mutacja wykonuje się później; równoległa zmiana roli wywołującego może pozwolić na jedną operację z już cofniętym uprawnieniem (właściciela chronią warunkowe zapytania, pozostałych nie).
- M2b: gdy w przyszłości dojdzie usuwanie kont, kaskada usunie członkostwo właściciela i zostawi tablicę bez właściciela (indeks pilnuje tylko „najwyżej jeden”); potrzebna reguła (np. przekazanie własności lub usunięcie tablicy).
- M2b: `POST /boards` przez użytkownika usuniętego po wydaniu tokena dałoby 500 (naruszenie FK) — obecnie nieosiągalne, bo nie ma usuwania kont.
- M2b: `Board.updatedAt` zmienia się tylko przy zmianie nazwy (nie przy zmianach członków ani zawartości).
- M2b (drobne po przeglądzie): komentarz w `schema.prisma` mówi „exactly one” owner, a poprawnie jest „at most one”; `canManageMember` nie jest używane przez API (reguła `role <> OWNER` żyje w repozytorium — utrzymywać zgodność); parametr `:userId` nie jest walidowany schematem Zod (nieznany id daje 404 członka).

## Przeszkody / otwarte pytania
- Nie potwierdzono przebiegu CI na GitHub (w tym `DATABASE_URL_TEST` z usługą PostgreSQL); sprawdzić tagi akcji (`checkout@v7`, `pnpm/action-setup@v6`, `setup-node@v7`) oraz działanie `DATABASE_URL_TEST` z usługą PostgreSQL.
- `@hello-pangea/dnd`: ostatnie wydanie luty 2025 — dnd-kit jako plan awaryjny.
