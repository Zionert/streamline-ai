# Decyzje architektoniczne (ADR)

## ADR-001: Monorepo pnpm ze wspólnym pakietem
- **Kontekst:** frontend i backend w TypeScript, wspólne DTO i reguły walidacji.
- **Decyzja:** pnpm workspaces: `apps/web`, `apps/api`, `packages/shared` (schematy Zod, typy, LexoRank, zdarzenia WebSocket).
- **Alternatywy:** dwa osobne repozytoria (duplikacja typów); Turborepo/Nx (nadmierna złożoność dla 2 osób).
- **Konsekwencje:** jeden PR może zmienić kontrakt i obie strony jednocześnie; jedno CI.

## ADR-002: Kolejność elementów przez LexoRank (implementacja własna)
- **Kontekst:** przeniesienie karty przy pozycjach całkowitoliczbowych wymaga aktualizacji O(n) wierszy.
- **Decyzja:** pole `rank` (string); nowy klucz wyznaczany pomiędzy sąsiadami — O(1) zapisów; okresowe rebalansowanie.
- **Alternatywy:** pozycja całkowitoliczbowa; pozycja zmiennoprzecinkowa (utrata precyzji); pakiet npm `fractional-indexing`.
- **Konsekwencje:** kluczowy algorytm pracy (rozdz. 4.1.2, 4.3.2, 5.3); wymaga starannych testów.

## ADR-003: REST dla zmian, Socket.io wyłącznie do rozgłaszania
- **Kontekst:** potrzebna jest zarówno trwałość danych, jak i synchronizacja w czasie rzeczywistym.
- **Decyzja:** wszystkie zmiany przechodzą przez REST (walidacja, uprawnienia, transakcja); po sukcesie serwer emituje zdarzenie do pokoju `board:<id>`.
- **Alternatywy:** zmiany bezpośrednio przez WebSocket (trudniejsza walidacja i obsługa błędów).
- **Konsekwencje:** jedno źródło prawdy, łatwiejsze testowanie; klient inicjujący stosuje aktualizację optymistyczną.

## ADR-004: Access token w pamięci, refresh token w ciasteczku httpOnly
- **Decyzja:** access 15 min (stan Zustand), refresh 7 dni (ciasteczko httpOnly, skrót w bazie, rotacja).
- **Konsekwencje:** ochrona przed kradzieżą tokena przez XSS; wymagany endpoint `/auth/refresh` i interceptor axios.

## ADR-005: Pakiet `shared` udostępniany przez niestandardowy warunek `exports`
- **Kontekst:** `apps/web` i `apps/api` importują `@streamline/shared`; kompilowanie pakietu przed każdym typecheckiem, testem czy startem serwera deweloperskiego spowalnia pracę.
- **Decyzja:** w `packages/shared/package.json` pole `exports` zawiera warunek `@streamline/source` wskazujący na `src/index.ts`. Typecheck, ESLint, Vite, `tsx` (dev) i testy używają źródeł TypeScript; skompilowany `dist/` jest używany tylko przy produkcyjnym budowaniu API (`pnpm -r build` buduje `shared` jako pierwszy).
- **Alternatywy:** zawsze kompilowany `dist/` (konieczny watch/build); aliasy ścieżek w `tsconfig` (duplikacja konfiguracji w każdym narzędziu).
- **Konsekwencje:** brak kroku budowania w pętli deweloperskiej; każde narzędzie musi być skonfigurowane z tym warunkiem.

## ADR-006: Jedna konfiguracja ESLint w katalogu głównym
- **Kontekst:** trzy pakiety o zbliżonych regułach.
- **Decyzja:** jeden plik `eslint.config.js` w korzeniu, z regułami typowanymi (type-checked) z `typescript-eslint`; Prettier uruchamiany jednym przebiegiem z korzenia (`format`, `format:check`; pomija `docs/` i `*.md`). Husky + lint-staged uruchamia lint i formatowanie przy commicie.
- **Alternatywy:** osobna konfiguracja w każdym pakiecie.
- **Konsekwencje:** spójne reguły; pliki w korzeniu (np. `eslint.config.js`) nie są objęte `pnpm -r lint`.

## ADR-007: TypeScript 6.0.3 zamiast 7.x
- **Kontekst:** istnieje TypeScript 7.0.2, ale `typescript-eslint` 8.72.2 wymaga wersji `<6.1`.
- **Decyzja:** przypięcie TypeScript 6.0.3.
- **Alternatywy:** TypeScript 7 bez lintowania typowanego; rezygnacja z reguł type-checked.
- **Konsekwencje:** migracja do TS 7, gdy `typescript-eslint` ją wesprze.

## ADR-008: Prisma 7 z adapterem `@prisma/adapter-pg`, wygenerowany klient poza repozytorium
- **Kontekst:** Prisma 7.10.1 to najnowsza wersja stabilna (8.0 dostępna tylko jako RC); w wersji 7 adres bazy znajduje się w pliku konfiguracyjnym, a połączenie wymaga adaptera sterownika.
- **Decyzja:** `datasource` konfigurowany w `apps/api/prisma.config.ts` (ładuje `.env` przez `process.loadEnvFile`); generator `prisma-client` zapisuje klienta w `apps/api/src/generated/prisma` (w `.gitignore`, generowany w `postinstall` pakietu API); klient tworzony w `repositories/db.ts` (`createDbClient`) z adapterem `@prisma/adapter-pg`, połączenie leniwe.
- **Alternatywy:** Prisma 6 (starszy model konfiguracji); commitowanie wygenerowanego klienta.
- **Konsekwencje:** po `pnpm install` klient jest generowany automatycznie; `/health` nie dotyka bazy.

## ADR-009: Potwierdzenie `@hello-pangea/dnd` z React 19
- **Kontekst:** w M0 należało zweryfikować zgodność biblioteki z najnowszym React.
- **Decyzja:** zostaje `@hello-pangea/dnd` 18.0.1 (peer deps `react ^18 || ^19`); test dymny `DragDropContext`/`Droppable`/`Draggable` przechodzi na React 19.3.1 bez błędów w konsoli.
- **Alternatywy:** dnd-kit.
- **Konsekwencje:** ostatnie wydanie biblioteki pochodzi z lutego 2025 — dnd-kit pozostaje rozwiązaniem awaryjnym.

## ADR-010: bcryptjs zamiast bcrypt do haszowania haseł
- **Kontekst:** pakiet `bcrypt` wymaga natywnej kompilacji/prebuildów, co sprawia problemy na Windows z Node 24.
- **Decyzja:** `bcryptjs` 3.0.3 (czysty JavaScript), współczynnik kosztu `BCRYPT_COST` (4–15, domyślnie 12, w produkcji min. 12). Hasła są ograniczone do 72 bajtów UTF-8 w schemacie Zod, bo bcrypt obcina dłuższe dane. Przy logowaniu nieznanego e-maila wykonywane jest porównanie z atrapą hasha (tworzoną przy starcie) — wyrównanie czasu odpowiedzi.
- **Alternatywy:** `bcrypt` (natywny, szybszy); argon2 (nowocześniejszy, również natywny).
- **Konsekwencje:** brak problemów z instalacją; wolniejsze haszowanie niż w wersji natywnej. W testach używany jest niski koszt.

## ADR-011: jose zamiast jsonwebtoken do tokenów dostępu
- **Kontekst:** potrzebna weryfikacja JWT z jawnym ograniczeniem dozwolonych algorytmów.
- **Decyzja:** `jose` 6.2.17, algorytm HS256 (`JWT_ACCESS_SECRET`, min. 32 znaki), weryfikacja z listą dozwolonych algorytmów (`algorithms: ['HS256']`) i wymaganymi claimami `sub`, `exp`. TTL 15 min (`ACCESS_TOKEN_TTL_SECONDS`).
- **Alternatywy:** `jsonwebtoken`; RS256 (zbędny przy jednym serwisie).
- **Konsekwencje:** brak claimów `iss`/`aud` — do dodania przed uwierzytelnianiem Socket.io.

## ADR-012: Atrybuty ciasteczka refresh token
- **Kontekst:** refresh token jest wrażliwszy niż access token (ADR-004); przeglądarka komunikuje się z API przez to samo źródło dzięki proxy `/api`.
- **Decyzja:** ciasteczko `refresh_token`: `httpOnly`, `SameSite=Strict`, `Secure` w produkcji, `Path=/api/auth` (konfigurowalne `REFRESH_COOKIE_PATH`), `Max-Age` 7 dni. Token jest wysyłany tylko do endpointów `/api/auth/*`.
- **Alternatywy:** `SameSite=Lax`; ciasteczko dla ścieżki `/`; refresh token w ciele odpowiedzi.
- **Konsekwencje:** ochrona przed XSS i CSRF; wdrożenie z frontendem i API w różnych witrynach (cross-site) wymagałoby zmiany `SameSite`; `REFRESH_COOKIE_PATH` musi odpowiadać ścieżce widzianej przez przeglądarkę.

## ADR-013: Rotacja refresh tokenów i wykrywanie ponownego użycia z oknem łaski
- **Kontekst:** skradziony refresh token pozwala na długotrwały dostęp; jednocześnie kilka kart przeglądarki może odświeżać sesję równolegle tym samym tokenem.
- **Decyzja:** każde `/auth/refresh` unieważnia stary token i wydaje nowy (transakcja; `updateMany ... WHERE revokedAt IS NULL` rozstrzyga wyścigi). Ponowne użycie unieważnionego tokena:
  - w oknie `REFRESH_REUSE_GRACE_SECONDS` (domyślnie 30 s, 0–300, 0 = tryb ścisły) po rotacji lub przy przegranej równoległej rotacji — 401 `REFRESH_TOKEN_ROTATED`, bez unieważniania pozostałych tokenów, ciasteczko zachowane;
  - po oknie albo dla tokena unieważnionego wylogowaniem/masowo — unieważnienie wszystkich refresh tokenów użytkownika, 401 `UNAUTHORIZED`, usunięcie ciasteczka, wpis w logu.
- **Alternatywy:** brak rotacji; ścisła rotacja bez okna (fałszywe alarmy przy wielu kartach).
- **Konsekwencje:** kompromis — jeśli atakujący odświeży token jako pierwszy, a prawowity klient użyje starego tokena w oknie łaski, wykrycie kradzieży jest opóźnione. Okno liczone zegarem aplikacji (poprawne dla jednej instancji). Klient (M1b) musi ponowić odświeżenie po `REFRESH_TOKEN_ROTATED`.

## ADR-014: Walidacja schematami `shared` bezpośrednio w kontrolerach, ręczne DI
- **Kontekst:** `docs/TECH_STACK.md` zakładał middleware `validate(schema)`.
- **Decyzja:** kontrolery wywołują `schema.parse(req.body)`; `ZodError` jest mapowany na 400 `VALIDATION_ERROR` w `error-handler`. Schematy nadal pochodzą wyłącznie z `packages/shared`. Zależności (konfiguracja, logger, klient bazy) są wstrzykiwane ręcznie przez `createApp({ config, logger, db })`.
- **Alternatywy:** middleware `validate()`; kontener DI.
- **Konsekwencje:** typ wejścia wynika z `parse` bez rzutowania; mniej warstw. Middleware można dodać później, jeśli powtórzenia się nasilą.

## ADR-015: Osobna baza testowa dla testów integracyjnych API
- **Kontekst:** testy Supertest muszą działać na prawdziwym PostgreSQL bez ryzyka skasowania danych deweloperskich.
- **Decyzja:** `globalSetup` Vitest wyznacza adres (`DATABASE_URL_TEST` lub `DATABASE_URL` z `_test` dodanym do nazwy bazy), odmawia pracy, gdy nazwa bazy nie kończy się na `_test`, wykonuje `prisma migrate deploy` (tworzy bazę, jeśli brak), a przed każdym testem `TRUNCATE`. `fileParallelism: false` (wspólna baza). CI ustawia `DATABASE_URL_TEST`.
- **Alternatywy:** mocki repozytoriów (nie sprawdzają transakcji ani ograniczeń); baza w pamięci (inny silnik).
- **Konsekwencje:** testy wymagają działającego PostgreSQL (`pnpm db:up`); są wolniejsze od testów z mockami, ale wykrywają błędy zapytań i wyścigów.

## ADR-016: Access token w pamięci i odświeżanie single-flight z jednym opóźnionym ponowieniem
- **Kontekst:** ADR-004 wymaga trzymania access tokena poza `localStorage`; równoległe żądania 401 nie mogą wywołać wielu rotacji refresh tokena, a serwer zwraca `REFRESH_TOKEN_ROTATED` przy wyścigu (ADR-013).
- **Decyzja:** token tylko w stanie Zustand. Interceptor axios po 401 wykonuje jedno wspólne `POST /auth/refresh` (współdzielona obietnica), po czym powtarza żądanie jeden raz (`authRetried`); jeśli token zmienił się w międzyczasie, żądanie jest powtarzane bez nowego odświeżenia. Przy `REFRESH_TOKEN_ROTATED` klient czeka 400 ms i ponawia odświeżenie jeden raz. Endpointy `/auth/refresh|login|register|logout` oraz żądania z `skipAuthRefresh` nie wyzwalają odświeżania (`/auth/me` tak). Sesja jest czyszczona tylko po 401 lub niepoprawnej odpowiedzi odświeżenia; błędy sieci, 5xx i 429 ją zachowują.
- **Alternatywy:** token w `localStorage`; odświeżanie cykliczne w tle; ponawianie bez limitu.
- **Konsekwencje:** brak tokena w pamięci trwałej (po przeładowaniu strony sesję przywraca `/auth/refresh`); stałe 400 ms jest heurystyką (`DEFAULT_ROTATED_RETRY_DELAY_MS`, wstrzykiwalne w testach).

## ADR-017: Epoka sesji i wylogowanie czekające na odświeżenie
- **Kontekst:** odświeżenie w toku mogłoby po wylogowaniu przywrócić sesję lub unieważnić nowszą (wynik z przeszłości).
- **Decyzja:** `sessionEpoch` rośnie przy `clearSession`; wynik odświeżenia z innej epoki jest odrzucany (`StaleRefreshError`), a nieaktualna porażka nie czyści nowszej sesji. `logout` najpierw czeka na `waitForPendingRefresh`, potem wysyła `POST /auth/logout`, zawsze czyści stan i nigdy nie odrzuca obietnicy.
- **Alternatywy:** anulowanie żądań (`AbortController`); brak ochrony.
- **Konsekwencje:** znane luki (`docs/PROJECT_STATE.md`): odświeżenie może wystartować w trakcie `POST /auth/logout`; `login`/`register` nie zwiększają epoki.

## ADR-018: Przywracanie sesji odporne na StrictMode
- **Kontekst:** w trybie deweloperskim React uruchamia efekty dwukrotnie; dwa `POST /auth/refresh` z tym samym ciasteczkiem dałyby `REFRESH_TOKEN_ROTATED`.
- **Decyzja:** `restoreSession` działa tylko dla `status === 'idle'` i synchronicznie przełącza stan na `loading`, więc drugie wywołanie jest pomijane. `/auth/refresh` zwraca użytkownika, więc `/auth/me` nie jest wywoływane. Wywołuje go `AuthBootstrap` (trasa główna).
- **Alternatywy:** flaga w `useRef`; wyłączenie StrictMode.
- **Konsekwencje:** stan sesji jest źródłem prawdy o tym, czy przywracanie już trwa.

## ADR-019: Testy HTTP z ręcznym adapterem axios zamiast MSW
- **Kontekst:** testy klienta wymagają deterministycznego sterowania odpowiedziami (kolejność, opóźnienia, równoległość 401).
- **Decyzja:** `apps/web/src/test/mockAdapter.ts` implementuje adapter axios i jest wstrzykiwany do `createApiClient({ adapter })`; MSW nie jest używany.
- **Alternatywy:** MSW (dodatkowa zależność i warstwa przechwytywania sieci).
- **Konsekwencje:** testy nie obejmują warstwy XHR/`fetch`; mniej zależności, pełna kontrola nad czasem (`sleep` wstrzykiwany).

## ADR-020: Analityka przepływu na podstawie rejestru przejść kart
- **Kontekst:** wyróżnikiem projektu jest analityka przepływu (lead/cycle time, przepustowość, CFD, aging WIP, prognoza Monte Carlo — `docs/REQUIREMENTS.md`, WF-FLOW). Metryki wymagają pełnej historii ruchu kart, której nie da się odtworzyć wstecz.
- **Decyzja:** od M3 każde przeniesienie karty między kolumnami zapisuje wiersz `CardTransition` w tej samej transakcji co zmianę `columnId`/`rank`. Kolumny mają kategorię (`BACKLOG`, `TODO`, `IN_PROGRESS`, `DONE`), dzięki której metryki nie zależą od nazw kolumn. Obliczenia metryk i symulacja Monte Carlo jako czyste funkcje w `packages/shared`; API dostarcza surowe dane lub zagregowane wyniki.
- **Alternatywy:** wyliczanie z ogólnego `ActivityLog` (słabsza struktura, trudniejsze zapytania); znaczniki czasu na karcie (tracą historię ponownych otwarć); zewnętrzne narzędzie BI (poza zakresem).
- **Konsekwencje:** dodatkowy zapis przy każdym przeniesieniu (akceptowalny koszt); indeksy na `(boardId, movedAt)` i `(cardId, movedAt)`; metryki testowalne jednostkowo na danych syntetycznych.

## ADR-021: Asystent AI jako opcjonalny moduł za adapterem dostawcy
- **Kontekst:** funkcje pomocnicze (podział zadania, kryteria akceptacji, podsumowanie tygodnia — WF-AI) korzystają z zewnętrznego modelu językowego, który jest płatny, niedeterministyczny i może być niedostępny.
- **Decyzja:** moduł po stronie serwera za interfejsem `LlmProvider` (konkretny dostawca wybierany konfiguracją), włączany zmienną `AI_ENABLED`; wyniki są wyłącznie propozycjami zatwierdzanymi przez użytkownika; limit zapytań na użytkownika (`AiUsage`); w testach dostawca atrapowy.
- **Alternatywy:** wywołania z przeglądarki (wyciek klucza API); automatyczne stosowanie wyników (ryzyko błędnych zmian); brak modułu.
- **Konsekwencje:** aplikacja działa w pełni bez AI; zmiana dostawcy nie wpływa na resztę kodu; moduł realizowany dopiero po ukończeniu M0–M6.

## ADR-022: Format klucza kolejności — base-62, bez końcowego `0`, porządek bajtowy
- **Kontekst:** ADR-002 wybiera pole `rank` (string); trzeba określić alfabet, postać kanoniczną i sposób porównywania dający ten sam wynik w JS i w PostgreSQL.
- **Decyzja:** alfabet `0-9A-Za-z` (62 znaki, rosnące kody ASCII). Klucz to ułamek `0.d1d2…`, niepusty, **nie może kończyć się na `0`** (jedna postać dla każdej wartości, więc porządek leksykograficzny = liczbowy; każdy klucz > 0, więc zawsze jest miejsce przed nim). Porównanie: `<` w JS = porządek bajtów UTF-8 = `COLLATE "C"` w PostgreSQL. `localeCompare`/ICU daje inny wynik dla mieszanej wielkości liter (potwierdza `collation.test.ts`), dlatego kolumna `rank` musi mieć `COLLATE "C"`. `rankFirst()` = `V` (0,5).
- **Alternatywy:** alfabet base-36/małe litery (dłuższe klucze, brak problemu z kollacją); dopuszczenie końcowego `0` (wiele zapisów tej samej wartości, nieprawdziwy porządek tekstowy); domyślna kollacja bazy (zależna od lokalizacji, niezgodna z JS).
- **Konsekwencje:** M2b musi ustawić `COLLATE "C"` na kolumnach `rank` i przetestować to na prawdziwym PostgreSQL; `rankSchema` odrzuca klucze niekanoniczne.

## ADR-023: Wybór klucza — najkrótszy w przedziale, narastanie o znak na ok. 61 dopisań
- **Kontekst:** długość kluczy decyduje o częstości rebalansowania; różne wzorce użycia mają inne najgorsze przypadki.
- **Decyzja:** `rankBetween(a, b)` przechodzi cyfra po cyfrze i zwraca najkrótszy klucz w przedziale (zweryfikowane wyrocznią BigInt). Przy dwóch granicach — środkowy kandydat; przy jednej otwartej — kandydat najbliższy sąsiadowi (`rankAfter` zwiększa pierwszą cyfrę z zapasem: `V`→`W`, `z`→`z1`), więc dopisywanie rośnie o jeden znak na ok. 61 wstawień. Błędy: `RankError` (`INVALID_RANK`, `INVALID_RANK_ORDER`, `INVALID_ARGUMENT`). Funkcja jest deterministyczna.
- **Alternatywy:** zawsze środek przedziału (szybki wzrost długości przy dopisywaniu); losowy klucz w przedziale (mniej kolizji, ale niedeterministyczny i trudniejszy do testowania).
- **Konsekwencje:** równoległe wstawienia w tę samą lukę dają ten sam klucz — M2b wymaga unikalności (kontener, `rank`) i ponowienia albo blokady wierszy. Pomiary: rozdz. 5.3.

## ADR-024: `RANK_MAX_LENGTH` = 32 i rebalansowanie całej listy
- **Kontekst:** klucze mogą rosnąć bez ograniczeń przy wielokrotnym wstawianiu w jedną lukę.
- **Decyzja:** `RANK_MAX_LENGTH` = 32, `rankSchema` odrzuca dłuższe. Gdy wyliczony klucz jest dłuższy, nie jest zapisywany; cała lista kontenera jest przepisywana przez `rebalanceRanks(n)` w tej samej transakcji. `rebalanceRanks(n)`: najmniejsze L, że 62^L > n; klucz i = floor((i+1)·62^L/(n+1)) w arytmetyce całkowitej (61 elementów → 1 znak, do 3843 → 2, 100 tys. → 3; ok. 8,5 ms dla 100 tys.). Klucze liczy wyłącznie serwer z zapisanych sąsiadów; API przyjmuje identyfikatory sąsiadów lub indeks docelowy.
- **Alternatywy:** rebalansowanie w tle (stany pośrednie, wyścigi); brak limitu długości (rosnące indeksy i koszt porównań).
- **Konsekwencje:** rebalansowanie to rzadki zapis O(n); M2b decyduje o zbiorczym zdarzeniu socket i mapuje `RankError` na HTTP 400/409.
- **Uwaga (drobna):** benchmark działa na `dist` przez natywne usuwanie typów Node 24 (`node bench/rank.bench.ts`, odwołanie do pakietu przez samoodniesienie), bez nowych zależności; dlatego generator PRNG jest zduplikowany w pliku benchmarku (testy używają `test-utils/prng.ts`).

## ADR-025: Dwuwarstwowa kontrola ról — czysta macierz w `shared`, middleware i ponowne sprawdzenie w serwisie
- **Kontekst:** role (`OWNER`/`ADMIN`/`MEMBER`/`VIEWER`) muszą być egzekwowane w API i odzwierciedlone w interfejsie; błąd w jednym miejscu nie może otwierać dostępu.
- **Decyzja:** macierz `BOARD_PERMISSIONS` (akcja → dozwolone role) i `hasBoardPermission` w `packages/shared` jako czysty, testowalny kod. W API: middleware `requireBoardPermission(action, { param })` ładuje członkostwo i zapisuje je w `res.locals.boardMembership`; `board.service` ponownie wywołuje `assertBoardPermission` w każdej metodzie przyjmującej `BoardMembership`. Rola jest sprawdzana przed walidacją ciała.
- **Alternatywy:** sprawdzanie ról wyłącznie w middleware (pomyłka w trasie = luka); wyłącznie w serwisach (brak wczesnego odrzucenia, zduplikowane ładowanie członkostwa); osobna macierz we frontendzie (rozjazd reguł).
- **Konsekwencje:** jedno źródło prawdy dla API i UI; nowe funkcje (kolumny, karty) dodają akcję do macierzy i używają tego samego middleware (trasy zagnieżdżone z `mergeParams`). Znany dług: wyścig między sprawdzeniem w middleware a mutacją przy równoległej degradacji roli; `canManageMember` w `shared` nie jest używane przez API (regułę `role <> OWNER` realizuje repozytorium) i trzeba je utrzymywać zgodnie.

## ADR-026: 404 dla nie-członków zamiast 403; właściciel nie może opuścić tablicy (409)
- **Kontekst:** odpowiedź 403 dla istniejącej, cudzej tablicy ujawniałaby jej istnienie; właściciel odchodzący zostawiłby tablicę bez właściciela.
- **Decyzja:** brak członkostwa i nieistniejąca tablica dają identyczne 404 `NOT_FOUND` („Board not found”); 403 `FORBIDDEN` tylko dla członka bez uprawnienia. `DELETE /boards/:boardId/members/:userId` z własnym id oznacza opuszczenie tablicy; dla właściciela kończy się 409 `OWNER_CANNOT_LEAVE` (najpierw trzeba przekazać własność — akcja `board:transfer`, endpoint w przyszłym etapie). Próba zmiany lub usunięcia `OWNER` przez innych: 403; `OWNER` w ciele żądania: 400. Nieznany e-mail przy dodawaniu: 404 `USER_NOT_FOUND` (ujawnia istnienie konta wyłącznie zalogowanym menedżerom tablicy); duplikat: 409 `ALREADY_MEMBER`.
- **Alternatywy:** 403 dla nie-członków (prostsze, ale ujawnia istnienie); automatyczne przekazanie własności przy odejściu (nieoczywiste dla użytkownika); usunięcie tablicy wraz z odejściem właściciela (utrata danych).
- **Konsekwencje:** nie da się odróżnić cudzej tablicy od nieistniejącej; odejście właściciela wymaga zaimplementowania `board:transfer`. Kody domenowe są w `BOARD_ERROR_CODES` (`shared`), a `NotFoundError`/`ConflictError` przyjmują opcjonalny kod.

## ADR-027: Jeden właściciel tablicy — częściowy indeks unikalny i zapytania warunkowe
- **Kontekst:** tablica powinna mieć zawsze jednego właściciela, także przy równoległych żądaniach; sprawdź-potem-zapisz w serwisie jest podatne na wyścigi.
- **Decyzja:** w migracji `20261111143833_boards_members` ręcznie dopisany indeks `BoardMember_one_owner_per_board` (`UNIQUE ("boardId") WHERE "role" = 'OWNER'`) gwarantuje najwyżej jednego właściciela. Co najmniej jednego zapewnia serwis: tablica powstaje wraz z właścicielem, a zmiana i usunięcie członka to warunkowe `UPDATE`/`DELETE ... WHERE role <> 'OWNER'`; gdy żaden wiersz nie pasuje, serwis ustala przyczynę (brak członka → 404, właściciel → 403). Duplikat członkostwa wykrywany jest przez błąd unikalności (`prisma-errors.ts`), więc równoległe dodania dają 409 `ALREADY_MEMBER`.
- **Alternatywy:** pole `ownerId` w `Board` (druga ścieżka prawdy, rozjazd z rolą); sama walidacja w serwisie (wyścigi); transakcje serializowalne (kosztowne, wymagają ponowień).
- **Konsekwencje:** inwariant „najwyżej jeden” egzekwuje baza; Prisma nie wyraża indeksów częściowych, więc indeks żyje tylko w SQL migracji (trzeba go zachować przy regenerowaniu migracji). Kaskada usunięcia użytkownika mogłaby osierocić tablicę — do rozstrzygnięcia przy dodaniu usuwania kont. Komentarz w `schema.prisma` mówi „exactly one” (poprawnie: „at most one”).

## ADR-028: Odraczalne ograniczenie unikalności `(boardId, rank)` i collation `"C"` pisane ręcznie w SQL
- **Kontekst:** `rank` musi mieć kolejność zgodną z porządkiem bajtowym z `shared` (ADR-022), a unikalność w kontenerze pozwala wykryć równoległe przypisanie tego samego klucza (`rankBetween` jest deterministyczne). Rebalansowanie przepisuje wszystkie klucze tablicy; zwykły indeks unikalny jest sprawdzany po każdym wierszu, więc `UPDATE` przesuwający klucze na klucze innych wierszy zawiódłby w stanie pośrednim.
- **Decyzja:** w migracji `20261111151836_columns` ręcznie: `"rank" TEXT COLLATE "C"` oraz `UNIQUE ("boardId", "rank") DEFERRABLE INITIALLY IMMEDIATE`. Zwykłe zapisy sprawdzają ograniczenie po każdej instrukcji (natychmiastowy, czysty P2002 do ponowienia), a rebalansowanie jest jedną instrukcją `UPDATE ... FROM unnest(ids, ranks)`, sprawdzaną na jej końcu. Prisma nie porównuje collation ani odraczalności — sprawdzono, że `migrate dev` nie zgłasza dryfu.
- **Alternatywy:** zwykły indeks unikalny i przepisywanie dwuetapowe (klucze tymczasowe, dwie instrukcje); `INITIALLY DEFERRED` (kolizja wykryta dopiero przy COMMIT, trudniejsza obsługa ponowień); brak unikalności i same blokady (kolizje wykrywane tylko dyscypliną kodu).
- **Konsekwencje:** schemat żyje częściowo tylko w SQL migracji (trzeba go zachować przy regenerowaniu migracji); test integracyjny porównuje `ORDER BY rank` w PostgreSQL z porządkiem w JS. To samo rozwiązanie należy zastosować dla kart w M3.

## ADR-029: Dwupoziomowa blokada wiersza tablicy (`FOR SHARE` / `FOR NO KEY UPDATE`)
- **Kontekst:** wstawienia i przeniesienia kolumn powinny móc działać równolegle, ale rebalansowanie (przepisanie wszystkich kluczy) oraz inwariant „co najmniej jedna kolumna `DONE`” wymagają wyłączności wśród zapisów kolumn tablicy.
- **Decyzja:** każda transakcja zapisująca kolumny zaczyna od blokady wiersza `Board`. Wstawienie i przeniesienie: `FOR SHARE` (wiele równolegle, kolizje łapie ograniczenie unikalności, do 5 prób, każda w osobnej transakcji READ COMMITTED, potem 409 `RANK_CONFLICT`). Rebalansowanie, usunięcie i zmiana ustawień: `FOR NO KEY UPDATE` (konfliktuje z `FOR SHARE` i z samym sobą, więc jest wyłączna wobec innych zapisów kolumn tablicy). Zwykłe zapytania o tablicę i członków nie są blokowane.
- **Alternatywy:** blokada wyłączna zawsze (brak współbieżności zapisów kolumn); poziom izolacji SERIALIZABLE z ponowieniami (kosztowny, szersze ponowienia); `pg_advisory_xact_lock` na tablicę (dodatkowy mechanizm poza modelem danych).
- **Konsekwencje:** zapisy kolumn jednej tablicy bywają kolejkowane za blokadą wyłączną; domyślny limit transakcji Prisma (`maxWait` 2 s, `timeout` 5 s) przy długiej kolejce mógłby dać błąd P2028 i 500 — rozwiązane w ADR-033.

## ADR-030: Eskalacja do rebalansowania przez restart próby, bez podnoszenia blokady
- **Kontekst:** o konieczności rebalansowania wiadomo dopiero po obliczeniu klucza (jest dłuższy niż `RANK_MAX_LENGTH`), gdy transakcja trzyma już blokadę współdzieloną. Podniesienie `FOR SHARE` do wyłącznej przy kilku równoległych transakcjach prowadzi do zakleszczenia.
- **Decyzja:** gdy obliczony klucz jest za długi, a trzymana jest blokada współdzielona, próba jest przerywana (`RebalanceRequiredError`, wycofanie transakcji) i powtarzana od początku z blokadą wyłączną; dopiero wtedy `rebalanceRanks(n)` przepisuje klucze (przy wstawieniu: jak gdyby nowa kolumna już była na swoim miejscu, a nowy klucz pochodzi z tego samego rozkładu) w tej samej transakcji.
- **Alternatywy:** zawsze blokada wyłączna (brak współbieżności); podniesienie blokady w trakcie transakcji (ryzyko zakleszczenia); rebalansowanie w tle (ADR-024).
- **Konsekwencje:** próba zużyta na eskalację wlicza się do limitu 5 prób; rebalansowanie jest rzadkie, więc koszt jest akceptowalny.

## ADR-031: Semantyka sąsiadów `afterId`/`beforeId` i złagodzone sprawdzanie po kolizji
- **Kontekst:** API nie przyjmuje surowych kluczy (ADR-024); klient opisuje miejsce przez sąsiadów, a jego widok może być nieaktualny. Dwa równoległe żądania „między A i B” obliczają ten sam klucz.
- **Decyzja:** `afterId` = kolumna bezpośrednio poprzedzająca, `beforeId` = bezpośrednio następująca; `null` = otwarty koniec listy, pominięcie = brak ograniczenia. Przy obu sąsiadach muszą być przyległe (z pominięciem przenoszonej kolumny), inaczej 409 `INVALID_POSITION`. Po kolizji klucza (to znaczy po wstawieniu innej kolumny w tę samą lukę) ponowienie wymaga jedynie, by `afterId` leżał przed `beforeId`, i umieszcza kolumnę bezpośrednio po `afterId` — dwa równoległe przeniesienia do jednej luki kończą się sukcesem. Sąsiad spoza tablicy: 404 `NEIGHBOR_NOT_FOUND`; kolumna jako własny sąsiad: 400. `RankError`: `INVALID_RANK`/`INVALID_ARGUMENT` → 400, `INVALID_RANK_ORDER` → 409.
- **Alternatywy:** docelowy indeks (wrażliwy na zmiany listy między odczytem a zapisem klienta); zawsze ścisła adjacencja (równoległe wstawienia w jedną lukę dawałyby 409 `INVALID_POSITION`); zwracanie błędu kolizji klientowi bez ponowień.
- **Konsekwencje:** klient z nieaktualnym widokiem dostaje 409 `INVALID_POSITION` i musi odświeżyć listę (M2d); po kolizji końcowa pozycja bywa „po `afterId`”, nie dokładnie pomiędzy dwoma pierwotnymi sąsiadami, jeśli ktoś wstawił kolumnę w lukę.

## ADR-032: Kontrakt odpowiedzi kolumn — `rank` w odpowiedzi, kategoria tylko przez `/settings`, rebalansowanie bez zmiany `updatedAt`
- **Kontekst:** kolumny mają kategorię (ADR-020) i limit WIP; trzeba ustalić, co widzi klient i kto zmienia kategorię (zmiana kategorii wpływa na analitykę i na inwariant `DONE`).
- **Decyzja:** odpowiedź zawiera `rank` (klient sortuje przez `<`). Nowa kolumna ma kategorię `TODO`; kategorię, limit WIP i tryb ścisły zmienia się wyłącznie przez `PATCH .../settings` (akcja `column:settings`: OWNER, ADMIN), a nazwę i pozycję — przez akcję `column:write`. Ostatnia kolumna `DONE` nie może być usunięta ani przekategoryzowana (409 `LAST_DONE_COLUMN`). Rebalansowanie nie zmienia `updatedAt`, bo kolejność kolumn się nie zmienia.
- **Alternatywy:** kategoria przy tworzeniu (rozmywa podział uprawnień); ukrycie `rank` (klient musiałby dopytywać o kolejność); `updatedAt` zmieniane przy rebalansowaniu (fałszywe sygnały zmiany).
- **Konsekwencje:** odpowiedź na przeniesienie zawiera jedną kolumnę, a rebalansowanie zmienia klucze innych — klient odświeża listę do czasu decyzji o zbiorczym zdarzeniu socket.

## ADR-033: Jawne limity czasu transakcji kolumn i odpowiedź 503 `BOARD_BUSY` bez ponowień po stronie serwera
- **Kontekst:** zapisy kolumn jednej tablicy są kolejkowane na blokadzie wiersza (ADR-029). Przy domyślnych opcjach Prisma długa kolejka kończyłaby się błędem P2028 i odpowiedzią 500, a czas oczekiwania na blokadę nie byłby ograniczony po stronie bazy.
- **Decyzja:** transakcje zapisu kolumn mają jawne limity (`ColumnTransactionOptions`, wstrzykiwane do `createColumnRepository(db, options)`, bez zmiennych środowiskowych): `lock_timeout` 2000 ms (ustawiany w transakcji przez `SELECT set_config('lock_timeout', $1, true)`, więc lokalnie dla transakcji), `maxWait` 2000 ms, `timeout` 5000 ms. Kod PostgreSQL 55P03 (`lock_not_available`) lub błąd Prisma P2028 jest tłumaczony na `BoardBusyError` (repozytorium), następnie na `ServiceUnavailableError` (serwis) i odpowiedź HTTP 503 z kodem `BOARD_BUSY` (`COLUMN_ERROR_CODES`). Serwer nie ponawia takiej próby. Dotyczy tworzenia, przeniesienia, zmiany ustawień i usunięcia; zmiana nazwy działa poza transakcją. Dodatkowo tylko P2002 na ograniczeniu `Column_boardId_rank_key` jest traktowany jako kolizja klucza (`isUniqueViolationOf` w `repositories/prisma-errors.ts` czyta `meta.target` albo `driverAdapterError.cause.constraint`, bo adapter pg w Prisma 7.10.1 nie ustawia `meta.target`).
- **Alternatywy:** domyślne opcje Prisma (500 przy kolejce, brak limitu oczekiwania na blokadę w bazie); ponowienia po stronie serwera (patrz niżej); zmienne środowiskowe dla limitów (więcej konfiguracji bez potrzeby); 409 zamiast 503.
- **Konsekwencje:** przeciążenie jest sygnalizowane klientowi szybko i czytelnie. Powody odrzucenia ponowień: to chwilowa rywalizacja o zasób, a nie konflikt stanu (w przeciwieństwie do `RANK_CONFLICT`, ADR-031), a ponawianie po stronie serwera wydłużałoby kolejkę i czas odpowiedzi (fail fast). Ograniczenia: P2028 obejmuje też wolną bazę i oczekiwanie na połączenie z puli, nie tylko blokady (wszystko mapuje się na `BOARD_BUSY`); `AppError` nie przenosi nagłówków, więc brak `Retry-After`.

## ADR-034: Przeniesienie kolumny w kliencie — optymistyczna zmiana, jedno żądanie naraz, częściowe wycofanie
- **Kontekst:** przeciąganie kolumn (M2d) musi dawać natychmiastową reakcję, a serwer przypisuje `rank` (ADR-031) i może go zmienić u innych kolumn przy rebalansowaniu (ADR-030). Odpowiedź REST zawiera tylko jedną kolumnę (ADR-032).
- **Decyzja:** `useBoardStore.moveColumn` (`apps/web/src/stores/board.ts`) najpierw zmienia kolejność lokalnie (`reorderList`), po czym wysyła `PATCH .../move`. Naraz trwa co najwyżej jedno przeniesienie (`isMovingColumn`; przeciąganie jest wyłączone w trakcie zapisu). Klient zawsze wysyła oba sąsiedztwa `afterId` i `beforeId` (`null` na krańcach listy; `getMoveNeighbors`). Przy błędzie wycofywana jest tylko przeniesiona kolumna na jej pierwotne miejsce w aktualnej liście (`restoreOriginalPosition`), dzięki czemu zachowane są zmiany wprowadzone w międzyczasie (np. zmiana nazwy). Po `INVALID_POSITION`, `NEIGHBOR_NOT_FOUND` i 404 lista kolumn jest pobierana ponownie.
- **Alternatywy:** wysyłanie tylko jednego sąsiada (serwer dostaje mniej informacji o intencji); kolejka wielu przeniesień (złożone, każde kolejne zależy od wyniku poprzedniego); wycofanie całej listy do migawki sprzed przeniesienia (gubi współbieżne zmiany); brak optymizmu (zauważalne opóźnienie).
- **Konsekwencje:** interfejs reaguje od razu, a stan po błędzie jest spójny. Cena: przeciąganie jest zablokowane na czas zapisu, a w razie rozjazdu z serwerem następuje pełne odświeżenie listy.

## ADR-035: Wykrywanie rebalansowania po stronie klienta przez porównanie `rank`
- **Kontekst:** po rebalansowaniu serwer zmienia `rank` także sąsiednich kolumn, ale odpowiedź zawiera jedną kolumnę (ADR-032). Zbiorcze zdarzenie socket nie jest jeszcze zaprojektowane.
- **Decyzja:** po udanym przeniesieniu klient sprawdza, czy zwrócony `rank` mieści się między sąsiadami z lokalnej listy (`prev.rank < moved.rank < next.rank`; `rankFitsAt`/`rankFitsBetween` w `columnOrder.ts`); jeśli nie, pobiera całą listę. To samo dotyczy `createColumn` (dopisanie na końcu; gdy `rank` nie jest większy od ostatniego — odświeżenie). Porównanie to ścisłe `<` na łańcuchach, nigdy `localeCompare` (kolejność ma być zgodna z `COLLATE "C"`, patrz M2a).
- **Alternatywy:** zawsze odświeżać listę po przeniesieniu (dodatkowe żądanie przy każdej operacji); zmiana kontraktu API tak, by zwracał wszystkie zmienione kolumny (poza zakresem M2d); czekanie na zdarzenie socket.
- **Konsekwencje:** zwykłe przeniesienie kosztuje jedno żądanie. Wykrywanie jest „best-effort”: gdy po rebalansowaniu przeniesiona kolumna nadal mieści się między lokalnymi sąsiadami, a inne kolumny mają nowe klucze, klient zobaczy to dopiero po następnym odświeżeniu. Wyświetlana kolejność pozostaje poprawna, bo rebalansowanie ją zachowuje.

## ADR-036: Natywny `<select>` zamiast Radix Select
- **Kontekst:** okno ustawień kolumny wymaga wyboru kategorii spośród czterech wartości. Pozostałe komponenty shadcn/ui opierają się na Radix.
- **Decyzja:** komponent `native-select` (`apps/web/src/components/ui/native-select.tsx`) opakowuje natywny `<select>` w stylu shadcn.
- **Alternatywy:** Radix Select (własny listbox i portal, trudniejsze testy w jsdom).
- **Konsekwencje:** pełna dostępność i obsługa klawiatury, proste testy w jsdom; wygląd rozwijanej listy zależy od przeglądarki. Przy większej liczbie opcji lub wyszukiwaniu można wrócić do Radix Select.

## ADR-037: `/` przekierowuje do `/boards`; lista tablic jest stroną startową
- **Kontekst:** strona `/` pokazywała tylko stan API (`HomePage`, `useApiHealth`), co po M2d traci sens.
- **Decyzja:** `HOME_PATH` w `features/auth/redirect.ts` wynosi `/boards`; trasa `/` przekierowuje do `/boards`, a `HomePage` i `useApiHealth` usunięto. Nagłówek (`AppLayout`) zawiera link z nazwą aplikacji do `/boards`.
- **Alternatywy:** zostawić `HomePage` jako stronę powitalną (zbędny ekran przed listą tablic).
- **Konsekwencje:** po zalogowaniu użytkownik trafia od razu do swoich tablic; zaktualizowano testy auth. `GET /health` pozostaje w API (bez klienta w web).

## ADR-038: Mechanizm motywu — klucz `localStorage`, skrypt przed Reactem, store Zustand, tryb systemowy
- **Kontekst:** DESIGN §8 wymaga motywu jasnego, ciemnego i systemowego bez migotania przy ładowaniu strony.
- **Decyzja:** wybór (`light`/`dark`/`system`) jest zapisywany w `localStorage` pod kluczem `streamline-theme`. Mały skrypt w `apps/web/index.html` ustawia klasę `dark` na `<html>` przed startem Reacta. Logikę utrzymuje `src/lib/theme.ts`, a stan `src/stores/theme.ts` (Zustand); tryb `system` śledzi `prefers-color-scheme`. Test sprawdza, że skrypt w `index.html` jest zgodny z `theme.ts`.
- **Alternatywy:** ustawianie klasy dopiero w efekcie Reacta (widoczne mignięcie jasnego motywu); ciasteczko (zbędne, wybór dotyczy tylko klienta); biblioteka do motywów (dodatkowa zależność dla kilkunastu linii).
- **Konsekwencje:** brak migotania; reguła jest zdublowana w skrypcie i w module, co pilnuje test. Skrypt wbudowany w HTML utrudniłby w przyszłości ścisłą politykę CSP bez `nonce`/hasha.

## ADR-039: Komunikaty (`notice`, nieudana zmiana nazwy) jako toasty sonner ze stałym identyfikatorem
- **Kontekst:** po przebudowie komunikaty o błędach operacji (np. `BOARD_BUSY`, wycofana zmiana nazwy) nie mają stałego miejsca w układzie.
- **Decyzja:** `notice` ze store tablic i nieudana optymistyczna zmiana nazwy są wyświetlane jako toasty biblioteki sonner (`AppToaster`, prawy dolny róg) ze stałym `id`, a `notice` jest czyszczone istniejącą akcją `dismissNotice`. Błędy walidacji formularzy pozostają w treści formularza.
- **Alternatywy:** pasek komunikatu w układzie strony (przesuwa treść); osobny toast na każde zdarzenie (duplikaty, m.in. przy podwójnym wywołaniu efektów w StrictMode).
- **Konsekwencje:** powtórzony komunikat zastępuje poprzedni zamiast się mnożyć, a w StrictMode pojawia się jeden toast (test). Nowa zależność `sonner` 2.0.8.

## ADR-040: Dedykowany uchwyt przeciągania kolumny zamiast przeciągania za nagłówek
- **Kontekst:** DESIGN §2 zakładał przeciąganie za nagłówek. `@hello-pangea/dnd` nadaje elementowi uchwytu `role=button`, co ukrywa dla czytników ekranu jego zawartość (nazwę kolumny i licznik).
- **Decyzja:** uchwytem jest osobna ikona `GripVertical`; nagłówek zachowuje semantykę nagłówka. DESIGN §2 zaktualizowano.
- **Alternatywy:** uchwyt na całym nagłówku (utrata dostępności nazwy i licznika); własna obsługa przeciągania (poza zakresem).
- **Konsekwencje:** nazwa i licznik pozostają dostępne; obszar chwytu jest mniejszy niż cały nagłówek. Przeciąganie klawiaturą działa tak jak wcześniej.

## ADR-041: Menu niemodalne i powrót fokusu do przycisku ⋯
- **Kontekst:** okna dialogowe otwierane z menu rozwijanego tracą fokus po zamknięciu menu.
- **Decyzja:** menu rozwijane (Radix) są niemodalne, a okna otwarte z menu zwracają fokus na przycisk ⋯ przez `returnFocusRef`.
- **Alternatywy:** menu modalne (blokuje resztę strony i komplikuje współpracę z oknami); pozostawienie domyślnego zachowania (fokus ginie na `body`).
- **Konsekwencje:** zachowana nawigacja klawiaturą; dodatkowa referencja w komponentach z menu (pokryta testami).

## ADR-042: Rozdzielenie `--danger` (wypełnienia) i `--destructive` (tekst) ze względu na kontrast AA
- **Kontekst:** `--danger` `#E34935` z DESIGN §6 ma na bieli kontrast poniżej 4,5:1 (WNF-05).
- **Decyzja:** `--danger` służy do wypełnień i obramowań, a czerwony tekst używa `--destructive`: `#AE2E24` (jasny) / `#F87168` (ciemny). W ciemnym motywie przyciski primary i destructive mają ciemny tekst.
- **Alternatywy:** ściemnić `--danger` (zmienia wypełnienia i odbiega od palety); zostawić kontrast poniżej AA.
- **Konsekwencje:** zgodność z AA dla tekstu; dwa odcienie czerwieni w systemie wizualnym (opisane w DESIGN §6).

## ADR-043: Karty — blokada wyłączna tablicy przy tworzeniu, współdzielona przy przeniesieniu, stała kolejność blokad
- **Kontekst:** karty dzielą z kolumnami blokadę wiersza tablicy (ADR-028). Numer karty pochodzi z licznika `Board.nextCardNumber`, czyli z `UPDATE` wiersza tablicy. Transakcja trzymająca `FOR SHARE` i wykonująca `UPDATE ... RETURNING` tego samego wiersza zakleszcza się z drugą taką transakcją (każda czeka na zwolnienie blokady współdzielonej drugiej).
- **Decyzja:** utworzenie karty od początku bierze blokadę wyłączną (`FOR NO KEY UPDATE`) i w niej zwiększa licznik, wstawia kartę i zapisuje pierwsze `CardTransition`; ewentualne rebalansowanie kolumny odbywa się w tej samej blokadzie. Przeniesienie bierze blokadę współdzieloną (równoległe przeniesienia różnych kart), następnie blokuje wiersz przenoszonej karty (`FOR NO KEY UPDATE`, z `boardId` w warunku), a dopiero potem — gdy karta wchodzi do kolumny ze ścisłym WIP — wiersz kolumny (ADR-044). Kolejność blokad jest zawsze: tablica → karta → kolumna. Gdy nowy `rank` byłby za długi, próba kończy się `RebalanceRequiredError` i jest powtarzana z blokadą wyłączną (ADR-030). `PATCH` i `DELETE` karty nie biorą blokady tablicy: nie zmieniają `rank` ani liczby kart w kolumnie. Numery nie są ponownie używane (usunięta karta zostawia lukę; wycofana transakcja zwraca numer).
- **Alternatywy:** `MAX(number) + 1` (wyścig, ponowne użycie numeru po usunięciu); sekwencja PostgreSQL na tablicę (osobny obiekt na każdą tablicę); podnoszenie blokady współdzielonej do wyłącznej (zakleszczenie); blokada wyłączna także przy przeniesieniach (serializuje wszystkie ruchy kart).
- **Konsekwencje:** tworzenie kart jednej tablicy jest szeregowane (akceptowalne: operacja rzadsza niż przenoszenie), przeniesienia działają równolegle; brak cykli blokad (w przeglądzie sprawdzono każdą parę ścieżek); zapis w kolejce dłuższej niż `lock_timeout` kończy się 503 `BOARD_BUSY` (ADR-033). Testy równoległe: tworzenie/tworzenie, przeniesienie/przeniesienie, tworzenie/przeniesienie.

## ADR-044: Blokada wiersza kolumny przy wejściu do kolumny ze ścisłym limitem WIP
- **Kontekst:** limit WIP w trybie ścisłym (`wipStrict`) ma być twardy: dwie równoległe operacje nie mogą zająć ostatniego wolnego miejsca. Przy `READ COMMITTED` samo `COUNT` bez blokady nie widzi niezatwierdzonych kart drugiej transakcji, więc obie uznałyby, że miejsce jest wolne.
- **Decyzja:** karta wchodząca do kolumny (utworzenie lub przeniesienie z innej kolumny) ze `wipStrict` i `wipLimit` blokuje wiersz tej kolumny (`FOR NO KEY UPDATE`; zgodne z `FOR KEY SHARE` z kontroli kluczy obcych) i dopiero potem liczy karty. Druga transakcja czeka na zatwierdzenie pierwszej i liczy już jej kartę. Zmiana kolejności w tej samej kolumnie nie sprawdza limitu. Ustawienia WIP zmieniają się wyłącznie pod blokadą wyłączną tablicy, więc przy trzymanej blokadzie współdzielonej są stabilne; ta sama blokada uniemożliwia usunięcie kolumny w trakcie dodawania do niej karty. Przekroczenie: 409 `WIP_LIMIT_EXCEEDED`. W trybie nieścisłym limit tylko ostrzega (licznik w interfejsie).
- **Alternatywy:** poziom `SERIALIZABLE` (ponowienia i błędy serializacji w całym API); blokada wszystkich kart kolumny (więcej wierszy); licznik kart w wierszu kolumny (dodatkowa denormalizacja); sprawdzanie tylko w interfejsie.
- **Konsekwencje:** transakcja przeniesienia trzyma co najwyżej jeden wiersz kolumny (docelowej), więc nie ma cyklu; blokada dotyczy tylko kolumn ze ścisłym limitem. Wyścigi pokrywają testy równoległe (przeniesienie/przeniesienie i utworzenie/przeniesienie ustawione w kolejce za trzymaną blokadą tablicy).

## ADR-045: Spójność osoby przypisanej do karty — blokada członkostwa i odpinanie przy usunięciu członka
- **Kontekst:** kartę może mieć przypisaną tylko osoba będąca członkiem tablicy. Sprawdzenie członkostwa i zapis przypisania są osobnymi instrukcjami, więc równoległe usunięcie członka mogłoby zostawić kartę przypisaną do nie-członka.
- **Decyzja:** przypisanie (utworzenie lub `PATCH`) blokuje wiersz członkostwa `FOR SHARE` przed zapisem; brak członkostwa daje 400 `ASSIGNEE_NOT_MEMBER`. Usunięcie członka (`removeUnlessOwner`) jest transakcją: najpierw `DELETE` członkostwa (czeka na transakcje trzymające wiersz), potem `UPDATE Card SET assigneeId = NULL` dla jego kart na tej tablicy. Kolejność blokad: członkostwo → karta. Usunięcie konta ustawia `assigneeId` na `NULL` kluczem obcym (`SetNull`).
- **Alternatywy:** wyzwalacz w bazie (logika poza kodem aplikacji); pozostawienie przypisania po odejściu członka (karta należy do osoby bez dostępu); odrzucenie usunięcia członka z przypisanymi kartami (blokuje zarządzanie).
- **Konsekwencje:** karta nigdy nie wskazuje nie-członka (test równoległy); usunięcie członka przechodzi przez runner transakcji, więc ma `lock_timeout` i może dać 503 `BOARD_BUSY`.

## ADR-046: Klucz tablicy unikalny w obrębie właściciela; klucz karty `<klucz>-<numer>`
- **Kontekst:** karty mają być adresowalne czytelnym kluczem (`SL-12`, jak w Jirze), a klucz ma wynikać z tablicy. Globalna unikalność kluczy wymuszałaby wymyślanie coraz mniej czytelnych skrótów.
- **Decyzja:** `Board.key` (2–5 wielkich liter ASCII) jest unikalny razem z `ownerId` (`@@unique([ownerId, key])`); `ownerId` jest kluczem obcym z `onDelete: Restrict`, utrzymywanym zgodnie z członkostwem `OWNER`. Klucz generuje serwer z nazwy (`pickBoardKey`: inicjały słów, a dla jednego słowa dwie pierwsze litery; polskie i cyrylickie litery są transliterowane; `BRD` bez użytecznych liter; przy kolizji kolejni kandydaci — rozszerzenia nazwy, potem sufiksy literowe), a użytkownik może go zmienić (`PATCH /boards/:boardId`, akcja `board:rename`; 409 `BOARD_KEY_TAKEN`). Karta zapisuje tylko `number`; klucz `SL-12` jest składany przy odczycie z aktualnego klucza tablicy, więc zmiana klucza zmienia klucze wszystkich kart bez migracji danych. `GET .../cards/:cardKey` z innym prefiksem niż aktualny klucz tablicy daje 404 (stare klucze nie są przekierowywane), a niepoprawny format — 400. Tworzenie tablicy ponawia próbę z następnym kandydatem, gdy równoległe żądanie tego samego użytkownika zajęło klucz (do 5 prób).
- **Alternatywy:** klucz globalnie unikalny (kolizje popularnych skrótów); klucz zapisany w każdej karcie (aktualizacja wszystkich kart przy zmianie klucza); adresowanie kart wyłącznie po `id` (nieczytelne dla użytkownika); klucz niezmienny (brak możliwości poprawienia skrótu).
- **Konsekwencje:** dwie tablice różnych właścicieli mogą mieć ten sam klucz, ale tablice jednego właściciela — nie. Przekazanie własności (WF-BOARD-04) musi zaktualizować `ownerId` i może zakończyć się `BOARD_KEY_TAKEN`. `Restrict` blokuje usunięcie konta właściciela, dopóki ma tablice — do rozstrzygnięcia przy usuwaniu kont. Stare adresy kart przestają działać po zmianie klucza.

## ADR-047: Rejestr przejść z kluczami obcymi `SetNull` oraz zakaz usuwania niepustej kolumny
- **Kontekst:** `CardTransition` jest źródłem analityki przepływu (ADR-020) i powinien przetrwać zmiany struktury tablicy. Jednocześnie usunięcie kolumny z kartami zostawiłoby karty bez kolumny.
- **Decyzja:** `CardTransition.fromColumnId`, `toColumnId` i `movedById` są opcjonalnymi kluczami obcymi z `onDelete: SetNull` (przejście zostaje po usunięciu kolumny lub konta); `cardId` i `boardId` działają kaskadowo (usunięcie karty lub tablicy usuwa jej historię). Pierwsze przejście karty ma `fromColumnId = null`. Przejście jest zapisywane w tej samej transakcji co tworzenie lub przeniesienie i tylko przy zmianie kolumny (zmiana kolejności w kolumnie i przeniesienie na bieżącą pozycję nic nie zapisują). Kolumny `fromColumnId` i `toColumnId` mają indeksy, bo bez nich `SetNull` przy usunięciu kolumny skanowałby całą tabelę przejść pod blokadą wyłączną tablicy; wspomogą też zapytania analityczne. `Card.columnId` ma `onDelete: NoAction`, a serwis odmawia usunięcia kolumny z kartami (409 `COLUMN_NOT_EMPTY`) pod blokadą wyłączną tablicy (zapisy kart biorą co najmniej współdzieloną).
- **Alternatywy:** kaskadowe usuwanie przejść razem z kolumną (utrata historii); automatyczne przenoszenie kart do innej kolumny (zaskakujące, wpływa na metryki); usuwanie kart wraz z kolumną (utrata danych).
- **Konsekwencje:** przejścia z `NULL` w `fromColumnId`/`toColumnId` oznaczają usuniętą kolumnę (zapytania analityczne muszą to uwzględnić); usunięcie kolumny wymaga najpierw przeniesienia lub usunięcia kart.

## ADR-048: Migracja uzupełniająca klucze i właścicieli istniejących tablic, z naprawą danych i komunikatami
- **Kontekst:** `Board.key` i `Board.ownerId` są wymagane, a w bazie mogą być tablice utworzone przed M3a. Dodanie kolumn jako `NOT NULL` bez wartości by się nie powiodło.
- **Decyzja:** migracja `20261112014500_cards_and_board_keys` dodaje kolumny jako opcjonalne, uzupełnia je blokiem PL/pgSQL (`DO`) i dopiero potem ustawia `NOT NULL`. Blok: (1) usuwa tablice bez członków (nieosiągalne przez API), (2) nadaje rolę `OWNER` najwyżej uprzywilejowanemu, najdawniej dołączonemu członkowi tablicy bez właściciela, (3) ustawia `ownerId` z członkostwa `OWNER`, (4) wylicza `key` przybliżeniem generatora z `packages/shared` (transliteracja, inicjały, `BRD`) z deduplikacją sufiksami literowymi w obrębie właściciela. Liczby usuniętych tablic i awansowanych członków są zgłaszane przez `RAISE NOTICE`. Test (`apps/api/src/test/board-key-backfill.test.ts`) uruchamia dokładnie ten blok z pliku migracji (znaczniki `backfill:begin`/`backfill:end`) w wycofywanej transakcji. Migrację edytowano w miejscu przed zmergowaniem (komunikaty, indeksy).
- **Alternatywy:** przerwanie migracji (`RAISE EXCEPTION`) przy niespójnych danych (bezpieczniejsze, ale wymaga ręcznej naprawy; takie dane mogą powstać tylko przez bezpośredni zapis do bazy, a projekt nie ma danych produkcyjnych); ten sam klucz `BRD` dla wszystkich istniejących tablic; generowanie kluczy poza migracją (okres, w którym schemat jest niezgodny z kodem).
- **Konsekwencje:** migracja może usunąć dane i nadać `OWNER`, co jest widoczne w logu migracji; generator SQL nie jest identyczny z TypeScriptowym (znany dług); osoba, która zastosowała wcześniejszy szkic migracji lokalnie, musi zresetować bazę deweloperską. Po zmergowaniu migracji nie wolno jej już zmieniać.

## ADR-049: Wspólny runner transakcji i pomocniki zapisu z `rank` dla kolumn i kart
- **Kontekst:** po dodaniu kart logika z M2c (blokada wiersza tablicy, limity czasu, mapowanie P2002 i 55P03/P2028, ponowienia, rebalansowanie, rozwiązywanie `afterId`/`beforeId`) byłaby powielona w repozytoriach i serwisach kolumn i kart. Dodatkowo odczyt kart całej tablicy wymaga jednej migawki.
- **Decyzja:** `apps/api/src/repositories/board-transaction.ts` zawiera `lockBoardRow`, `BoardLockMode`, `RankCollisionError`, `BoardBusyError`, domyślne limity (ADR-033) i `createBoardTransactionRunner(db, rankConstraint | null, options)`; runner ustawia `lock_timeout` lokalnie dla transakcji, przyjmuje poziom izolacji per wywołanie (`GET .../cards` używa `RepeatableRead`, więc klucz tablicy, kolumny i karty pochodzą z jednej migawki) i mapuje błędy. Ograniczenie rank jest opcjonalne (`null` dla usunięcia karty i członka). `apps/api/src/services/ranked-write.ts` zawiera `withRankRetry`, `inLockedBoard`, `resolveInsertIndex`, `rankAt`, `rebalanceAroundNewItem`, `rebalancedAssignments`, `isAtIndex` i `busyToServiceUnavailable`. Schematy `afterId`/`beforeId` są wspólne (`packages/shared/src/schemas/position.ts`). Zachowanie kolumn się nie zmieniło (testy kolumn bez zmian); reeksporty w `column.repository` i `column.service` zostawiono dla zgodności.
- **Alternatywy:** kopiowanie kodu kolumn do kart (rozjazd poprawek); klasa bazowa serwisu (dziedziczenie zamiast prostych funkcji); trzy równoległe zapytania bez transakcji (brak wspólnej migawki, karta przeniesiona do świeżo utworzonej kolumny mogłaby zniknąć z odpowiedzi — tak działała pierwsza wersja).
- **Konsekwencje:** poprawki współbieżności trafiają do obu agregatów naraz; odczyt `GET .../cards` jest spójny. Gwarancję migawki daje poziom izolacji; nie ma deterministycznego testu przeplotu.

## ADR-050: Przeniesienie karty w kliencie — protokół z ADR-034/035 i drugi licznik generacji listy kart
- **Kontekst:** karty przeciąga się w obrębie kolumny i między kolumnami; odpowiedź `PATCH .../cards/:id/move` zawiera jedną kartę, a rebalansowanie zmienia `rank` pozostałych kart kolumny. Lista kart (`GET .../cards`) ładuje się niezależnie od kolumn i może być w locie w chwili przeniesienia.
- **Decyzja:** `useBoardStore.moveCard` stosuje ten sam protokół co `moveColumn` (ADR-034, ADR-035): zmiana optymistyczna (`moveCardLocally`), zawsze oba `afterId`/`beforeId` (`null` na krańcach), jedno przeniesienie naraz (`isMovingCard`), przy błędzie wycofanie tylko przeniesionej karty (`restoreCardPosition` nie cofa edycji wykonanych w międzyczasie), odświeżenie listy po `INVALID_POSITION`/`NEIGHBOR_NOT_FOUND`/404 oraz po wykryciu rebalansowania (ścisłe `prev.rank < rank < next.rank`, nigdy `localeCompare`). Dodatkowo `cardsGeneration` odrzuca listę kart zażądaną przed przeniesieniem (zwiększany na jego początku). 404 `COLUMN_NOT_FOUND` przy tworzeniu lub przeniesieniu odświeża kolumny i karty („The board has been refreshed.”). Przeciąganie ma jeden `DragDropContext` z typami `COLUMN` i `CARD`; kolumna docelowa to `Droppable` `cards:<columnId>`. Błędy `WIP_LIMIT_EXCEEDED`, `COLUMN_NOT_EMPTY`, `ASSIGNEE_NOT_MEMBER` mają czytelne komunikaty (`cardErrorMessage.ts`, `boardErrorMessage.ts`).
- **Alternatywy:** czekanie na odpowiedź serwera przed zmianą widoku (opóźnienie przy każdym upuszczeniu); zawsze pełne odświeżenie listy po przeniesieniu (migotanie, dodatkowe żądanie); wspólny licznik generacji dla kolumn i kart (odrzucałby listę kart przy przeniesieniu kolumny).
- **Konsekwencje:** zachowanie kart i kolumn jest spójne i testowane tymi samymi technikami (prawdziwe przeciąganie klawiaturą). Wykrywanie rebalansowania pozostaje „best-effort” — niewykryte rozbieżności naprawi najbliższe odświeżenie po 409. Do czasu Socket.io zmiany innych użytkowników są widoczne po przeładowaniu.

## ADR-051: Szczegóły karty jako zagnieżdżona trasa modalna `/boards/:boardId/cards/:cardKey`
- **Kontekst:** DESIGN §4 wymaga modalu z adresem do udostępnienia. Adres może zostać otwarty bezpośrednio (karta nie jest jeszcze w store) lub z tablicy (karta już jest).
- **Decyzja:** trasa zagnieżdżona w `BoardPage` (`<Outlet />`), dzięki czemu tablica pozostaje pod modalem. `CardDetailsRoute` szuka karty w store po kluczu; jeśli jej brak (i lista kart jest gotowa), pobiera ją przez `GET .../cards/:cardKey`; niepoprawny klucz (`cardKeySchema`) nie wywołuje żądania; brak karty → „Card not found”. Zamknięcie: `navigate(-1)`, gdy modal otwarto z tablicy (stan lokalizacji `CardLocationState`, `isOpenedFromBoard` w `paths.ts`), w przeciwnym razie `navigate(boardPath, { replace: true })` — Wstecz w przeglądarce nie otwiera ponownie zamkniętego modalu. Pola zapisują się pojedynczo (tytuł: Enter/blur; opis: blur; termin: Enter/blur; priorytet, osoba, kolumna: zmiana); Esc na zmienionym polu najpierw je odrzuca. Dla roli bez `card:write` modal jest tylko do odczytu (bez „Delete”; „Copy link” dostępne).
- **Alternatywy:** stan modalu w store bez trasy (brak linku); osobna strona karty (utrata kontekstu tablicy); zapis całego formularza przyciskiem „Save” (wolniejsza praca, trudniejsze wycofanie).
- **Konsekwencje:** link `SL-12` działa po bezpośrednim otwarciu. Brak testu historii przeglądarki przy zamykaniu (dług techniczny).

## ADR-052: Filtr kart wyłącza przeciąganie, a liczniki kolumn pokazują sumę bez filtra
- **Kontekst:** przy aktywnym filtrze widać tylko część kart, więc sąsiedzi widoczni w interfejsie nie są sąsiadami w danych — `afterId`/`beforeId` wskazywałyby złą lukę. Limit WIP dotyczy wszystkich kart kolumny.
- **Decyzja:** filtr (tekst w tytule lub kluczu; osoby — wielokrotny wybór awatarów członków) działa lokalnie po stronie klienta (`cardFilter.ts`). Gdy jest aktywny, przeciąganie kart jest wyłączone (podpowiedź „Clear the filter to drag cards.”), a licznik kolumny i podświetlenie przekroczonego WIP pokazują sumę bez filtra. Karta utworzona w trakcie filtrowania, której filtr nie przepuszcza, daje w kompozytorze podpowiedź.
- **Alternatywy:** przeciąganie po przefiltrowanej liście z wyznaczaniem sąsiadów w pełnych danych (zaskakujące miejsce upuszczenia); liczniki przefiltrowane (mylące względem limitu WIP).
- **Konsekwencje:** prosty i bezpieczny model; użytkownik musi wyczyścić filtr, by zmienić kolejność. Filtr nie obejmuje etykiet (brak etykiet — WF-CARD-05).

## ADR-053: Edycja pól karty — optymistyczna, z numerami kolejności per `<cardId>:<pole>`
- **Kontekst:** pola karty zapisują się pojedynczo i mogą być zmieniane szybko (kilka żądań `PATCH` równolegle); odpowiedzi i błędy mogą wracać w dowolnej kolejności. Pierwsza wersja nadpisywała nowszą wartość starszą odpowiedzią i wycofywała pole ponad nowszym udanym zapisem (wykryte w przeglądzie kodu).
- **Decyzja:** `updateCard` numeruje każdą edycję osobno dla pary `<cardId>:<pole>` (`latestCardEdits`). Odpowiedź serwera jest scalana tylko w pola, dla których jest to najnowsza edycja i nie ma nowszej w locie (`mergeEditResponse`); kolumna i `rank` pozostają lokalne (zmieniają je tylko przeniesienia). Wycofanie po błędzie obejmuje tylko pola, których ta edycja była najnowsza. Wysyłane są tylko zmienione pola (`cardChanges.ts`, walidacja `updateCardSchema`). Pole terminu zapisuje się po Enter/blur, a nie przy każdym zdarzeniu `input` (przeglądarka zgłasza je dla każdej cyfry roku). 404 przy edycji usuwa kartę lokalnie.
- **Alternatywy:** blokowanie interfejsu do czasu odpowiedzi (sprzeczne z DESIGN §9); jeden numer na kartę (niepotrzebnie odrzucałby odpowiedzi dotyczące innych pól); kolejka żądań (opóźnienia).
- **Konsekwencje:** znana luka: dwie nakładające się nieudane edycje tego samego pola mogą zostawić na ekranie starszą wartość optymistyczną (poprawka: przywrócić pole lub `reloadCards()`); `latestCardEdits` nie jest czyszczony (znikomy wzrost).

## ADR-054: Ponowne odczytanie listy kart po mutacji wykonanej w trakcie ładowania listy
- **Kontekst:** lista kart ładuje się równolegle z kolumnami, a kompozytor jest dostępny, gdy karty jeszcze pokazują szkielety. Spóźniona lista mogłaby usunąć świeżo utworzoną kartę lub przywrócić usuniętą. Samo zwiększenie `cardsGeneration` zostawiłoby `cardsStatus` na `loading`.
- **Decyzja:** po utworzeniu, edycji, usunięciu i przeniesieniu karty store sprawdza, czy żądanie listy jest w locie (`pendingCardsLoad`), i wtedy uruchamia odczyt ponownie (`reloadIfListInFlight`); status kończy się poprawnie jako `ready`. Przy pierwszym nieudanym ładowaniu pokazywany jest tylko baner „Could not load the cards.” z „Retry”; kolejne błędy to toast.
- **Alternatywy:** dołączanie mutacji do spóźnionego wyniku (złożone, podatne na błędy); blokowanie tworzenia do końca ładowania.
- **Konsekwencje:** jedno dodatkowe żądanie w rzadkim przeplocie; brak zniknięć i „zmartwychwstań” kart.
