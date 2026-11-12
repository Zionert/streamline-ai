# Stan projektu StreamLine

_Aktualizacja: 2026-11-12_

## W skrócie
M0, M1a, M1b, M2a–M2e i M3a są zmergowane do `main` (M2b: PR #7, `60f9c48`; M2c: PR #8, `643afbd`; M2d + M2e: PR #9, `2a15599`; M3a: PR #11, `7195d9b`), podobnie poprawka trybu watch serwera deweloperskiego (PR #10, `1f547a6`). M3b (frontend kart: WF-CARD-01 ✅, WF-CARD-02 ✅, WF-CARD-04 ✅ także w UI; WF-CARD-07 — część: filtr po tekście i osobie) jest zaimplementowany na gałęzi `feat/m3b-cards-frontend` i zatwierdzony w przeglądzie kodu (2 rundy); zmiany nie są jeszcze zacommitowane. Następny etap: M4 (Socket.io, synchronizacja w czasie rzeczywistym) wg `docs/WORKFLOW.md`, sekcja 7.

## Gotowe
- Dokumentacja stosu, organizacji pracy i decyzji (ADR-001–042).
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
- M2b (tablice, członkowie, role; WF-BOARD-01..03 ✅; ADR-025–027; PR #7, zmergowane):
  - modele `Board`, `BoardMember`, enum `BoardRole` (`OWNER`/`ADMIN`/`MEMBER`/`VIEWER`), migracja `20261111143833_boards_members` z ręcznym częściowym indeksem unikalnym `BoardMember_one_owner_per_board` (najwyżej jeden `OWNER` na tablicę; co najmniej jednego pilnuje serwis) — `docs/DATABASE.md`,
  - endpointy `GET/POST /boards`, `GET/PATCH/DELETE /boards/:boardId`, `GET/POST /boards/:boardId/members`, `PATCH/DELETE /boards/:boardId/members/:userId` (usunięcie siebie = opuszczenie tablicy) — `docs/API.md`,
  - `packages/shared`: `schemas/board.ts` (schematy, `BOARD_ERROR_CODES`: `USER_NOT_FOUND`, `ALREADY_MEMBER`, `OWNER_CANNOT_LEAVE`), `permissions.ts` (`BOARD_ACTIONS`, `BOARD_PERMISSIONS`, `hasBoardPermission`, `canManageMember`, `canLeaveBoard`),
  - API: `ForbiddenError`, opcjonalne kody domenowe w `NotFoundError`/`ConflictError`, `repositories/prisma-errors.ts`, repozytoria `board` i `board-member` (warunkowe `UPDATE`/`DELETE` z `role <> 'OWNER'`), `board-access.service`, `board.service` (ponownie sprawdza uprawnienia), middleware `requireBoardPermission(action, { param })` zapisujące `res.locals.boardMembership` — `docs/ARCHITECTURE.md`,
  - dwuwarstwowa kontrola ról (ADR-025), 404 dla nie-członków i 409 przy opuszczaniu przez właściciela (ADR-026), inwariant jednego właściciela (ADR-027); brak zdarzeń Socket.io,
  - nowe testy: jednostkowe i schematy w `shared`, serwis, middleware, integracyjne `board.routes.test.ts`.
- M2c (kolumny, backend; WF-COL-01..04 ✅; ADR-028–033; PR #8, zmergowane):
  - model `Column` (`rank` jako `TEXT COLLATE "C"`, `category` enum `ColumnCategory` `BACKLOG`/`TODO`/`IN_PROGRESS`/`DONE`, `wipLimit`, `wipStrict`), migracja `20261111151836_columns` z ręcznym SQL (collation, ograniczenie `UNIQUE (boardId, rank)` jako `DEFERRABLE INITIALLY IMMEDIATE`) — `docs/DATABASE.md`,
  - 4 domyślne kolumny nowej tablicy (Backlog, To Do, In Progress, Done) tworzone razem z tablicą (`DEFAULT_BOARD_COLUMNS`, `rankSequenceBetween`); istniejące tablice nie są uzupełniane,
  - endpointy pod `/boards/:boardId/columns`: `GET`, `POST`, `PATCH /:columnId` (zmiana nazwy), `PATCH /:columnId/move` (`afterId`/`beforeId`), `PATCH /:columnId/settings` (kategoria, limit WIP, tryb ścisły), `DELETE /:columnId` — `docs/API.md`; ostatnia kolumna `DONE` nie może być usunięta ani przekategoryzowana (409 `LAST_DONE_COLUMN`),
  - kontrakt `rank` zrealizowany: unikalność + ponowienia (do 5 prób), rebalansowanie w tej samej transakcji, mapowanie `RankError` w `error-handler` (`INVALID_RANK`/`INVALID_ARGUMENT` → 400, `INVALID_RANK_ORDER` → 409), dwupoziomowa blokada wiersza tablicy (ADR-028–031) — `docs/ARCHITECTURE.md`; jawne limity czasu transakcji (`lock_timeout` 2 s, `maxWait` 2 s, `timeout` 5 s) i 503 `BOARD_BUSY` bez ponowień po stronie serwera (ADR-033); kolizją klucza jest tylko P2002 na `Column_boardId_rank_key`,
  - `packages/shared`: `schemas/column.ts` (`COLUMN_CATEGORIES`, ścisłe schematy `createColumnSchema`, `renameColumnSchema`, `moveColumnSchema`, `updateColumnSettingsSchema`, `columnSchema`, `COLUMN_ERROR_CODES`: `NEIGHBOR_NOT_FOUND`, `INVALID_POSITION`, `RANK_CONFLICT`, `LAST_DONE_COLUMN`, `BOARD_BUSY`, `DEFAULT_BOARD_COLUMNS`),
  - API: `column.repository` (blokady wiersza tablicy, `setRanks` jednym `UPDATE ... FROM unnest`), `column.service`, `column.controller`, `column.routes` (`Router({ mergeParams: true })`); brak kart (M3), brak zdarzeń Socket.io, brak frontendu,
  - testy: schematy, serwis, integracyjne `column.routes.test.ts` (m.in. porządek `ORDER BY rank` w PostgreSQL = porządek JS, równoległe przeniesienia w tę samą lukę, wstrzyknięta kolizja z ponowieniem, rebalansowanie przy wstawianiu i przeniesieniu, wyścigi o inwariant `DONE`).
- M2d (frontend tablic i kolumn, `apps/web`; WF-BOARD-01..03 i WF-COL-01..04 ✅ także w UI; ADR-034–037; PR #9, zmergowane):
  - nagłówek (`AppLayout`): link z nazwą aplikacji do `/boards`, nazwa użytkownika, wylogowanie; `/` przekierowuje do `/boards`, `HOME_PATH` = `/boards`; usunięto `HomePage` i `useApiHealth` (ADR-037),
  - `/boards`: lista tablic z rolą użytkownika, tworzenie (`createBoardSchema`), zmiana nazwy (`board:rename`), usuwanie z potwierdzeniem (`board:delete`), stany ładowania, pusty i błąd z ponowieniem,
  - `/boards/:boardId`: kolumny poziomo (nazwa, kategoria, licznik WIP `n / limit` czerwony po przekroczeniu, liczba kart przez prop `getCardCount` — domyślnie 0 do M3), dodawanie kolumny, zmiana nazwy w miejscu (Enter/Escape, optymistycznie z wycofaniem), usuwanie z potwierdzeniem (komunikat `LAST_DONE_COLUMN`), przeciąganie `@hello-pangea/dnd` (poziome; wyłączone bez `column:write` i w trakcie zapisu przeniesienia), okno ustawień (natywny select kategorii, pusty limit WIP = `null`, `wipStrict`; wysyłane tylko zmienione pola), okno członków (dodanie po e-mailu i roli, zmiana roli, usunięcie z potwierdzeniem, opuszczenie tablicy → `/boards`; `canManageMember`/`canLeaveBoard`; własny wiersz bez kontrolek zarządzania), stan 404 i komunikat `BOARD_BUSY`,
  - `useBoardStore` (`src/stores/board.ts`): lista tablic, otwarta tablica, kolumny, członkowie, `isMovingColumn`, `notice`; liczniki generacji odrzucają spóźnione odpowiedzi; zerowanie przy zmianie `sessionEpoch`/użytkownika,
  - protokół przeniesienia (ADR-034, ADR-035): zmiana optymistyczna, zawsze oba `afterId`/`beforeId` (`null` na krańcach), jedno przeniesienie naraz, przy błędzie wycofanie tylko przeniesionej kolumny (`restoreOriginalPosition`), odświeżenie listy po `INVALID_POSITION`/`NEIGHBOR_NOT_FOUND`/404, wykrywanie rebalansowania po `prev.rank < moved.rank < next.rank` (ścisłe `<`, nigdy `localeCompare`) z pełnym odświeżeniem, `createColumn` dopisuje na końcu i odświeża, gdy `rank` nie pasuje,
  - nowe komponenty UI (radix-ui, bez nowych zależności): `dialog`, `alert-dialog`, `badge`, `checkbox`, `native-select` (ADR-036); stub `ResizeObserver` w `test/setup.ts`,
  - testy: web 77 → 172 (`columnOrder`, `columnDrag`, `columnSettings`, `boardErrorMessage`, `stores/board`, `BoardsPage`, `BoardPage` z prawdziwym przeciąganiem klawiaturą); zaktualizowano testy auth.
- M2e (przebudowa interfejsu wg `docs/DESIGN.md`, tylko warstwa prezentacji — bez zmian logiki, API, bazy i zmiennych środowiskowych; ADR-038–042; PR #9, zmergowane):
  - nowe zależności (wersje dokładne): `lucide-react` 1.72.0, `sonner` 2.0.8, `@fontsource-variable/inter` 5.3.0,
  - `src/index.css`: wszystkie tokeny z DESIGN §6 (jasny i ciemny), `--shadow-card`/`--shadow-raised`, mapowanie na tokeny shadcn, narzędzia Tailwind (`bg-canvas`, `bg-surface`, `bg-column`, `bg-sidebar`, `text-subtle`, `bg-primary-subtle`, `bg-danger-subtle`, `shadow-card`, `shadow-raised`), zaokrąglenia 4/6/8 px, Inter z fallbackiem `system-ui`, bazowo 14 px,
  - motyw jasny/ciemny/systemowy: `src/lib/theme.ts`, `src/stores/theme.ts` (Zustand), klucz `localStorage` `streamline-theme`, skrypt w `index.html` ustawia klasę `dark` przed startem Reacta (brak migotania), tryb systemowy śledzi `prefers-color-scheme`; test sprawdza zgodność skryptu z `theme.ts` (ADR-038),
  - `AppHeader` (własne logo SVG, `ThemeSwitcher`, menu awatara: nazwa, e-mail, „Log out”); `AppLayout` jest tylko powłoką; `Breadcrumbs`, `AppToaster` (sonner, prawy dolny róg),
  - `Avatar`/`AvatarStack` (paleta 8 barw wybierana FNV-1a z id, nakładanie −8 px, maks. 4 + „+N”), `BoardIcon`, `src/lib/avatar.ts`, hook `useMediaQuery` (progi 1024 i 768 px),
  - `BoardSidebar`: 240 px, zwijany do 0 (zapis w `streamline-sidebar-collapsed`, domyślnie zwinięty poniżej 1024 px), poniżej 768 px szuflada (Radix Dialog; zamyka się przy powiększeniu okna); pozycje: Board, Analytics (Soon), Members (otwiera `MembersDialog`), Settings (tylko z `board:rename`, Soon),
  - `/boards`: okruszki, H1 „Your boards”, „Create board” otwiera okno (ten sam formularz), siatka kart `minmax(260px)`, plakietka roli, menu ⋯ (Rename/Delete), stan pusty, szkielety ładowania,
  - `/boards/:boardId`: okruszki, H1 „Board”, pasek narzędzi (pole filtra i przełącznik awatarów członków — tylko lokalny stan, filtrowanie dopiero w M3; „Sort” wyłączony), szkielety kolumn,
  - `BoardColumn` wg DESIGN §2: 280 px, nazwa wielkimi literami + licznik `(n)` / `(n / limit)`, przekroczony WIP = licznik w kolorze błędu, górna krawędź i tło `danger-subtle`; kropka kategorii; wyłączony „+” (M3); menu ⋯ (Rename/Settings/Delete); pusty stan `CircleSlash` „No cards”; przerywane „Add column”; uchwyt przeciągania to osobny `GripVertical` (ADR-040),
  - strony logowania i rejestracji wg DESIGN §5 (`AuthCard`, `AuthField` 40 px),
  - komunikaty: `notice` ze store i nieudana optymistyczna zmiana nazwy jako toasty sonner ze stałym id (ADR-039); błędy walidacji formularzy pozostają w treści formularza; menu rozwijane są niemodalne, a okna otwarte z menu zwracają fokus na przycisk ⋯ (`returnFocusRef`; ADR-041),
  - zaakceptowane odstępstwa od DESIGN: tekst w kolorze błędu używa `--destructive` (`#AE2E24` jasny / `#F87168` ciemny), bo `--danger` `#E34935` nie spełnia AA na bieli (ADR-042); brak opisu tablicy w panelu bocznym (brak pola w modelu); brak awatarów członków na kartach tablic (API nie zwraca danych); kropka TODO w `--primary`, BACKLOG w `--text-subtle`; w ciemnym motywie przyciski primary/destructive mają ciemny tekst,
  - testy: web 172 → 219 (motyw, awatary, `BoardColumn`, `BoardSidebar`, menu `AppHeader`, powrót fokusu, pojedynczy toast w StrictMode, stany puste, szkielety).
- M3a (karty, backend; WF-CARD-01 ✅, WF-CARD-03 ✅, WF-CARD-04 ✅, WF-CARD-02 — część serwerowa ✅; ADR-043–049; PR #11, zmergowane):
  - tablica: `key` (2–5 wielkich liter, generowany z nazwy: inicjały słów, transliteracja polskich i cyrylicznych liter, `BRD` gdy brak liter, kolejni kandydaci przy kolizji), `ownerId` (FK, `Restrict`) z `@@unique([ownerId, key])` oraz licznik `nextCardNumber`; `PATCH /boards/:boardId` przyjmuje `{ name?, key? }` (409 `BOARD_KEY_TAKEN`), odpowiedzi tablic zawierają `key`; `packages/shared`: `board-key.ts` (`generateBoardKey`, `boardKeyCandidates`, `pickBoardKey`), `boardKeySchema`, `renameBoardSchema`,
  - modele `Card` (numer unikalny na tablicę, `rank` jako `TEXT COLLATE "C"` z odraczalnym `UNIQUE (columnId, rank)`, enum `CardPriority`, `assigneeId` z `SetNull`, `dueDate` jako `DATE`) i `CardTransition` (klucze obce kolumn i autora `SetNull`; indeksy `(boardId, movedAt)`, `(cardId, movedAt)`, `fromColumnId`, `toColumnId`); migracja `20261112014500_cards_and_board_keys` z ręcznym uzupełnieniem istniejących tablic (PL/pgSQL; `docs/DATABASE.md`),
  - endpointy `/boards/:boardId/cards`: `GET` (karty pogrupowane w kolumny, jedna migawka `REPEATABLE READ`), `POST` (dopisanie na końcu kolumny; pierwsze przejście z `fromColumnId = null`), `GET /:cardKey` (np. `SL-12`), `PATCH /:cardId`, `PATCH /:cardId/move` (`columnId` + `afterId`/`beforeId`; przejście tylko przy zmianie kolumny, w tej samej transakcji), `DELETE /:cardId` — `docs/API.md`; uprawnienia `board:view` i `card:write`,
  - tryb ścisły WIP egzekwowany przy tworzeniu i wejściu karty do kolumny (409 `WIP_LIMIT_EXCEEDED`); usunięcie kolumny z kartami → 409 `COLUMN_NOT_EMPTY`; przypisanie wymaga członkostwa (400 `ASSIGNEE_NOT_MEMBER`), a usunięcie członka odpina go od kart w tej samej transakcji,
  - współbieżność (`docs/ARCHITECTURE.md`, ADR-043–045, ADR-049): tworzenie karty — blokada wyłączna wiersza tablicy (licznik), przeniesienie — blokada współdzielona, potem wiersz karty, potem (dla kolumny ze ścisłym WIP) wiersz kolumny; kolejność blokad: tablica → karta → kolumna; usunięcie karty i członka przechodzą przez runner transakcji (`lock_timeout` → 503 `BOARD_BUSY`),
  - refaktoryzacja: `repositories/board-transaction.ts` (runner transakcji z izolacją per wywołanie i opcjonalnym ograniczeniem rank) oraz `services/ranked-write.ts` (wspólne dla kolumn i kart: ponowienia przy kolizji, rebalansowanie, mapowanie `BOARD_BUSY`); `BadRequestError`; schematy pozycji `schemas/position.ts`; reeksporty w `column.service`/`column.repository` zachowane dla zgodności,
  - frontend: tylko poprawki kompilacji (`BoardsPage.tsx` używa `renameBoardSchema`, fixture testowy z `key`); brak nowych funkcji UI, brak zdarzeń Socket.io,
  - testy: shared 368 → 484, api 342 → 478, web 219 (bez zmian); `lint`, `typecheck`, `test` przechodzą (testy API na prawdziwej bazie).
- M3b (karty, frontend `apps/web`; WF-CARD-01 ✅, WF-CARD-02 ✅, WF-CARD-04 ✅, WF-CARD-07 częściowo; ADR-050–054), gałąź `feat/m3b-cards-frontend`, niezacommitowane:
  - `useBoardStore`: `cardsByColumn`, `cardsStatus`, `isMovingCard`, akcje `reloadCards`, `createCard`, `moveCard`, `updateCard`, `deleteCard`, `rememberCard`; karty ładują się równolegle z kolumnami w `openBoard` (szkielety); `cardsGeneration` odrzuca spóźnione listy; `removeMember` odpina osobę lokalnie, `deleteColumn` usuwa karty kolumny; 404 `COLUMN_NOT_FOUND` odświeża kolumny i karty,
  - `features/cards/`: `cardsApi`, `cardOrder`, `cardDrag`, `cardFilter`, `cardChanges`, `cardErrorMessage`, `cardLabels`, `priority.tsx`, `format.ts` (Intl en-GB), `CardTile` (tytuł do 2 linii, klucz `SL-12`, flaga priorytetu, awatar 24 px), `CardComposer` (szybkie tworzenie przez „+”: Enter tworzy, Escape zamyka, pozostaje otwarty), `CardDetailsRoute` i `CardDetailsDialog` (modal na `/boards/:boardId/cards/:cardKey`: tytuł, opis, osoba, priorytet, termin, kolumna, usuwanie z potwierdzeniem, „Copy link”, Esc; tylko do odczytu bez `card:write`),
  - `BoardColumn` pokazuje prawdziwą liczbę kart i podświetlenie przekroczonego WIP zamiast `getCardCount`; `BoardColumns`: jeden `DragDropContext` (typy `COLUMN`/`CARD`, `Droppable` `cards:<columnId>`), przeciąganie w kolumnie i między kolumnami (protokół ADR-034/035 + ADR-050); `BoardPage`: filtr po tekście i po awatarach członków (wielokrotny wybór), baner błędu kart z „Retry”, `Outlet` dla modalu; `routes.tsx` (trasa zagnieżdżona `cards/:cardKey`), `paths.ts`, `boardErrorMessage.ts` (nowe kody), `components/ui/textarea.tsx`,
  - komunikaty dla `WIP_LIMIT_EXCEEDED`, `COLUMN_NOT_EMPTY`, `ASSIGNEE_NOT_MEMBER`; kontrolki ukryte bez `card:write`; brak skrótów klawiszowych, Socket.io, etykiet, komentarzy i asystenta AI,
  - zaakceptowane odstępstwa od DESIGN (zapisane w `docs/DESIGN.md`): półprzezroczyste podświetlenie miejsca upuszczenia zamiast przerywanego prostokąta; ogólna ikona `SquareCheck` zamiast typu karty; kafelek bez terminu; `MEDIUM` w `--warning`, `URGENT` w `--destructive` + `ChevronsUp`; licznik kolumny = suma bez filtra, przeciąganie wyłączone przy aktywnym filtrze (ADR-052),
  - testy: web 218 → 326 (shared 484, api 478 bez zmian); nowe zależności, migracje, endpointy i zmienne środowiskowe: brak.
- Sprawdzone lokalnie (M3b): `lint`, `typecheck`, `test`, `format:check` i `build` — shared 484, api 478, web 326; testy API wymagają działającej bazy. Interfejsu nie sprawdzono w prawdziwej przeglądarce.
- `build` web: ostrzeżenie o fragmencie 849 kB (537 kB po M1b, 711 kB po M2d, 827 kB po M2e).

## W toku
- M3b oczekuje na commit i PR (commit tylko na wyraźną prośbę).

## Kolejne kroki
Zakres i priorytety: `docs/REQUIREMENTS.md`; plan: `docs/WORKFLOW.md`, sekcja 7.
1. **Zatwierdzenie M3b**: commit i PR.
2. **M4** — Socket.io: zdarzenia do pokoju `board:<id>` po udanej zmianie REST (karty, kolumny, członkowie), w tym zbiorcze zdarzenie po rebalansowaniu `rank`; po tym klient przestanie wymagać przeładowania, by zobaczyć zmiany innych użytkowników. Uwaga: access token bez `iss`/`aud` — uzupełnić przed uwierzytelnianiem gniazda.
3. Pozostałe z WF-CARD: etykiety i komentarze (WF-CARD-05), historia aktywności (WF-CARD-06), filtr po etykiecie (WF-CARD-07).
4. Analityka przepływu (M6) ma już dane: każde wejście karty do kolumny zapisuje `CardTransition` (ADR-020).
5. WF-BOARD-04 (przekazanie własności): akcja `board:transfer` jest w macierzy uprawnień, ale endpointu jeszcze nie ma — do zrobienia przed możliwością opuszczenia tablicy przez właściciela. Przekazanie musi zaktualizować `Board.ownerId` razem z członkostwem `OWNER` i może zakończyć się 409 `BOARD_KEY_TAKEN` (nowy właściciel ma już tablicę o tym kluczu).

### Uwagi dla kolejnych etapów
- Żądania chronione wykonywać przez instancję `api` z `apps/web/src/lib/api.ts` (obsługuje Bearer i odświeżanie); żądań do `/auth/*` nie opakowywać własnym odświeżaniem.
- Przeglądarka wywołuje `/api/auth/*` (proxy Vite); ciasteczko ma `Path=/api/auth`.

## Uruchamianie
1. Wymagane: Node 24 (`.nvmrc`), pnpm 10.32.0, Docker.
2. `pnpm install` (generuje klienta Prisma w `postinstall`).
3. Skopiować `.env.example` do `.env` (root, dla compose), `apps/api/.env.example` do `apps/api/.env`, `apps/web/.env.example` do `apps/web/.env`.
4. W `apps/api/.env` ustawić `JWT_ACCESS_SECRET` (wymagany, min. 32 znaki; API nie wystartuje bez niego). Przykład: `node -e "console.log(require('node:crypto').randomBytes(48).toString('base64url'))"`.
5. `pnpm db:up` (PostgreSQL w Dockerze).
6. `pnpm --filter @streamline/api db:migrate` — stosuje migracje (istnieją migracje `20261111010926_auth_users_refresh_tokens`, `20261111143833_boards_members`, `20261111151836_columns` i `20261112014500_cards_and_board_keys`).
7. `pnpm dev` — API na `:4000`, web na `:5173` (proxy `/api` do API); po zalogowaniu aplikacja otwiera `/boards`.

Kontrole: `pnpm lint && pnpm typecheck && pnpm test`, `pnpm format:check`, `pnpm build`. Testy API wymagają działającej bazy (`pnpm db:up`); baza `streamline_test` jest tworzona i migrowana automatycznie.

Zmienne środowiskowe uwierzytelniania (`apps/api/.env.example`): `JWT_ACCESS_SECRET`, `ACCESS_TOKEN_TTL_SECONDS` (900), `REFRESH_TOKEN_TTL_SECONDS` (604800), `REFRESH_REUSE_GRACE_SECONDS` (30), `REFRESH_COOKIE_PATH` (`/api/auth`), `BCRYPT_COST` (12), `AUTH_RATE_LIMIT_WINDOW_MS` (900000), `AUTH_RATE_LIMIT_MAX` (100), `AUTH_CREDENTIALS_RATE_LIMIT_MAX` (10), opcjonalnie `DATABASE_URL_TEST`.

## Znane problemy / dług techniczny
- Frontend: odświeżenie może rozpocząć się w trakcie `POST /auth/logout` (równoległe 401 w ciągu ok. 50 ms) i zrotować ciasteczko, zanim wylogowanie je unieważni; możliwe wzmocnienie: flaga „wylogowywanie” powodująca odrzucenie `refreshSession`.
- Frontend: `login`/`register` nie zwiększają `sessionEpoch` — odświeżenie w toku podczas logowania mogłoby nadpisać lub wyczyścić nową sesję (w obecnym UI nieosiągalne).
- Kody błędów (np. `REFRESH_TOKEN_ROTATED`) są zduplikowane w `apps/api` i `apps/web` — przenieść do `packages/shared`.
- Brak podziału kodu (jeden fragment 849 kB po M3b) — `lazy()` dla tras, w pierwszej kolejności `BoardPage` (uwaga z przeglądu M2d, nierealizowana).
- M2d: zmiany ról wprowadzone przez innych użytkowników są widoczne dopiero po przeładowaniu (do czasu Socket.io); wykrywanie rebalansowania w kliencie jest „best-effort” (ADR-035).
- M2d: mapowanie komunikatów błędów dla auth (`authErrorMessage.ts`) i tablic (`boardErrorMessage.ts`) można ujednolicić we wspólnym helperze.
- M2d/M2e/M3b: w przeglądarce nie sprawdzono wizualnie toastów, stanu przekroczonego limitu WIP, przeciągania kart, modalu szczegółów ani braku migotania motywu (sprawdzono tylko, że klasa `dark` jest ustawiona po przeładowaniu); obie poprawki z przeglądu w przeglądarce nie mają testów (układ nie jest mierzalny w jsdom).
- M2e/M3b: `components/ui/card.tsx` nadal nieużywany — usunąć.
- M2e: `aria-label` na liczniku kolumny jest zbędny (testy wyszukują po nim) — uprościć przy okazji.
- M2e/M3b: przycisk „Sort” jest wyłączony, a pozycje „Analytics” i „Settings” nieaktywne (filtr tekstowy i po osobie działają od M3b).
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
- M2c (po przeglądzie, runda 2): `ServiceUnavailableError` gubi `cause`, a `error-handler` nie loguje `AppError` — P2028 wywołany błędem w kodzie byłby cichym 503; przekazywać `cause` i logować `warn` dla `AppError` >= 500.
- M2c: `waitForLockWaiters` w `column.routes.test.ts` czeka do 3000 ms, a domyślny `lock_timeout` to 2000 ms — przy bardzo wolnym przebiegu komunikat porażki byłby mylący (503 zamiast limitu pomocnika); skrócić limit pomocnika lub podnieść `lockTimeoutMs` w testach.
- M2c: P2028 obejmuje też wolną bazę i oczekiwanie na połączenie z puli, nie tylko blokady — wszystko mapuje się na `BOARD_BUSY` (ADR-033).
- M2c: brak nagłówka `Retry-After` w odpowiedzi 503 (`AppError` nie przenosi nagłówków).
- M2c: nieaktualny widok klienta (żądania niezachodzące na siebie) kończy się 409 `INVALID_POSITION` — klient musi odświeżyć listę.
- M2c: rebalansowanie zmienia rank innych kolumn, a odpowiedź zwraca jedną kolumnę — klient odświeża listę do czasu decyzji o zbiorczym zdarzeniu socket.
- M2c: tworzenie i przeniesienie czytają całą listę kolumn tablicy (dla kolumn akceptowalne); karty (M3) mogą wymagać zapytań tylko o sąsiadów.
- M2c: schematy tablic to zwykłe `z.object` (nieznane pola są ignorowane), a schematy kolumn — `strictObject`; ujednolicić.
- M2c: wyścig przy degradacji z M2b dotyczy też kolumn; `Board.updatedAt` nie zmienia się przy zmianach kolumn; `WS_EVENTS.COLUMN_CREATED` istnieje, ale nie jest emitowane.
- M2c: istniejące tablice (sprzed migracji) nie mają domyślnych kolumn — brak uzupełnienia; nowa kolumna dostaje kategorię `TODO`, a kategorię zmienia się tylko przez `/settings`.
- M3a: generator klucza tablicy w SQL (uzupełnienie w migracji) przybliża generator z `packages/shared` (np. pomija rozszerzenia nazwy kolejnymi literami); dotyczy tylko tablic sprzed migracji.
- M3a: odpowiedź `PATCH .../cards/:id/move` zawiera jedną kartę, a rebalansowanie zmienia `rank` pozostałych kart kolumny — klient odświeża listę (jak dla kolumn).
- M3a: licznik `nextCardNumber` nie zmienia `Board.updatedAt`.
- M3a: `Board.ownerId` z `Restrict` — przy przyszłym usuwaniu kont trzeba najpierw przekazać własność lub usunąć tablice (por. uwaga M2b o kaskadzie).
- M3a: brak deterministycznego testu wyścigu migawki `GET .../cards` (gwarancję daje `REPEATABLE READ`; test sprawdza tylko poziom izolacji).
- M3a (kosmetyka): `let board` bez typu w `card.service.ts` (`list`) — dopisać `BoardCards | null`; pomocnicze funkcje testów kolumn można współdzielić z testami kart; reeksporty w `column.repository` i `column.service` zostały dla zgodności i można je usunąć.

- M3b: `@hello-pangea/dnd` nie wspiera zagnieżdżonych kontenerów przewijania (lista kart w przewijanym rzędzie kolumn): ostrzeżenie w trybie deweloperskim i brak automatycznego przewijania tablicy podczas przeciągania karty.
- M3b: dwie nakładające się nieudane edycje tego samego pola karty mogą zostawić na ekranie starszą wartość optymistyczną (poprawka: przywrócić pole lub `reloadCards()`; ADR-053); `latestCardEdits` nie jest czyszczony przy zmianie tablicy (dodać do `resetBoardStore`/`forgetOpenedBoard`).
- M3b: brak testu historii przeglądarki przy zamykaniu modalu szczegółów (`navigate(-1)` a `replace`; ADR-051).
- M3b: test przeciągania klawiaturą między kolumnami opiera się na sztucznym układzie z `apps/web/src/test/boardLayout.ts` (jsdom nie mierzy układu).
- M3b: wykrywanie rebalansowania kart po stronie klienta jest „best-effort” (jak dla kolumn, ADR-035).

## Przeszkody / otwarte pytania
- Nie potwierdzono przebiegu CI na GitHub (w tym `DATABASE_URL_TEST` z usługą PostgreSQL); sprawdzić tagi akcji (`checkout@v7`, `pnpm/action-setup@v6`, `setup-node@v7`) oraz działanie `DATABASE_URL_TEST` z usługą PostgreSQL.
- Zbiorcze zdarzenie socket po rebalansowaniu rank (kolumny i karty) — do decyzji przy realizacji Socket.io.
- `@hello-pangea/dnd`: ostatnie wydanie luty 2025 — dnd-kit jako plan awaryjny.
