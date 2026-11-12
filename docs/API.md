# API

Stan: M3a — `GET /health`, uwierzytelnianie (`/auth/*`), tablice i członkowie (`/boards/*`), kolumny (`/boards/:boardId/columns`) oraz karty (`/boards/:boardId/cards`). Kolejne endpointy dochodzą z funkcjonalnościami.

## Konwencje
- Ścieżki w tym dokumencie są podane względem korzenia API (np. `/auth/login`). Przeglądarka używa prefiksu `/api` (np. `/api/auth/login`) — serwer Vite przekazuje `/api` do API (`VITE_API_PROXY_TARGET`) i usuwa prefiks.
- Uwierzytelnianie: nagłówek `Authorization: Bearer <accessToken>` (access token JWT, 15 min). Refresh token to ciasteczko httpOnly (sekcja „Ciasteczko refresh_token”).
- Limit ciała żądania JSON: 1 MB.
- Walidacja wejścia wyłącznie schematami Zod z `packages/shared`.

### Format błędu
Schemat `errorResponseSchema` (`packages/shared`):
```json
{ "error": { "code": "NOT_FOUND", "message": "..." } }
```
Obiekt `error` może zawierać dodatkowe pola (np. szczegóły walidacji). Obsługa w `apps/api/src/middleware/error-handler.ts`:
- `AppError` (w tym `ValidationError`, `NotFoundError`) — status i kod z błędu,
- `ZodError` — błąd walidacji (400 `VALIDATION_ERROR`),
- `RankError` (`packages/shared`) — `INVALID_RANK` i `INVALID_ARGUMENT` → 400, `INVALID_RANK_ORDER` → 409 (kod błędu = kod `RankError`; błąd jest logowany jako ostrzeżenie, bo może wskazywać uszkodzone rank),
- błędy parsera ciała żądania 4xx — `BAD_REQUEST`,
- pozostałe — 500 (bez ujawniania szczegółów).

Nieistniejąca ścieżka zwraca 404 w tym samym formacie (`middleware/not-found.ts`).

## REST

### GET /health
- Uprawnienia: brak (publiczny).
- Ciało żądania: brak.
- Odpowiedź 200 (`healthResponseSchema`): `{ "status": "ok" }`
- Nie wykonuje zapytań do bazy danych.
- Przykład: `curl http://localhost:4000/health`

## Uwierzytelnianie (`/auth/*`)

Implementacja: `apps/api/src/routes/auth.routes.ts`, `controllers/auth.controller.ts`, `services/auth.service.ts`. Schematy: `packages/shared/src/schemas/auth.ts`. Odpowiedzi z tokenami mają nagłówek `Cache-Control: no-store`.

### Kody błędów uwierzytelniania
| Status | `error.code` | Kiedy |
|---|---|---|
| 400 | `VALIDATION_ERROR` | nieprawidłowe ciało żądania (schemat Zod) |
| 401 | `UNAUTHORIZED` | brak/nieprawidłowy access token; błędne dane logowania; nieprawidłowy, wygasły lub użyty ponownie refresh token |
| 401 | `REFRESH_TOKEN_ROTATED` | refresh token został wymieniony przed chwilą (patrz `/auth/refresh`) |
| 409 | `CONFLICT` | rejestracja na zajęty adres e-mail |
| 429 | `TOO_MANY_REQUESTS` | przekroczony limit żądań |

### Ograniczanie liczby żądań (express-rate-limit, według IP)
- Wszystkie `/auth/*`: `AUTH_RATE_LIMIT_MAX` (100) żądań na okno `AUTH_RATE_LIMIT_WINDOW_MS` (15 min).
- `POST /auth/login` i `POST /auth/register` (wspólny licznik): `AUTH_CREDENTIALS_RATE_LIMIT_MAX` (10) nieudanych prób na okno; udane żądania nie są liczone (`skipSuccessfulRequests`).
- Po przekroczeniu: 429 `TOO_MANY_REQUESTS`. Licznik jest w pamięci procesu.

### Ciasteczko `refresh_token`
Ustawiane przez `register`, `login`, `refresh`; usuwane przez `logout` i przy odrzuceniu tokena w `refresh`.
- `httpOnly`, `SameSite=Strict`, `Secure` tylko gdy `NODE_ENV=production`,
- `Path` = `REFRESH_COOKIE_PATH` (domyślnie `/api/auth`) — przeglądarka nie wysyła ciasteczka do pozostałych ścieżek API,
- `Max-Age` = `REFRESH_TOKEN_TTL_SECONDS` (7 dni).

### Schemat `AuthResponse` (rejestracja, logowanie, odświeżenie)
```json
{
  "accessToken": "<JWT HS256>",
  "user": { "id": "cm...", "email": "jan@example.com", "name": "Jan", "createdAt": "2026-11-11T10:00:00.000Z" }
}
```
Access token: claimy `sub` (id użytkownika), `iat`, `exp`; brak `iss`/`aud`.

### POST /auth/register
- Uprawnienia: brak. Limity: oba (ogólny + dane logowania).
- Ciało (`registerSchema`): `email` (przycinany, małe litery, poprawny e-mail, maks. 254), `password` (min. 8 znaków, maks. 72 bajty UTF-8), `name` (przycinane, 1–100 znaków).
- Odpowiedź 201: `AuthResponse` + ciasteczko `refresh_token`.
- Błędy: 400, 409 (adres zajęty), 429.
- Przykład:
```bash
curl -i -X POST http://localhost:4000/auth/register -H 'Content-Type: application/json' \
  -d '{"email":"jan@example.com","password":"tajne-haslo-123","name":"Jan"}'
```

### POST /auth/login
- Uprawnienia: brak. Limity: oba.
- Ciało (`loginSchema`): `email` (jak wyżej), `password` (1–72 bajty; minimalna długość nie jest sprawdzana, aby nie ujawniać polityki haseł).
- Odpowiedź 200: `AuthResponse` + ciasteczko `refresh_token`.
- Błędy: 400, 401 (`Invalid email or password` — identyczna odpowiedź dla nieznanego e-maila i błędnego hasła), 429.

### POST /auth/refresh
- Uprawnienia: ciasteczko `refresh_token` (bez ciała żądania). Limit: ogólny.
- Odpowiedź 200: `AuthResponse` + nowe ciasteczko (rotacja: poprzedni token zostaje unieważniony).
- Błędy 401:
  - brak ciasteczka, token nieznany lub wygasły — `UNAUTHORIZED`, ciasteczko usuwane;
  - token unieważniony przez rotację w oknie łaski (`REFRESH_REUSE_GRACE_SECONDS`, domyślnie 30 s) lub przegrany wyścig równoległej rotacji — `REFRESH_TOKEN_ROTATED`; **ciasteczko nie jest usuwane**, nie są unieważniane inne tokeny; klient powinien chwilę poczekać i ponowić odświeżenie (raz);
  - ponowne użycie tokena po oknie łaski albo tokena unieważnionego przez wylogowanie/masowe unieważnienie — `UNAUTHORIZED`; unieważniane są wszystkie refresh tokeny użytkownika, ciasteczko usuwane, zdarzenie logowane (warn).
- `REFRESH_REUSE_GRACE_SECONDS=0` wyłącza okno łaski (tryb ścisły).

### POST /auth/logout
- Uprawnienia: ciasteczko `refresh_token` (opcjonalne). Limit: ogólny.
- Odpowiedź 204 bez ciała; ciasteczko usuwane. Idempotentne — brak lub nieznany token nie powoduje błędu.

### GET /auth/me
- Uprawnienia: access token (`requireAuth`). Limit: ogólny.
- Odpowiedź 200 (`meResponseSchema`): `{ "user": { id, email, name, createdAt } }`.
- Błędy: 401 (brak/nieprawidłowy/wygasły token, użytkownik nie istnieje).
- Przykład: `curl http://localhost:4000/auth/me -H "Authorization: Bearer $TOKEN"`

### Middleware `requireAuth`
`apps/api/src/middleware/require-auth.ts`: wymaga nagłówka `Authorization: Bearer <token>`; weryfikuje podpis (jose, dozwolony tylko algorytm HS256) oraz claimy `sub` i `exp`; ustawia `req.userId`. Każdy błąd daje 401 `UNAUTHORIZED`.

## Tablice i członkowie (`/boards/*`)

Implementacja: `apps/api/src/routes/board.routes.ts`, `controllers/board.controller.ts`, `services/board.service.ts`, `services/board-access.service.ts`, `middleware/require-board-permission.ts`. Schematy i uprawnienia: `packages/shared/src/schemas/board.ts`, `permissions.ts`. Wszystkie trasy wymagają access tokena (`requireAuth`).

### Role i uprawnienia
Role: `OWNER`, `ADMIN`, `MEMBER`, `VIEWER`. Macierz akcji (`BOARD_PERMISSIONS`):

| Akcja | OWNER | ADMIN | MEMBER | VIEWER |
|---|---|---|---|---|
| `board:view` | tak | tak | tak | tak |
| `board:rename`, `members:manage`, `column:settings` | tak | tak | nie | nie |
| `column:write`, `card:write`, `ai:use` | tak | tak | tak | nie |
| `board:delete`, `board:transfer` | tak | nie | nie | nie |

Akcje `column:write` i `column:settings` mają endpointy od M2c (sekcja „Kolumny”). Akcja `card:write` ma endpointy od M3a (sekcja „Karty”). Akcje `ai:use` i `board:transfer` są zdefiniowane, ale nie mają jeszcze endpointów.

### Zasady dostępu i kody błędów
- Użytkownik niebędący członkiem tablicy dostaje 404 `NOT_FOUND` („Board not found”) — identycznie jak dla nieistniejącej tablicy (nie ujawniamy istnienia; ADR-026).
- Członek bez wymaganej roli dostaje 403 `FORBIDDEN`. Rola jest sprawdzana przed walidacją ciała żądania.
- Rola `OWNER` nie może być nadana przez API (ciało z `role: "OWNER"` daje 400 `VALIDATION_ERROR`); właściciel jest ustawiany przy tworzeniu tablicy.
- `ADMIN` może zarządzać innymi `ADMIN` oraz zmienić własną rolę na niższą; nikt nie może zmienić ani usunąć `OWNER` (403).

| Status | `error.code` | Kiedy |
|---|---|---|
| 400 | `VALIDATION_ERROR` | nieprawidłowe ciało żądania |
| 401 | `UNAUTHORIZED` | brak/nieprawidłowy access token |
| 403 | `FORBIDDEN` | rola nie pozwala na akcję; zmiana lub usunięcie `OWNER` |
| 404 | `NOT_FOUND` | tablica nie istnieje lub brak członkostwa; członek nie istnieje |
| 404 | `USER_NOT_FOUND` | brak konta o podanym e-mailu (dodawanie członka) |
| 409 | `ALREADY_MEMBER` | użytkownik już należy do tablicy (także przy równoległych żądaniach) |
| 409 | `OWNER_CANNOT_LEAVE` | właściciel próbuje opuścić tablicę |
| 409 | `BOARD_KEY_TAKEN` | inna tablica tego samego właściciela ma już podany klucz (`PATCH /boards/:boardId`) lub nie udało się wybrać wolnego klucza przy tworzeniu (równoległe żądania) |
| 503 | `BOARD_BUSY` | usunięcie członka nie uzyskało blokad w limicie czasu (jak dla kolumn, ADR-033) |

### Obiekty odpowiedzi
- `Board`: `{ id, name, key, createdAt, updatedAt, role }` — `role` to rola wywołującego; `key` to prefiks kluczy kart (2–5 wielkich liter, np. `SL`), unikalny w obrębie właściciela tablicy.
- `BoardMember`: `{ userId, name, email, role, joinedAt }`.

### GET /boards
- Uprawnienia: zalogowany. Odpowiedź 200: `{ "boards": Board[] }` — tablice wywołującego, od najnowszej (`createdAt` malejąco, remis rozstrzyga `id`).

### POST /boards
- Uprawnienia: zalogowany. Ciało (`createBoardSchema`): `name` (przycinane, 1–100 znaków).
- Odpowiedź 201: `{ "board": Board }` z `role: "OWNER"`; twórca zostaje właścicielem. Klucz tablicy jest generowany z nazwy (inicjały słów; jedno słowo — dwie pierwsze litery; polskie i cyrylickie litery są transliterowane; `BRD` bez liter; przy zajętym kluczu następny kandydat, ADR-046). Przykłady: `StreamLine Sprint` → `SS`, `Roadmap` → `RO`, `42` → `BRD`.
- Błędy: 400, 401.
```bash
curl -X POST http://localhost:4000/boards -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' -d '{"name":"Sprint 1"}'
```

### GET /boards/:boardId
- Uprawnienia: `board:view` (każdy członek). Odpowiedź 200: `{ "board": Board }`. Błędy: 401, 404.

### PATCH /boards/:boardId
- Uprawnienia: `board:rename` (OWNER, ADMIN). Ciało (`updateBoardSchema`, ścisłe): `name` (1–100 znaków) i/lub `key` (przycinany i zamieniany na wielkie litery, 2–5 liter A–Z); co najmniej jedno pole. Zmiana klucza zmienia klucze wszystkich kart tablicy (numery zostają); dotychczasowe klucze kart przestają działać (404).
- Odpowiedź 200: `{ "board": Board }`. Błędy: 400, 401, 403, 404, 409 `BOARD_KEY_TAKEN`.
```bash
curl -X PATCH http://localhost:4000/boards/$BOARD_ID -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' -d '{"key":"SL"}'
```

### DELETE /boards/:boardId
- Uprawnienia: `board:delete` (OWNER). Odpowiedź 204 bez ciała; członkostwa są usuwane kaskadowo. Błędy: 401, 403, 404.

### GET /boards/:boardId/members
- Uprawnienia: `board:view`. Odpowiedź 200: `{ "members": BoardMember[] }` — według kolejności roli (`OWNER`, `ADMIN`, `MEMBER`, `VIEWER`), potem daty dołączenia.

### POST /boards/:boardId/members
- Uprawnienia: `members:manage` (OWNER, ADMIN). Ciało (`addBoardMemberSchema`): `email` (jak w rejestracji; użytkownik musi mieć konto), `role` (`ADMIN` | `MEMBER` | `VIEWER`, domyślnie `MEMBER`).
- Odpowiedź 201: `{ "member": BoardMember }`.
- Błędy: 400 (m.in. `role: "OWNER"`), 401, 403, 404 (`NOT_FOUND`, `USER_NOT_FOUND`), 409 `ALREADY_MEMBER`.
```bash
curl -X POST http://localhost:4000/boards/$BOARD_ID/members -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' -d '{"email":"anna@example.com","role":"VIEWER"}'
```

### PATCH /boards/:boardId/members/:userId
- Uprawnienia: `members:manage`. Ciało (`updateBoardMemberSchema`): `role` (`ADMIN` | `MEMBER` | `VIEWER`).
- Odpowiedź 200: `{ "member": BoardMember }`. Zmiana jest warunkowa (`role <> OWNER`), więc `OWNER` nie zostanie zmodyfikowany nawet przy równoległych żądaniach.
- Błędy: 400, 401, 403 (cel to `OWNER`, brak roli), 404 (tablica lub członek).

### DELETE /boards/:boardId/members/:userId
- Uprawnienia: każdy członek (`board:view`) w przypadku usunięcia siebie (opuszczenie tablicy); usunięcie innej osoby wymaga `members:manage`.
- Odpowiedź 204 bez ciała. Usunięcie członka odpina go od przypisanych kart tablicy w tej samej transakcji (`assigneeId` = `null`, ADR-045).
- Błędy: 401, 403 (usunięcie innej osoby bez uprawnień lub usunięcie `OWNER`), 404, 409 `OWNER_CANNOT_LEAVE` (właściciel usuwa siebie), 503 `BOARD_BUSY`.

### Middleware `requireBoardPermission`
`requireBoardPermission(action, { param })` (domyślnie `param = 'boardId'`) działa po `requireAuth`: wczytuje członkostwo wywołującego, zwraca 404 dla nie-członka i 403 przy braku uprawnienia, a członkostwo zapisuje w `res.locals.boardMembership` (odczyt: `getBoardMembership(res)`). Trasy zagnieżdżone wymagają `Router({ mergeParams: true })`. Serwis dodatkowo sprawdza uprawnienia samodzielnie (ADR-025).

## Kolumny (`/boards/:boardId/columns`)

Implementacja: `apps/api/src/routes/column.routes.ts` (montowane w `board.routes.ts`, `Router({ mergeParams: true })`), `controllers/column.controller.ts`, `services/column.service.ts`, `repositories/column.repository.ts`. Schematy i kody błędów: `packages/shared/src/schemas/column.ts`. Wszystkie trasy wymagają access tokena i członkostwa w tablicy (nie-członek: 404 `NOT_FOUND`, brak roli: 403 `FORBIDDEN`; rola jest sprawdzana przed walidacją ciała). Ciała żądań są ścisłe (`strictObject`): nieznane pola dają 400 `VALIDATION_ERROR`.

### Obiekt `Column`
`{ id, boardId, title, rank, category, wipLimit, wipStrict, createdAt, updatedAt }`:
- `rank` — klucz porządkujący (LexoRank, `packages/shared/src/rank/`); porównywać przez `<`, nigdy `localeCompare`. Klient nie wysyła klucza — serwer liczy go z zapisanych sąsiadów.
- `category` — `BACKLOG` | `TODO` | `IN_PROGRESS` | `DONE`; `wipLimit` — liczba całkowita 1–999 albo `null` (brak limitu); `wipStrict` — `true`: limit blokuje dodawanie kart do pełnej kolumny (409 `WIP_LIMIT_EXCEEDED` przy tworzeniu i przenoszeniu karty, ADR-044), `false`: tylko ostrzega.
- `title` — przycinany, 1–100 znaków.

### Pozycja względem sąsiadów (`afterId`, `beforeId`)
Dotyczy `POST` i `PATCH .../move`. `afterId` to kolumna, która ma BEZPOŚREDNIO POPRZEDZAĆ umieszczaną kolumnę (`null` = na początek); `beforeId` to kolumna, która ma BEZPOŚREDNIO PO NIEJ NASTĘPOWAĆ (`null` = na koniec); pominięte pole = brak ograniczenia z tej strony.
- Przy jednym sąsiedzie drugim jest ten, który aktualnie leży obok niego. Bez żadnego (`POST`) kolumna trafia na koniec; `PATCH .../move` wymaga co najmniej jednego pola.
- Przy obu sąsiadach muszą być obecnie przyległe (z pominięciem przenoszonej kolumny), inaczej 409 `INVALID_POSITION` (nieaktualny widok klienta — odświeżyć listę).
- Wyjątek: po kolizji klucza z równoległym żądaniem (ta sama luka) ponowienie wymaga tylko, by `afterId` leżał przed `beforeId`, i umieszcza kolumnę bezpośrednio po `afterId`; dzięki temu dwa równoległe przeniesienia do tej samej luki kończą się sukcesem.
- Sąsiad spoza tablicy lub usunięty w międzyczasie: 404 `NEIGHBOR_NOT_FOUND`. `afterId` i `beforeId` równe sobie albo równe przenoszonej kolumnie: 400.
- Przeniesienie na aktualną pozycję nic nie zmienia i zwraca kolumnę bez zapisu.

### Kody błędów kolumn
| Status | `error.code` | Kiedy |
|---|---|---|
| 400 | `VALIDATION_ERROR` | nieprawidłowe ciało (m.in. brak pól, nieznane pole, `afterId` = `beforeId`, brak ustawień w `/settings`, kolumna sama swoim sąsiadem) |
| 400 | `INVALID_RANK`, `INVALID_ARGUMENT` | `RankError` z funkcji rank (nie powinno wystąpić przy poprawnych danych) |
| 401 | `UNAUTHORIZED` | brak/nieprawidłowy access token |
| 403 | `FORBIDDEN` | rola nie pozwala (`column:write`: OWNER, ADMIN, MEMBER; `column:settings`: OWNER, ADMIN) |
| 404 | `NOT_FOUND` | tablica nie istnieje lub brak członkostwa; kolumna nie należy do tablicy |
| 404 | `NEIGHBOR_NOT_FOUND` | `afterId`/`beforeId` nie jest kolumną tej tablicy |
| 409 | `INVALID_POSITION` | `afterId` i `beforeId` nie są przyległe (nieaktualny widok) |
| 409 | `RANK_CONFLICT` | 5 prób przypisania klucza zakończyło się kolizją przy równoległych zmianach; żądanie można ponowić |
| 409 | `LAST_DONE_COLUMN` | usunięcie lub zmiana kategorii ostatniej kolumny `DONE` |
| 409 | `COLUMN_NOT_EMPTY` | usunięcie kolumny, w której są karty (najpierw przenieść lub usunąć karty; ADR-047) |
| 409 | `INVALID_RANK_ORDER` | `RankError`: niemożliwa kolejność kluczy |
| 503 | `BOARD_BUSY` | transakcja zapisu kolumn nie uzyskała blokad lub połączenia w limicie czasu (`lock_timeout` 2 s, `maxWait` 2 s, `timeout` 5 s; ADR-033); serwer nie ponawia, klient może ponowić żądanie po chwili. Dotyczy tworzenia, przeniesienia, zmiany ustawień i usunięcia; zmiana nazwy nie używa transakcji. Bez nagłówka `Retry-After` |

### GET /boards/:boardId/columns
- Uprawnienia: `board:view`. Odpowiedź 200: `{ "columns": Column[] }` w kolejności rank. Błędy: 401, 404.

### POST /boards/:boardId/columns
- Uprawnienia: `column:write`. Ciało (`createColumnSchema`): `title`, opcjonalnie `afterId`, `beforeId` (zasady powyżej; domyślnie na koniec).
- Odpowiedź 201: `{ "column": Column }`; nowa kolumna ma `category: "TODO"`, `wipLimit: null`, `wipStrict: false` (kategorię zmienia się przez `/settings`).
- Błędy: 400, 401, 403, 404 (`NOT_FOUND`, `NEIGHBOR_NOT_FOUND`), 409 (`INVALID_POSITION`, `RANK_CONFLICT`).
```bash
curl -X POST http://localhost:4000/boards/$BOARD_ID/columns -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' -d '{"title":"Review","afterId":"$IN_PROGRESS_ID"}'
```

### PATCH /boards/:boardId/columns/:columnId
- Uprawnienia: `column:write`. Ciało (`renameColumnSchema`): `title`. Odpowiedź 200: `{ "column": Column }`. Błędy: 400, 401, 403, 404.

### PATCH /boards/:boardId/columns/:columnId/move
- Uprawnienia: `column:write`. Ciało (`moveColumnSchema`): `afterId` i/lub `beforeId` (co najmniej jedno; `null` dozwolone).
- Odpowiedź 200: `{ "column": Column }` — tylko przenoszona kolumna. Jeśli w trakcie nastąpiło rebalansowanie, rank innych kolumn też się zmieniły, a odpowiedź ich nie zawiera — klient powinien odświeżyć listę.
- Błędy: 400, 401, 403, 404 (`NOT_FOUND`, `NEIGHBOR_NOT_FOUND`), 409 (`INVALID_POSITION`, `RANK_CONFLICT`).
```bash
curl -X PATCH http://localhost:4000/boards/$BOARD_ID/columns/$COLUMN_ID/move -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' -d '{"afterId":null}'   # na początek
```

### PATCH /boards/:boardId/columns/:columnId/settings
- Uprawnienia: `column:settings` (OWNER, ADMIN). Ciało (`updateColumnSettingsSchema`): `category`, `wipLimit` (1–999 lub `null` = bez limitu), `wipStrict`; co najmniej jedno pole.
- Odpowiedź 200: `{ "column": Column }`. Zmiana kategorii ostatniej kolumny `DONE` na inną: 409 `LAST_DONE_COLUMN`.
- Błędy: 400, 401, 403, 404, 409.

### DELETE /boards/:boardId/columns/:columnId
- Uprawnienia: `column:write`. Odpowiedź 204 bez ciała. Usunięcie ostatniej kolumny `DONE`: 409 `LAST_DONE_COLUMN`; kolumna z kartami: 409 `COLUMN_NOT_EMPTY`. Inwariant „co najmniej jedna kolumna `DONE`” jest chroniony blokadą wyłączną wiersza tablicy (ADR-029), więc równoległe usunięcia dwóch kolumn `DONE` nie usuną obu.
- Błędy: 401, 403, 404, 409.

### Domyślne kolumny
`POST /boards` tworzy w tej samej transakcji 4 kolumny: Backlog (`BACKLOG`), To Do (`TODO`), In Progress (`IN_PROGRESS`), Done (`DONE`) z równomiernie rozłożonymi kluczami. Tablice utworzone przed M2c nie są uzupełniane.

## Karty (`/boards/:boardId/cards`)

Implementacja: `apps/api/src/routes/card.routes.ts` (montowane w `board.routes.ts`, `Router({ mergeParams: true })`), `controllers/card.controller.ts`, `services/card.service.ts`, `repositories/card.repository.ts`. Schematy i kody błędów: `packages/shared/src/schemas/card.ts`. Wszystkie trasy wymagają access tokena i członkostwa w tablicy (nie-członek: 404 `NOT_FOUND`, brak roli: 403 `FORBIDDEN`; rola jest sprawdzana przed walidacją ciała). Odczyt wymaga `board:view` (każdy członek, także `VIEWER`), zapis `card:write` (OWNER, ADMIN, MEMBER). Ciała żądań są ścisłe (`strictObject`): nieznane pola dają 400 `VALIDATION_ERROR`. Karta należy do tablicy z URL: `cardId` z innej tablicy daje 404 `NOT_FOUND`.

### Obiekt `Card`
`{ id, key, number, boardId, columnId, title, description, rank, priority, assigneeId, dueDate, createdAt, updatedAt }`:
- `key` — `<klucz tablicy>-<numer>`, np. `SL-12`; składany przy odczycie z aktualnego klucza tablicy; `number` — kolejny numer w tablicy, nigdy niepowtarzany (ADR-043, ADR-046),
- `title` — przycinany, 1–200 znaków; `description` — przycinany, do 10 000 znaków; pusty lub złożony z samych spacji zapisywany jako `null`,
- `priority` — `LOW` | `MEDIUM` | `HIGH` | `URGENT` (domyślnie `MEDIUM`),
- `assigneeId` — id członka tablicy lub `null`; `dueDate` — data kalendarzowa `YYYY-MM-DD` (bez godziny i strefy) lub `null`,
- `rank` — klucz porządkujący w obrębie kolumny (LexoRank); porównywać przez `<`, nigdy `localeCompare`; klient go nie wysyła.

W `PATCH` i `POST`: `null` czyści opis, osobę i termin; pole pominięte pozostaje bez zmian (`PATCH`) lub dostaje wartość domyślną (`POST`).

### Pozycja karty (`afterId`, `beforeId`)
Dotyczy `PATCH .../move` (nowa karta zawsze trafia na koniec kolumny). Zasady jak dla kolumn („Pozycja względem sąsiadów”), z tą różnicą, że sąsiadami są karty KOLUMNY DOCELOWEJ: `afterId` poprzedza przenoszoną kartę (`null` = góra kolumny), `beforeId` po niej następuje (`null` = dół); bez sąsiadów karta trafia na dół kolumny. Nieprzyległa para: 409 `INVALID_POSITION` (odświeżyć listę); sąsiad spoza kolumny docelowej: 404 `NEIGHBOR_NOT_FOUND`; karta jako własny sąsiad: 400. Przeniesienie na bieżącą pozycję nic nie zapisuje.

### Kody błędów kart
| Status | `error.code` | Kiedy |
|---|---|---|
| 400 | `VALIDATION_ERROR` | nieprawidłowe ciało lub parametr (m.in. brak pól w `PATCH`, nieznane pole, zła data, `afterId` = `beforeId`, karta jako własny sąsiad); niepoprawny format klucza w `GET /:cardKey` (np. `sl12`, `S-1`, `SL-0`, numer powyżej 2 147 483 647) |
| 400 | `ASSIGNEE_NOT_MEMBER` | `assigneeId` nie jest członkiem tej tablicy |
| 401 | `UNAUTHORIZED` | brak/nieprawidłowy access token |
| 403 | `FORBIDDEN` | rola nie pozwala (`card:write` nie ma `VIEWER`) |
| 404 | `NOT_FOUND` | tablica nie istnieje lub brak członkostwa; karta nie należy do tablicy; klucz karty z innym prefiksem niż aktualny klucz tablicy albo nieistniejący numer |
| 404 | `COLUMN_NOT_FOUND` | `columnId` nie jest kolumną tej tablicy |
| 404 | `NEIGHBOR_NOT_FOUND` | `afterId`/`beforeId` nie jest kartą kolumny docelowej |
| 409 | `INVALID_POSITION` | `afterId` i `beforeId` nie są przyległe (nieaktualny widok) |
| 409 | `RANK_CONFLICT` | 5 prób przypisania klucza zakończyło się kolizją przy równoległych zmianach; żądanie można ponowić |
| 409 | `WIP_LIMIT_EXCEEDED` | kolumna docelowa ma ścisły limit WIP i jest pełna (tworzenie lub przeniesienie do innej kolumny; zmiana kolejności w tej samej kolumnie nie sprawdza limitu) |
| 409 | `INVALID_RANK_ORDER` | `RankError`: niemożliwa kolejność kluczy |
| 503 | `BOARD_BUSY` | transakcja nie uzyskała blokad lub połączenia w limicie czasu (ADR-033); dotyczy też usunięcia karty i odczytu listy; klient może ponowić |

### GET /boards/:boardId/cards
- Uprawnienia: `board:view`. Odpowiedź 200: `{ "columns": [{ "columnId": string, "cards": Card[] }] }` — wszystkie kolumny tablicy (także puste) w kolejności rank, w nich karty w kolejności rank. Klucz tablicy, kolumny i karty pochodzą z jednej migawki (`REPEATABLE READ`, ADR-049). Błędy: 401, 404, 503.
```bash
curl http://localhost:4000/boards/$BOARD_ID/cards -H "Authorization: Bearer $TOKEN"
```

### POST /boards/:boardId/cards
- Uprawnienia: `card:write`. Ciało (`createCardSchema`): `columnId`, `title`; opcjonalnie `description`, `priority`, `assigneeId`, `dueDate`.
- Odpowiedź 201: `{ "card": Card }` — karta dopisana na końcu kolumny z kolejnym numerem tablicy; w tej samej transakcji zapisywane jest pierwsze `CardTransition` (`fromColumnId: null`, ADR-047).
- Błędy: 400 (`VALIDATION_ERROR`, `ASSIGNEE_NOT_MEMBER`), 401, 403, 404 (`NOT_FOUND`, `COLUMN_NOT_FOUND`), 409 (`WIP_LIMIT_EXCEEDED`, `RANK_CONFLICT`), 503.
```bash
curl -X POST http://localhost:4000/boards/$BOARD_ID/cards -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"columnId":"$COLUMN_ID","title":"Fix login","priority":"HIGH","dueDate":"2026-12-01"}'
```

### GET /boards/:boardId/cards/:cardKey
- Uprawnienia: `board:view`. `:cardKey` to klucz karty (`SL-12`). Odpowiedź 200: `{ "card": Card }`.
- Błędy: 400 (niepoprawny format klucza), 401, 404 (prefiks inny niż aktualny klucz tablicy — stare klucze nie są przekierowywane — lub brak karty o tym numerze).

### PATCH /boards/:boardId/cards/:cardId
- Uprawnienia: `card:write`. Ciało (`updateCardSchema`): `title`, `description`, `priority`, `assigneeId`, `dueDate` (co najmniej jedno pole; `null` czyści opis, osobę i termin). Kolumna i pozycja zmieniają się tylko przez `/move`.
- Odpowiedź 200: `{ "card": Card }`. Błędy: 400 (`VALIDATION_ERROR`, `ASSIGNEE_NOT_MEMBER`), 401, 403, 404, 503.

### PATCH /boards/:boardId/cards/:cardId/move
- Uprawnienia: `card:write`. Ciało (`moveCardSchema`): `columnId` (kolumna docelowa; może być bieżąca — zmiana kolejności), opcjonalnie `afterId` i/lub `beforeId` (zasady powyżej).
- Odpowiedź 200: `{ "card": Card }` — tylko przenoszona karta. Jeśli nastąpiło rebalansowanie, `rank` pozostałych kart kolumny też się zmieniły, a odpowiedź ich nie zawiera — klient powinien odświeżyć listę. Przy zmianie kolumny w tej samej transakcji zapisywane jest `CardTransition` (`fromColumnId` → `toColumnId`, autor = wywołujący); zmiana kolejności w kolumnie i przeniesienie na bieżącą pozycję przejścia nie zapisują.
- Błędy: 400, 401, 403, 404 (`NOT_FOUND`, `COLUMN_NOT_FOUND`, `NEIGHBOR_NOT_FOUND`), 409 (`INVALID_POSITION`, `RANK_CONFLICT`, `WIP_LIMIT_EXCEEDED`), 503.
```bash
curl -X PATCH http://localhost:4000/boards/$BOARD_ID/cards/$CARD_ID/move -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' -d '{"columnId":"$DONE_ID","afterId":null}'   # na górę kolumny Done
```

### DELETE /boards/:boardId/cards/:cardId
- Uprawnienia: `card:write`. Odpowiedź 204 bez ciała; przejścia karty są usuwane kaskadowo, numer karty nie jest ponownie używany. Błędy: 401, 403, 404, 503 `BOARD_BUSY` (karta zablokowana przez wolniejsze przeniesienie; karta zostaje).

## Zdarzenia Socket.io
Brak zaimplementowanych zdarzeń (endpointy tablic, członków, kolumn i kart niczego nie rozgłaszają; zdarzenie dla rebalansowania rank kolumn i kart — do ustalenia). W `packages/shared` zdefiniowano wstępną stałą `WS_EVENTS` (`board:updated`, `column:created`, `card:moved`) oraz funkcję `boardRoom()` (pokój `board:<id>`). Nazwy są robocze i zostaną zatwierdzone razem z funkcjonalnościami.
