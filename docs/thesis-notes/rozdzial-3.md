# Rozdział 3 — Projekt (notatki)

## 3.x Projekt uwierzytelniania i autoryzacji
- Model dwóch tokenów (ADR-004): krótkotrwały access token JWT (15 min) przechowywany w pamięci aplikacji (nie w `localStorage`, co ogranicza skutki XSS) oraz długotrwały refresh token (7 dni) w ciasteczku `httpOnly`.
- Refresh token jest nieprzezroczysty (losowy), a w bazie przechowywany jest tylko jego skrót SHA-256; każde odświeżenie wymienia token na nowy (rotacja). Powiązanie `RefreshToken.replacedById` tworzy łańcuch wymian.
- Atrybuty ciasteczka: `SameSite=Strict` (ochrona przed CSRF; możliwe, bo SPA i API są w tej samej witrynie dzięki ścieżce `/api`), `Secure` w produkcji, `Path=/api/auth` (ciasteczko nie jest wysyłane do pozostałych endpointów) — ADR-012.
- Schemat decyzyjny odświeżania: patrz diagram sekwencji w `docs/ARCHITECTURE.md`; uzasadnienie okna łaski w ADR-013.
- Model danych: diagram ER w `docs/DATABASE.md` (`User`, `RefreshToken`).
- Wybory technologiczne z uzasadnieniem: bcryptjs (brak natywnej kompilacji, ADR-010), jose (ADR-011).
- Zagrożenia i środki zaradcze: kradzież refresh tokena (rotacja + wykrywanie ponownego użycia), zgadywanie haseł (ograniczanie liczby żądań, bcrypt koszt 12), wyliczanie kont (jednakowa odpowiedź logowania, wyrównanie czasu; kompromis przy rejestracji), wyciek bazy (skróty tokenów, hasła w bcrypt), XSS (access token w pamięci, refresh `httpOnly`).

## 3.x Projekt uwierzytelniania po stronie klienta (M1b)
- Access token wyłącznie w stanie aplikacji (Zustand); po przeładowaniu strony sesja jest przywracana żądaniem `/auth/refresh` z ciasteczkiem `httpOnly` (ADR-004, ADR-018).
- Odświeżanie tokena: interceptor HTTP z jednym wspólnym odświeżeniem dla równoległych żądań 401 (single-flight) i jednym opóźnionym ponowieniem po `REFRESH_TOKEN_ROTATED`, spójnie z oknem łaski serwera (ADR-013, ADR-016). Diagram sekwencji: `docs/ARCHITECTURE.md`.
- Spójność sesji: „epoka sesji” odrzuca wyniki odświeżania z poprzedniej sesji; wylogowanie czeka na odświeżenie w toku (ADR-017).
- Trasy: komponenty `ProtectedRoute` i `GuestRoute`; adres powrotny po zalogowaniu akceptowany tylko, jeśli jest ścieżką wewnątrz aplikacji (ochrona przed otwartym przekierowaniem).
- Walidacja formularzy tymi samymi schematami Zod co backend (`loginSchema`, `registerSchema`).

## 3.x Projekt modelu ról i uprawnień do tablicy (M2b)
- Role: `OWNER`, `ADMIN`, `MEMBER`, `VIEWER`; uprawnienia opisuje macierz akcja → dozwolone role (`BOARD_PERMISSIONS`, `packages/shared/src/permissions.ts`), zgodna z macierzą z `docs/REQUIREMENTS.md`. Macierz jest czystą strukturą danych, więc ta sama reguła obsługuje API i interfejs oraz da się ją testować jednostkowo.
- Dwie warstwy egzekwowania (ADR-025): middleware `requireBoardPermission` (wczesne odrzucenie) i ponowne sprawdzenie w serwisie (obrona w głąb).
- Ochrona przed wyliczaniem zasobów (ADR-026): nie-członek dostaje 404 identyczne jak dla nieistniejącej tablicy; 403 tylko dla członka bez uprawnienia.
- Inwariant „jeden właściciel” (ADR-027): częściowy indeks unikalny w PostgreSQL + zapytania warunkowe; właściciel nie może opuścić tablicy (409), co wymaga przyszłej akcji przekazania własności (`board:transfer`).
- Model danych: diagram ER w `docs/DATABASE.md` (`Board`, `BoardMember`, relacja wiele-do-wielu `User`–`Board` z atrybutem roli).

## 3.x Projekt modelu kolumn i porządkowania elementów (M2c)
- Kolumna ma kategorię przepływu (`BACKLOG`, `TODO`, `IN_PROGRESS`, `DONE`), limit WIP (`wipLimit`, 1–999 lub brak) i tryb ścisły (`wipStrict`: blokuje albo tylko ostrzega). Analityka przepływu i reguły WIP opierają się na kategorii, a nie na swobodnie edytowalnym tytule (ADR-020). Nowa tablica otrzymuje cztery kolumny domyślne (Backlog, To Do, In Progress, Done); tablica musi zawsze mieć co najmniej jedną kolumnę `DONE` (inwariant egzekwowany w serwisie pod blokadą, ADR-029).
- Podział uprawnień: nazwa, pozycja i usuwanie kolumny — `column:write` (OWNER, ADMIN, MEMBER); kategoria, limit WIP i tryb ścisły — `column:settings` (OWNER, ADMIN).
- Kolejność: pole `rank` (tekst, `COLLATE "C"`) zamiast pozycji całkowitoliczbowych. Klient opisuje miejsce przez sąsiadów (`afterId`/`beforeId`), a klucz liczy serwer z zapisanych sąsiadów (ADR-031); dzięki temu przeniesienie zmienia jeden wiersz, a nie renumeruje listy.
- Współbieżność: ponieważ `rankBetween` jest deterministyczne, dwa równoległe wstawienia w tę samą lukę dają ten sam klucz. Kolizję wykrywa ograniczenie unikalności `(boardId, rank)` (odraczalne, ADR-028), a żądanie jest ponawiane z aktualnymi danymi. Dwupoziomowa blokada wiersza tablicy (`FOR SHARE` dla wstawień i przeniesień, `FOR NO KEY UPDATE` dla rebalansowania, usunięć i ustawień; ADR-029) godzi równoległość zwykłych zapisów z wyłącznością rebalansowania. Eskalacja do rebalansowania odbywa się przez restart próby, nie przez podniesienie blokady (ADR-030).
- Diagram sekwencji przeniesienia kolumny: `docs/ARCHITECTURE.md`; diagram ER: `docs/DATABASE.md`.

## 3.x Projekt interfejsu tablic i kolumn oraz synchronizacji stanu klienta (M2d)
- Stan widoku trzyma jeden magazyn Zustand (`useBoardStore`): lista tablic, otwarta tablica, kolumny i członkowie. Dane trwałe pochodzą wyłącznie z REST; magazyn jest ich lokalną kopią, aktualizowaną optymistycznie tam, gdzie opóźnienie byłoby zauważalne (zmiana nazwy kolumny, przeniesienie).
- Spóźnione odpowiedzi: każda grupa operacji ma licznik generacji; odpowiedź z poprzedniej generacji (np. po zmianie tablicy lub wylogowaniu) jest odrzucana. Magazyn jest zerowany przy zmianie `sessionEpoch` lub użytkownika.
- Protokół przeniesienia kolumny (ADR-034, ADR-035): jedno przeniesienie naraz, zawsze oba sąsiedztwa `afterId`/`beforeId`, wycofanie częściowe (tylko przeniesiona kolumna), odświeżenie listy po błędach rozjazdu z serwerem. Ponieważ odpowiedź zawiera jedną kolumnę, a rebalansowanie zmienia klucze innych, klient wykrywa ten przypadek, sprawdzając, czy nowy `rank` mieści się ściśle między lokalnymi sąsiadami.
- Uprawnienia w interfejsie: kontrolki są ukrywane na podstawie tej samej macierzy (`hasBoardPermission`, `canManageMember`, `canLeaveBoard`) co w API; ukrycie jest wyłącznie wygodą — autorytatywna kontrola pozostaje po stronie serwera (ADR-025).
- Wybór natywnego `<select>` (ADR-036) i przekierowania `/` → `/boards` (ADR-037).

## 3.x Projekt systemu wizualnego interfejsu (M2e)
- Interfejs oparto na tokenach projektowych zdefiniowanych jako zmienne CSS (jasny i ciemny motyw) i zmapowanych na tokeny biblioteki shadcn/ui, dzięki czemu komponenty biblioteki przejmują styl bez modyfikacji. Dodano dwa poziomy cieni (`--shadow-card`, `--shadow-raised`), zaokrąglenia 4/6/8 px i font Inter dostarczany lokalnie.
- Motyw (jasny/ciemny/systemowy) zapisuje wybór w `localStorage`; skrypt w dokumencie HTML ustawia klasę `dark` przed renderowaniem aplikacji, co eliminuje błysk niewłaściwego motywu (ADR-038).
- Układ responsywny: panel boczny 240 px zwijany, domyślnie zwinięty poniżej 1024 px i prezentowany jako szuflada poniżej 768 px.
- Dostępność (WNF-05): rozdzielenie `--danger` (wypełnienia) i `--destructive` (tekst) z powodu kontrastu poniżej AA (ADR-042); osobny uchwyt przeciągania kolumny, aby nazwa i licznik pozostały czytelne dla czytników ekranu (ADR-040); menu niemodalne z powrotem fokusu (ADR-041).
- Komunikaty operacyjne przedstawiane jako toasty ze stałym identyfikatorem, walidacja formularzy pozostaje przy polach (ADR-039).

## 3.x Projekt modelu kart, rejestru przejść i kluczy (M3a)
- Karta (`Card`) należy do kolumny i tablicy, ma numer kolejny w obrębie tablicy (z licznika `Board.nextCardNumber`), tytuł, opis, priorytet (`LOW`–`URGENT`), opcjonalną osobę (tylko członek tablicy) i termin jako datę kalendarzową bez strefy. Kolejność w kolumnie wyznacza `rank` (LexoRank, jak dla kolumn; ADR-002, ADR-031); klient opisuje pozycję sąsiadami `afterId`/`beforeId`.
- Identyfikator czytelny dla użytkownika: klucz karty `<klucz tablicy>-<numer>` (np. `SL-12`), jak w narzędziach klasy Jira. Klucz tablicy (2–5 liter) jest generowany z nazwy (inicjały słów, transliteracja polskich i cyrylickich liter) i jest unikalny w obrębie właściciela (ADR-046). Zapisywany jest tylko numer, a klucz składany przy odczycie — zmiana klucza tablicy nie wymaga aktualizacji kart.
- Rejestr przejść (`CardTransition`, ADR-020, ADR-047): wiersz przy utworzeniu karty (`fromColumnId = null`) i przy każdej zmianie kolumny, zapisywany w tej samej transakcji co zmiana. Klucze obce kolumn i autora z `SetNull` zachowują historię po usunięciu kolumny lub konta; zmiana kolejności w kolumnie przejścia nie tworzy. Rejestr jest źródłem metryk przepływu (lead time, cycle time, przepustowość, diagram CFD, wiek elementów w toku, prognoza Monte Carlo — WF-FLOW), których nie da się odtworzyć wstecz, dlatego zapis rozpoczyna się od pierwszej karty.
- Limit WIP w trybie ścisłym jest twardym ograniczeniem po stronie serwera (409 `WIP_LIMIT_EXCEEDED`), a nie tylko wskazówką w interfejsie (ADR-044). Kolumna z kartami nie może zostać usunięta (409 `COLUMN_NOT_EMPTY`).
- Model współbieżności (ADR-043–045): stała kolejność blokad wierszy tablica → karta → kolumna; tworzenie karty serializowane blokadą wyłączną (licznik numerów), przeniesienia równoległe pod blokadą współdzieloną; blokada wiersza kolumny przed zliczeniem kart w kolumnie ze ścisłym limitem; spójność osoby przypisanej zapewnia blokada członkostwa i odpinanie kart przy usunięciu członka. Diagram sekwencji przeniesienia karty: `docs/ARCHITECTURE.md`; diagram ER: `docs/DATABASE.md`.

## 3.x Projekt interfejsu kart i synchronizacji stanu klienta (M3b)
- Interfejs kart stosuje ten sam model aktualizacji optymistycznej co kolumny (ADR-034, ADR-035, ADR-050): zmiana widoku następuje natychmiast, a żądanie `PATCH .../move` opisuje pozycję sąsiadami `afterId`/`beforeId`. Po błędzie wycofywana jest wyłącznie przeniesiona karta, więc edycje wykonane w międzyczasie nie giną. Rebalansowanie wykrywane jest w kliencie ścisłym porównaniem `prev.rank < rank < next.rank`; wtedy lista kart jest czytana ponownie.
- Edycja pól karty ma jawny model kolejności: każda zmiana pola dostaje numer w obrębie pary `<cardId>:<pole>`, a odpowiedź lub wycofanie dotyczy tylko pól, dla których jest to najnowsza edycja (ADR-053). Zapobiega to nadpisaniu nowszej wartości starszą odpowiedzią, gdy żądania wracają w innej kolejności. Podobnie lista kart czytana jest ponownie, jeśli mutacja nastąpiła w trakcie jej ładowania (ADR-054).
- Szczegóły karty to zagnieżdżona trasa modalna z adresem do udostępnienia (`/boards/:boardId/cards/:cardKey`; ADR-051); karta spoza store jest pobierana po kluczu.
- Filtr kart działa po stronie klienta; ponieważ przy filtrze sąsiedzi widoczni nie są sąsiadami w danych, przeciąganie jest wtedy wyłączone, a liczniki kolumn i limit WIP pokazują sumę bez filtra (ADR-052).
