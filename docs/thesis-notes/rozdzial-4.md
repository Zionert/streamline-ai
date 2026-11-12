# Rozdział 4 — Implementacja (notatki)

## 4.x Szkielet projektu (M0)
- Monorepo pnpm (`apps/web`, `apps/api`, `packages/shared`) — jeden pakiet kontraktów (schematy Zod) współdzielony przez frontend i backend (ADR-001).
- Pakiet `shared` jest konsumowany ze źródeł dzięki niestandardowemu warunkowi w polu `exports` (warunek `@streamline/source` wskazuje `src/index.ts`). Skraca to pętlę deweloperską (brak kompilacji pośredniej); `dist/` powstaje tylko dla produkcyjnego buildu API (ADR-005).
- Backend: warstwy Controller → Service → Repository; fabryka `createApp({ config, logger })` oddzielona od `server.ts`, co umożliwia testy Supertest bez nasłuchiwania na porcie. Spójny format błędów `{ error: { code, message } }` zdefiniowany schematem Zod.
- Dostęp do bazy: Prisma 7 z adapterem `@prisma/adapter-pg`, klient generowany do `apps/api/src/generated/prisma` i niewersjonowany (ADR-008).
- Wersje zależności przypięte dokładnie; TypeScript 6.0.3 ze względu na ograniczenie `typescript-eslint` (ADR-007).
- Weryfikacja `@hello-pangea/dnd` 18.0.1 z React 19.3.1: test dymny `DragDropContext`/`Droppable`/`Draggable` bez błędów w konsoli (ADR-009).

## 4.x Uwierzytelnianie (M1a)
- Modele `User` i `RefreshToken` (Prisma), migracja `20261111010926_auth_users_refresh_tokens`. E-mail zapisywany znormalizowany (przycięty, małe litery), unikalny.
- Hasła: `bcryptjs` 3.0.3, koszt 12 (w produkcji minimum 12); wejście ograniczone do 72 bajtów UTF-8, ponieważ bcrypt obcina dłuższe hasła (ADR-010).
- Access token: JWT HS256 (`jose`), 15 minut, claimy `sub`, `iat`, `exp`; weryfikacja z listą dozwolonych algorytmów (ADR-011):
```ts
jwtVerify(token, secret, { algorithms: ['HS256'], requiredClaims: ['sub', 'exp'] })
```
- Refresh token: 32 losowe bajty (`randomBytes`, base64url), 7 dni; w bazie wyłącznie skrót SHA-256, więc wyciek bazy nie ujawnia użytecznych tokenów. Przekazywany w ciasteczku `refresh_token` (`httpOnly`, `SameSite=Strict`, `Secure` w produkcji, `Path=/api/auth`) — ADR-012.
- Rotacja: `RefreshTokenRepository.rotate` w jednej transakcji unieważnia stary token (`updateMany` z warunkiem `revokedAt: null`) i tworzy nowy; wynik `count === 0` oznacza przegraną równoległą rotację.
- Warstwa API: `createApp({ config, logger, db })` z ręcznym wstrzykiwaniem zależności; walidacja schematami `shared` bezpośrednio w kontrolerach (ADR-014); `requireAuth` ustawia `req.userId`.
- Logi pino: redakcja nagłówków `authorization`, `cookie`, `set-cookie` oraz pól `password` i tokenów.
- Ograniczanie liczby żądań (express-rate-limit): 100 żądań/15 min na wszystkie `/auth/*` oraz 10 nieudanych prób logowania/rejestracji/15 min (`skipSuccessfulRequests`).

## 4.x Wykrywanie ponownego użycia refresh tokena
Rotacja pozwala wykryć kradzież: jeśli ktoś przedstawi token już wymieniony, token mógł zostać skopiowany. Reakcja zależy od czasu i rodzaju unieważnienia:
- token wymieniony w ciągu `REFRESH_REUSE_GRACE_SECONDS` (domyślnie 30 s) albo przegrany wyścig rotacji — odpowiedź 401 `REFRESH_TOKEN_ROTATED`; nie unieważnia się innych tokenów, ciasteczko zostaje (zwycięska odpowiedź ustawiła już nowe). Obsługuje to uzasadniony przypadek wielu kart przeglądarki odświeżających sesję jednocześnie;
- ponowne użycie po oknie lub tokena unieważnionego wylogowaniem — unieważnienie wszystkich tokenów użytkownika (`revokeAllForUser`), 401 `UNAUTHORIZED`, usunięcie ciasteczka, wpis ostrzegawczy w logu.

Rozróżnienie opiera się na polach `revokedAt` i `replacedById` (ADR-013). Kompromis do omówienia w pracy: jeśli atakujący odświeży skradziony token jako pierwszy, a klient użyje starego tokena w oknie łaski, wykrycie kradzieży jest opóźnione do kolejnego użycia po oknie; okno 0 s daje tryb ścisły kosztem fałszywych alarmów.

## 4.x Wnioski dotyczące bezpieczeństwa (kompromisy)
- **Wyliczanie kont:** logowanie zwraca identyczną odpowiedź 401 dla nieznanego e-maila i błędnego hasła, a dla nieistniejącego konta wykonywane jest porównanie z atrapą hasha (wyrównanie czasu, ochrona przed atakiem czasowym). Natomiast `409 CONFLICT` przy rejestracji ujawnia istnienie konta; zaakceptowano to ze względu na użyteczność formularza i ograniczenie limitem nieudanych prób (10/15 min).
- **Polityka haseł:** schemat logowania nie sprawdza minimalnej długości, aby nie ujawniać polityki.
- **Ograniczenia znane:** licznik limitów w pamięci procesu, brak `trust proxy`, brak `iss`/`aud` w JWT, brak czyszczenia wygasłych tokenów (`docs/PROJECT_STATE.md`).
- Odpowiedzi z tokenami mają `Cache-Control: no-store`.

## 4.x Uwierzytelnianie we frontendzie (M1b)
- Stos: Zustand 5.1.0, axios 1.21.1, react-hook-form 7.91.0 z `@hookform/resolvers` 7.0.5, shadcn/ui.
- `createApiClient` jest fabryką z wstrzykiwanymi zależnościami (`adapter`, `sleep`, `rotatedRetryDelayMs`), co umożliwia deterministyczne testy. Jedno wspólne odświeżenie realizuje współdzielona obietnica `refreshInFlight`; limit jednego powtórzenia żądania zapewnia znacznik `authRetried`.
- Odporność na wyścigi: `sessionEpoch` (zwiększany w `clearSession`) pozwala odrzucić nieaktualny wynik odświeżenia; `logout` oczekuje na `waitForPendingRefresh`. Błędy sieci, 5xx i 429 nie kończą sesji, w odróżnieniu od 401 lub niepoprawnej odpowiedzi.
- Przywracanie sesji: `restoreSession` działa tylko dla stanu `idle` i natychmiast przechodzi w `loading`, więc podwójne wywołanie efektu w StrictMode nie powoduje drugiej rotacji tokena.
- Obsługa błędów formularzy: komunikaty dla 401, 409, 400 (tekst serwera), 429, błędu sieci i ogólnego; pola z `aria-invalid` i `aria-describedby`.
- Znane ograniczenia: patrz „Znane problemy” w `docs/PROJECT_STATE.md` (m.in. odświeżenie w trakcie wylogowania, brak podziału kodu — fragment 537 kB).

## 4.1.2 / 4.3.2 Algorytm kolejności elementów — LexoRank (M2a)
- **Reprezentacja.** Klucz to ułamek zapisany cyframi systemu 62-cyfrowego (`0-9A-Za-z`, rosnące kody ASCII): `0.d1d2…`. Klucz jest niepusty i nie kończy się na `0`, więc każda wartość ma jedną postać, a porządek leksykograficzny odpowiada liczbowemu. Każdy klucz jest większy od 0 — zawsze istnieje miejsce przed pierwszym elementem. Klucz początkowy: `V` (0,5).
- **Porównywanie.** Operator `<` w JavaScript, porządek bajtów UTF-8 i `COLLATE "C"` w PostgreSQL dają ten sam wynik; kollacje językowe (ICU, `localeCompare`) porządkują mieszaną wielkość liter inaczej, co dowodzi test `collation.test.ts`.
- **`rankBetween(a, b)`.** Przechodzi po cyfrach obu granic (`null` = granica otwarta) i zwraca najkrótszy klucz leżący w przedziale (zgodność z wyrocznią BigInt). Przy obu granicach wybiera kandydata środkowego, przy jednej — najbliższego sąsiadowi (`rankAfter`: zwiększenie pierwszej cyfry mającej zapas, np. `V`→`W`, `z`→`z1`), dzięki czemu dopisywanie na końcu wydłuża klucz o jeden znak na ok. 61 wstawień. `rankSequenceBetween` generuje wiele kluczy naraz.
- **Rebalansowanie.** `rebalanceRanks(n)` wybiera najmniejsze L, że 62^L > n, i przypisuje kluczowi i wartość floor((i+1)·62^L/(n+1)) w arytmetyce całkowitej: 61 elementów → 1 znak, do 3843 → 2, 100 tys. → 3 znaki, ok. 8,5 ms dla 100 tys. Limit `RANK_MAX_LENGTH` = 32; po jego przekroczeniu cała lista jest przepisywana w jednej transakcji.
- **Kontrakt użycia.** Klucze liczy wyłącznie serwer z zapisanych sąsiadów (API przyjmuje identyfikatory sąsiadów lub indeks docelowy). Funkcja jest deterministyczna, więc równoległe wstawienia w tę samą lukę dają kolizję — potrzebne ograniczenie unikalności (kontener, `rank`) z ponowieniem lub blokada wierszy (M2b). `RankError` mapowany na HTTP 400/409.
- **Uzasadnienie.** ADR-002, ADR-022–024; pomiary: rozdz. 5.3.

## 4.2.2 Tablice, członkowie i role (M2b)
- **Warstwy.** Router → kontroler → serwis (`board.service`) → repozytoria (`board`, `board-member`); tylko repozytoria używają Prisma. Zależności składane ręcznie w `createApp`.
- **Autoryzacja.** `requireBoardPermission('members:manage')` wczytuje członkostwo wywołującego i zapisuje je w `res.locals.boardMembership`; serwis ponownie wywołuje `assertBoardPermission`. Fragment macierzy: `'board:delete': OWNER_ONLY`, `'members:manage': MANAGERS` (`OWNER`, `ADMIN`), `'card:write': CONTRIBUTORS` (bez `VIEWER`).
- **Operacje na członkach.** Zmiana roli i usunięcie to jedno warunkowe zapytanie (`... WHERE boardId = ? AND userId = ? AND role <> 'OWNER'`); gdy zwróci 0 wierszy, serwis rozróżnia brak członka (404) i próbę ruszenia właściciela (403). Dzięki temu nie ma okna wyścigu między sprawdzeniem a zapisem dla roli `OWNER`.
- **Właściciel.** Częściowy indeks `UNIQUE ("boardId") WHERE "role" = 'OWNER'` zapobiega drugiemu właścicielowi; tablica powstaje razem z członkostwem `OWNER`. Dodanie istniejącego członka (także równoległe) wykrywane jest przez błąd unikalności i mapowane na 409 `ALREADY_MEMBER`.
- **Kontrakt HTTP.** Endpointy i kody błędów: `docs/API.md`. Role przypisywalne przez API: `ADMIN`, `MEMBER`, `VIEWER` (`OWNER` odrzucany walidacją Zod).
- **Znane ograniczenia.** Wyścig między sprawdzeniem roli w middleware a mutacją przy równoległej degradacji wywołującego; brak endpointu przekazania własności.

## 4.2.3 / 4.3.3 Kolumny tablicy i przypisywanie kluczy porządkujących (M2c)
- **Schemat.** Model `Column` z enumem `ColumnCategory`; migracja `20261111151836_columns` zawiera ręcznie dopisany SQL, bo Prisma nie wyraża collation ani odraczalności ograniczeń:
```sql
"rank" TEXT COLLATE "C" NOT NULL,
ALTER TABLE "Column" ADD CONSTRAINT "Column_boardId_rank_key"
  UNIQUE ("boardId", "rank") DEFERRABLE INITIALLY IMMEDIATE;
```
  Collation `"C"` sprawia, że `ORDER BY "rank"` daje kolejność identyczną z porównaniem `<` w JavaScript (dla kluczy base-62 z mieszaną wielkością liter inaczej niż domyślne collation). Ograniczenie `DEFERRABLE INITIALLY IMMEDIATE` jest sprawdzane po każdej instrukcji, więc zwykły zapis dostaje natychmiastowy błąd P2002 (do ponowienia), a rebalansowanie można wykonać jedną instrukcją, w której klucze przejściowo pokrywają się z kluczami innych wierszy (ADR-028).
- **Warstwy.** `column.routes` (montowane w routerze tablic z `mergeParams`) → `column.controller` → `column.service` → `column.repository`; tylko repozytorium korzysta z Prisma. Repozytorium udostępnia `transaction(fn)` z interfejsem `ColumnStore` (`lockBoard`, `listForBoard`, `setRanks` itd.), a serwis składa z nich przypadki użycia; błąd P2002 jest tłumaczony na `RankCollisionError`.
- **Wstawianie i przeniesienie.** `resolveInsertIndex` zamienia `afterId`/`beforeId` na indeks w liście kolumn (z pominięciem przenoszonej), a `rankBetween` liczy klucz z sąsiadów. Każda próba to osobna transakcja READ COMMITTED zaczynająca się od blokady wiersza tablicy; po kolizji klucza próba jest powtarzana (maksymalnie `MAX_RANK_ATTEMPTS` = 5, potem 409 `RANK_CONFLICT`) ze złagodzonym sprawdzaniem przyległości sąsiadów, co pozwala obu równoległym przeniesieniom do jednej luki zakończyć się sukcesem (ADR-031).
- **Rebalansowanie.** Gdy obliczony klucz przekracza `RANK_MAX_LENGTH`, próba z blokadą współdzieloną jest przerywana i powtarzana z blokadą wyłączną; wtedy `rebalanceRanks(n)` przepisuje wszystkie klucze tablicy jedną instrukcją:
```sql
UPDATE "Column" AS c SET "rank" = v."rank"
FROM unnest($1::text[], $2::text[]) AS v("id", "rank")
WHERE c."id" = v."id" AND c."boardId" = $3
```
  Rebalansowanie nie zmienia `updatedAt` (kolejność się nie zmienia). Odpowiedź REST zawiera tylko jedną kolumnę, więc klient po rebalansowaniu odświeża listę.
- **Inwariant `DONE`.** Usunięcie i zmiana kategorii biorą blokadę wyłączną, zliczają kolumny `DONE` i przy ostatniej zwracają 409 `LAST_DONE_COLUMN`; równoległe usunięcia dwóch ostatnich kolumn `DONE` są serializowane.
- **Mapowanie błędów.** `error-handler` tłumaczy `RankError`: `INVALID_RANK` i `INVALID_ARGUMENT` → 400, `INVALID_RANK_ORDER` → 409.
- **Domyślne kolumny.** `POST /boards` tworzy tablicę, członkostwo właściciela i cztery kolumny w jednym zagnieżdżonym `create` (jedna transakcja); klucze z `rankSequenceBetween(null, null, 4)`.
- Szczegóły i kody błędów: `docs/API.md`; decyzje: ADR-028–032.

## 4.x Frontend tablic i kolumn (M2d)
- **Struktura.** `apps/web/src/features/boards/` (strony, okna dialogowe, klienty HTTP) i `apps/web/src/stores/board.ts`. Logika niezależna od Reacta jest w czystych funkcjach: `columnOrder.ts` (`reorderList`, `getMoveNeighbors`, `restoreOriginalPosition`, `rankFitsAt`), `columnDrag.ts` (przekształcenie `DropResult` w przeniesienie), `columnSettings.ts` (wyznaczenie zmienionych pól).
- **Porównywanie kluczy.** `compareRanks` używa operatorów `<`/`>` na łańcuchach, a nie `localeCompare`, aby kolejność zgadzała się z `COLLATE "C"` w PostgreSQL (por. 5.3, dowód różnicy względem `localeCompare`):

  ```ts
  // columnOrder.ts (idea)
  const fits = prev.rank < moved.rank && moved.rank < next.rank;
  if (!fits) await reloadColumns(); // rebalansowanie zmieniło inne klucze
  ```
- **Optymistyczny interfejs z wycofaniem.** `moveColumn` zmienia kolejność lokalnie, wysyła `PATCH .../move` z `afterId` i `beforeId` (`null` na krańcach), a przy błędzie przywraca wyłącznie przeniesioną kolumnę w aktualnej liście (`restoreOriginalPosition`), zachowując współbieżne zmiany (np. nazwy). Błędy `INVALID_POSITION`, `NEIGHBOR_NOT_FOUND` i 404 wywołują pełne odświeżenie. Podobnie zmiana nazwy kolumny jest zapisywana optymistycznie z wycofaniem.
- **Przeciąganie.** `@hello-pangea/dnd`, jeden poziomy `Droppable`; przeciąganie jest wyłączone bez `column:write` oraz w trakcie zapisu poprzedniego przeniesienia (`isMovingColumn`).
- **Ustawienia kolumny.** Formularz wysyła tylko zmienione pola; pusty limit WIP oznacza `null`.
- **Komponenty UI.** Dodano `dialog`, `alert-dialog`, `badge`, `checkbox`, `native-select` na bazie `radix-ui` (bez nowych zależności).
- **Zmiana strony startowej.** Usunięto stronę stanu API; `/` przekierowuje do `/boards`.
- Rozmiar paczki po M2d: 711 kB (wcześniej 537 kB); podział kodu (`lazy()`) odłożony.
- Decyzje: ADR-034–037.

## 4.x Przebudowa interfejsu (M2e)
- Zmiana dotyczyła wyłącznie warstwy prezentacji `apps/web`; logika, API i baza nie zostały zmienione. Dodano zależności `lucide-react` 1.72.0, `sonner` 2.0.8 i `@fontsource-variable/inter` 5.3.0.
- Motyw: `src/lib/theme.ts`, `src/stores/theme.ts` (Zustand), klucz `streamline-theme`; tryb systemowy nasłuchuje `prefers-color-scheme`. Skrypt w `index.html` powtarza regułę przed startem Reacta, a test weryfikuje ich zgodność.
- Awatary: kolor wybierany deterministycznie funkcją skrótu FNV-1a z identyfikatora użytkownika spośród 8 barw (`src/lib/avatar.ts`), stos z nakładaniem −8 px i limitem 4 awatarów plus „+N”.
- Responsywność: hook `useMediaQuery` (progi 1024 i 768 px); `BoardSidebar` zwija się (stan w `streamline-sidebar-collapsed`) lub jest szufladą opartą na Radix Dialog.
- `BoardColumn`: licznik `(n)`/`(n / limit)`, wizualizacja przekroczenia limitu WIP, kropka kategorii, stan pusty, osobny uchwyt `GripVertical`.
- Toasty (sonner) ze stałym `id`; `notice` czyszczone akcją `dismissNotice`.
- Znane ograniczenie: fragment po budowaniu 827 kB (711 kB po M2d) — planowane `lazy()` dla tras.
