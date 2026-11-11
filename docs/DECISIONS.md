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
