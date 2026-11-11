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