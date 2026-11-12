# Rozdział 5 — Testy (notatki)

## 5.x Stan po M1a
Vitest: `packages/shared` — 28 testów, `apps/api` — 77, `apps/web` — 5 (łącznie 110). Jakość kodu: ESLint z regułami typowanymi, Prettier, hook pre-commit; CI w GitHub Actions.

## 5.x Testy integracyjne API
- Supertest na prawdziwym PostgreSQL, osobna baza `streamline_test` (ADR-015): `globalSetup` stosuje migracje (`prisma migrate deploy`), przed każdym testem tabele są czyszczone (`TRUNCATE`), pliki testowe uruchamiane sekwencyjnie. Zabezpieczenie: odmowa pracy na bazie, której nazwa nie kończy się na `_test`.
- Testy jednostkowe serwisów (`auth.service`, `token.service`), middleware (`require-auth`), konfiguracji i logera (redakcja danych wrażliwych), schematów Zod w `shared`.
- Scenariusze bezpieczeństwa objęte testami: rotacja refresh tokena, ponowne użycie w oknie łaski i po nim, równoległa rotacja, wylogowanie (idempotentne), jednakowa odpowiedź logowania dla nieznanego e-maila i błędnego hasła, limity żądań (429), atrybuty ciasteczka, odrzucanie tokenów z niedozwolonym algorytmem.

## 5.x Stan po M1b i testy frontendu
- Vitest: `packages/shared` — 28 testów, `apps/api` — 77, `apps/web` — 77 (łącznie 182).
- Testy `apps/web`: klient HTTP `api.test` (27: single-flight, ponowienie po `REFRESH_TOKEN_ROTATED`, błędy sieci/5xx/429, odrzucanie nieaktualnych wyników), `session` (6), `authApi` (2), `LoginPage` (7), `RegisterPage` (5), `ProtectedRoute` (9), `AppLayout` (3), `redirect` (10), `stores/auth` (3) oraz istniejące testy strony startowej.
- Podejście: React Testing Library z ręcznym adapterem axios zamiast MSW (ADR-019); czas sterowany przez wstrzykiwany `sleep`.
- Przegląd kodu (2 rundy) wykrył wyścig wylogowania z odświeżeniem w toku; poprawka (`waitForPendingRefresh`, `sessionEpoch`) ma pokrycie testami.

## 5.3 Testy jednostkowe LexoRank (M2a)
- Vitest: `packages/shared` — 226 testów (198 nowych: `keys`, `lexorank`, `rebalance`, `collation`). Po M2a: shared 226, api 77, web 77 (testy API nie były uruchamiane w tej sesji).
- Poprawność: porównanie `rankBetween` z wyrocznią na BigInt (najkrótszy klucz w przedziale), testy losowe ze stałym ziarnem (mulberry32, `test-utils/prng.ts`), inwariant porządku po seriach wstawień i rebalansowań.
- `collation.test.ts` dowodzi, że `localeCompare`/ICU daje inną kolejność niż `<` (bajtowa) dla mieszanej wielkości liter; stąd wymóg `COLLATE "C"` w PostgreSQL.

## 5.3 Benchmark: LexoRank a reindeksacja całkowitoliczbowa
**Metodyka.** Skrypt `packages/shared/bench/rank.bench.ts` (`pnpm --filter @streamline/shared bench`), Node v24.14.0, 1000 operacji na komórkę, ziarno 20261111. Porównywane: (a) gęste pozycje całkowitoliczbowe — przeniesienie elementu wymaga przepisania wszystkich elementów między pozycją źródłową a docelową; (b) klucze LexoRank — zapis jednego wiersza plus uśredniony koszt rebalansowania. Metryka główna: **wiersze zapisywane na operację** (`rows/op`) — dokładna i deterministyczna, bo to liczba zapisów do bazy (dla kluczy uwzględnia rebalansowanie rozłożone na operacje). Kolumny µs/op to wyłącznie czas obliczenia wartości porządkujących w pamięci (pomiar wsadowy `process.hrtime.bigint`, mediana z 5 powtórzeń), bez kosztu bazy danych — nie jest to czas odpowiedzi API.

**Przeniesienie losowej karty na początek** (int = pozycje całkowitoliczbowe, rank = LexoRank):

| n | int wiersze/op | rank wiersze/op | int µs | rank µs | rank maks. dł. | śr. dł. | rebalansowania |
|---|---|---|---|---|---|---|---|
| 100 | 51,5 | 1,00 | 0,094 | 0,216 | 18 | 16,51 | 0 |
| 1 000 | 481,5 | 1,00 | 0,439 | 0,104 | 19 | 8,30 | 0 |
| 10 000 | 4 996,0 | 1,00 | 3,539 | 0,127 | 20 | 3,79 | 0 |
| 100 000 | 49 257,1 | 1,00 | 35,144 | 0,104 | 20 | 3,07 | 0 |

**Ta sama pozycja środkowa** (powtarzany nacisk na jedną lukę):

| n | int wiersze/op | rank wiersze/op | int µs | rank µs | rank maks. dł. | śr. dł. | rebalansowania |
|---|---|---|---|---|---|---|---|
| 100 | 26,4 | 1,50 | 0,020 | 0,311 | 14 | 5,86 | 5 |
| 1 000 | 243,7 | 6,00 | 0,161 | 0,572 | 18 | 2,79 | 5 |
| 10 000 | 2 488,5 | 51,00 | 1,752 | 2,200 | 22 | 3,09 | 5 |
| 100 000 | 24 705,2 | 501,00 | 17,590 | 24,023 | 25 | 3,00 | 5 |

**Przeniesienie na koniec:**

| n | int wiersze/op | rank wiersze/op | int µs | rank µs | rank maks. dł. | śr. dł. | rebalansowania |
|---|---|---|---|---|---|---|---|
| 100 | 50,8 | 1,00 | 0,046 | 0,134 | 18 | 16,61 | 0 |
| 1 000 | 496,3 | 1,00 | 0,361 | 0,105 | 19 | 8,28 | 0 |
| 10 000 | 4 965,9 | 1,00 | 3,447 | 0,103 | 20 | 3,79 | 0 |
| 100 000 | 49 520,5 | 1,00 | 35,397 | 0,105 | 20 | 3,07 | 0 |

**Losowa pozycja:**

| n | int wiersze/op | rank wiersze/op | int µs | rank µs | rank maks. dł. | śr. dł. | rebalansowania |
|---|---|---|---|---|---|---|---|
| 100 | 34,8 | 1,00 | 0,025 | 0,063 | 4 | 2,51 | 0 |
| 1 000 | 339,1 | 1,00 | 0,231 | 0,059 | 3 | 2,12 | 0 |
| 10 000 | 3 163,5 | 1,00 | 2,227 | 0,076 | 4 | 2,95 | 0 |
| 100 000 | 33 524,8 | 1,00 | 24,199 | 0,101 | 4 | 2,98 | 0 |

**Wstawienie 1000 nowych kart w jedno miejsce** (najgorszy przypadek dla kluczy):

| n | int wiersze/op | rank wiersze/op | int µs | rank µs | rank maks. dł. | śr. dł. | rebalansowania |
|---|---|---|---|---|---|---|---|
| 100 | 550,5 | 4,84 | 0,374 | 0,309 | 17 | 2,53 | 6 |
| 1 000 | 1 000,5 | 10,20 | 0,693 | 0,509 | 20 | 2,38 | 6 |
| 10 000 | 5 500,5 | 64,15 | 3,874 | 2,552 | 23 | 3,07 | 6 |
| 100 000 | 50 500,5 | 604,09 | 36,247 | 27,348 | 27 | 3,00 | 6 |

**Liczba wstawień do konieczności rebalansowania** (w nawiasie początkowa długość klucza):

| n (dł. początkowa) | stały sąsiad „po” | stały sąsiad „przed” | naprzemiennie w luce | zawsze na końcu | zawsze na początku |
|---|---|---|---|---|---|
| 100 (2) | 156 | 187 | 186 | 1 869 | 1 868 |
| 1 000 (2) | 153 | 182 | 182 | 1 834 | 1 833 |
| 10 000 (3) | 150 | 180 | 179 | 1 793 | 1 792 |
| 100 000 (3) | 148 | 176 | 176 | 1 772 | 1 771 |

**Interpretacja.**
- Dla przeniesień na początek, na koniec i w losowe miejsce LexoRank zapisuje dokładnie 1 wiersz niezależnie od n, a reindeksacja — średnio ok. n/3 do ok. n/2 wierszy (ok. 49 tys. dla n = 100 000); jej koszt rośnie liniowo. To główny wynik: O(1) zapisów wobec O(n).
- Najgorszy przypadek dla kluczy to wielokrotne wstawianie w tę samą lukę: po ok. 150–190 wstawieniach klucz osiąga limit i lista jest rebalansowana (zapis n wierszy). Zamortyzowany koszt (501 wierszy/op dla n = 100 000 przy powtarzaniu tej samej pozycji) jest mimo to ok. 50 razy mniejszy niż reindeksacji (24 705). Przy dopisywaniu na końcu lub początku rebalansowanie jest potrzebne dopiero po ok. 1770–1870 wstawieniach.
- Czas obliczeń w pamięci dla dużych n jest dla kluczy stały (ok. 0,1 µs), a dla reindeksacji rośnie liniowo (35 µs dla n = 100 000); dla małych n i w scenariuszach z rebalansowaniem klucze bywają wolniejsze (0,216 wobec 0,094 µs dla n = 100). Różnice są pomijalne wobec kosztu zapytań do bazy, dlatego argumentację opieramy na liczbie wierszy.
- Długości kluczy pozostają niewielkie: maksimum 27 znaków (poniżej `RANK_MAX_LENGTH` = 32), średnio ok. 3 znaki po rebalansowaniu.
- Jedyną linią bazową jest gęsta reindeksacja całkowitoliczbowa. Pozycje zmiennoprzecinkowe odrzucono, bo tracą precyzję po ok. 50 kolejnych połowieniach przedziału; pozycje całkowitoliczbowe z odstępami nadal wymagają okresowej renumeracji, czyli w praktyce tego samego rebalansowania, tylko z mniejszym zapasem.
- Ograniczenia: pomiar nie obejmuje bazy danych, sieci ani współbieżności; wyniki z jednej maszyny i jednego ziarna.

## 5.x Testy tablic, członków i ról (M2b)
- Vitest: `packages/shared` — 316 testów (z 226), `apps/api` — 193 (z 77), `apps/web` — 77; `lint`, `typecheck`, `format:check` bez błędów.
- `shared`: `permissions.test.ts` (macierz uprawnień, `canManageMember`, `canLeaveBoard`), `schemas/board.test.ts` (schematy tablic i członków).
- `apps/api`: `board.service.test.ts` (reguły serwisu), `require-board-permission.test.ts` (middleware), integracyjne `board.routes.test.ts` na prawdziwym PostgreSQL (Supertest) obejmujące kody 400/401/403/404/409, m.in. identyczność 404 dla nie-członka i nieistniejącej tablicy, kolejność sprawdzeń (rola przed walidacją), 409 `OWNER_CANNOT_LEAVE`, 403 przy zmianie i usunięciu `OWNER`, `ALREADY_MEMBER`.
- Przegląd kodu: zatwierdzony w 1 rundzie; uwagi drobne (komentarz w schemacie, nieużywane `canManageMember`, brak walidacji `:userId`) odnotowane w stanie projektu.

## 5.x Testy kolumn i współbieżności (M2c)
- Vitest: `packages/shared` — 368 testów (z 316, +52), `apps/api` — 342 (z 193, +149), `apps/web` — 77; `lint`, `typecheck`, `test`, `format:check` bez błędów (potwierdzone w przeglądzie kodu, 2 rundy, zatwierdzone).
- `shared`: `schemas/column.test.ts` (schematy ścisłe, granice limitu WIP, reguły sąsiadów).
- `apps/api`: `column.service.test.ts` (reguły serwisu, `resolveInsertIndex`), `column.routes.test.ts` (integracyjne, prawdziwy PostgreSQL, Supertest), `column.repository.test.ts` (integracyjne: limity czasu transakcji, odpowiedź 503), `prisma-errors.test.ts` (jednostkowe: rozpoznawanie ograniczenia unikalności i kodów PostgreSQL), uzupełnione `board.service.test.ts` (kolumny domyślne) i `error-handler.test.ts` (mapowanie `RankError`).
- Testy współbieżności i spójności na prawdziwej bazie:
  - kolejność `ORDER BY rank` w PostgreSQL zgodna z porządkiem w JavaScript (mieszana wielkość liter; uzasadnia `COLLATE "C"`),
  - równoległe przeniesienia w tę samą lukę — oba kończą się sukcesem; synchronizacja żądań przez `pg_stat_activity` (oczekiwanie, aż transakcje faktycznie czekają na blokadę),
  - deterministycznie wstrzyknięta kolizja klucza i ponowienie,
  - rebalansowanie przy wstawianiu i przy przeniesieniu (kolejność zachowana),
  - wyścigi o inwariant „co najmniej jedna kolumna `DONE`” (równoległe usunięcia i zmiany kategorii).
- Ograniczenia: brak testu obciążeniowego; ponowienia ograniczone do 5 prób, więc przy bardzo dużej rywalizacji możliwe jest 409 `RANK_CONFLICT` (nie mierzono częstości). Limity czasu transakcji są jawne (`lock_timeout` 2 s, `maxWait` 2 s, `timeout` 5 s); przekroczenie daje 503 `BOARD_BUSY` bez ponowień (ADR-033). Uwagi z drugiej rundy przeglądu odnotowane w stanie projektu.

## 5.x Testy frontendu tablic i kolumn (M2d)
- Liczba testów: web 77 → 172 (shared 368, api 342, bez zmian).
- Testy jednostkowe czystej logiki: `columnOrder` (porównywanie kluczy bez `localeCompare`, wyznaczanie sąsiadów, częściowe wycofanie, sprawdzanie dopasowania `rank`), `columnDrag`, `columnSettings`, `boardErrorMessage`.
- Testy magazynu `stores/board.test.ts`: odrzucanie spóźnionych odpowiedzi (liczniki generacji, także `renameBoard`), zerowanie przy zmianie sesji, optymistyczne przeniesienie z wycofaniem zachowującym współbieżne zmiany, jedno przeniesienie naraz, odświeżenie po `INVALID_POSITION`/`NEIGHBOR_NOT_FOUND`/404 i po wykryciu rebalansowania.
- Testy komponentów (RTL, ręczny adapter axios): `BoardsPage` (stany ładowania, pusty, błąd z ponowieniem, tworzenie, zmiana nazwy, usuwanie) i `BoardPage`, w tym rzeczywiste przeciąganie kolumny klawiaturą przez `@hello-pangea/dnd`, okno ustawień, okno członków, stan 404 i komunikat `BOARD_BUSY`. W `test/setup.ts` dodano stub `ResizeObserver` (jsdom). Natywny `<select>` ułatwił testy (ADR-036).
- Ograniczenia: brak testów end-to-end (Playwright) i ręcznej weryfikacji w przeglądarce (układ, przewijanie poziome, przeciąganie myszą); nie mierzono wydajności interfejsu.

## 5.x Testy przebudowy interfejsu (M2e)
- Liczba testów frontendu wzrosła z 172 do 219 (shared 368, api 342 bez zmian). Nowe przypadki: motyw (`theme.test.ts`, w tym zgodność skryptu w `index.html` z modułem), awatary (`avatar.test.ts`), `BoardColumn` (liczniki, przekroczenie WIP, stan pusty), `BoardSidebar` (zwijanie, szuflada), menu `AppHeader`, powrót fokusu do przycisku ⋯, pojedynczy toast w StrictMode, stany puste i szkielety ładowania.
- Kontrole `lint`, `typecheck`, `test`, `build` i `format:check` przechodzą.
- Luka: nie wykonano ręcznej weryfikacji w przeglądarce (układ, motyw ciemny, brak migotania, przeciąganie myszą, toasty); testy jsdom nie obejmują renderowania CSS.
