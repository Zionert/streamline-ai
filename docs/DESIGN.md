# StreamLine — system wizualny (UI/UX)

Dokument obowiązuje dla wszystkich zadań dotyczących interfejsu. Styl: czysty, „korporacyjny”
interfejs narzędzi klasy Jira — biała przestrzeń, szare tło kolumn, białe karty, niebieski akcent.

> Inspiracja wizualna: układ typowy dla narzędzi klasy Jira (m.in. publiczne projekty typu
> „jira clone”). **Nie kopiujemy kodu, ikon, logotypów ani fontów** z innych projektów —
> implementacja jest własna, na naszym stosie (Tailwind CSS + shadcn/ui + lucide-react).

## 0. Język interfejsu
Wszystkie teksty widoczne dla użytkownika (etykiety, przyciski, komunikaty, błędy, puste stany)
są **w języku angielskim**. Przykłady etykiet w tym dokumencie podano w brzmieniu docelowym.
Daty i liczby formatowane przez `Intl` z lokalizacją `en-GB` (np. „12 Nov 2026”).

---

## 1. Układ aplikacji (app shell)

```
┌──────────────────────────────────────────────────────────────────────┐
│ ◆ StreamLine                                   [?] [☀/☾]  (NB) ▾     │  ← pasek górny 56 px
├───────────────┬──────────────────────────────────────────────────────┤
│ [■] Name      │ Boards / Board name                                  │  ← okruszki
│     descr… ‹  │ Board                                                │  ← tytuł H1
│               │ [Filter cards 🔍] (A)(B)(C)(+2)  [⇅ Sort]             │  ← pasek narzędzi
│ ▥ Board       │ Press Shift + N to create a card                     │  ← podpowiedź
│ 📈 Analytics  │ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│ 👥 Members    │ │TO DO (2)+│ │IN PROG(1)│ │REVIEW   +│ │DONE     +│   │
│ ⚙ Settings    │ │┌────────┐│ │┌────────┐│ │          │ │   ⊘      │   │
│               │ ││ karta  ││ ││ karta  ││ │          │ │ No cards │   │
│               │ │└────────┘│ │└────────┘│ │          │ │          │   │
│ 240 px, zwijany│└──────────┘ └──────────┘ └──────────┘ └──────────┘   │
└───────────────┴──────────────────────────────────────────────────────┘
```

### Pasek górny (`AppHeader`)
- Wysokość 56 px, tło `--surface`, dolna krawędź `--border`.
- Lewo: logo (własna ikona — romb/strzałka w kolorze `--primary`) + „StreamLine” (font 600, 18 px), link do `/boards`.
- Prawo: przycisk pomocy (skróty klawiszowe), przełącznik motywu (jasny / ciemny / systemowy), awatar użytkownika z menu (imię, e-mail, „Log out”).

### Panel boczny (`BoardSidebar`) — tylko na stronach tablicy
- Szerokość 240 px, tło `--sidebar`, prawa krawędź `--border`; zwijany przyciskiem `‹` na krawędzi do 0 px (stan zapamiętany w `localStorage`).
- Nagłówek: kwadratowa ikona tablicy (kolor z palety wg identyfikatora tablicy) + nazwa (600, 16 px) + opis skrócony do 2 linii.
- Nawigacja: **Board**, **Analytics** (od M6; wcześniej nieaktywna z etykietą „Soon”), **Members**, **Settings** (tylko z uprawnieniem). Pozycja aktywna: tło `--primary-subtle`, tekst i ikona `--primary`.
- Poniżej 1024 px panel domyślnie zwinięty; poniżej 768 px — wysuwany (drawer).

### Lista tablic (`/boards`)
- Pasek górny bez panelu bocznego; okruszki „Boards”, tytuł „Your boards”, przycisk „Create board” (primary).
- Siatka kart tablic (min. 260 px): ikona tablicy, nazwa, rola użytkownika (badge), liczba członków (awatary), menu „⋯” („Rename”, „Delete” — wg uprawnień).
- Pusty stan: ilustracja-ikona + „You don't have any boards yet” + przycisk.

### Strona tablicy (`/boards/:boardId`)
- Okruszki: „Boards / {name}”; tytuł H1 „Board”.
- Pasek narzędzi: pole „Filter cards” (szer. 200 px, ikona lupy), stos awatarów członków (max 4 + „+N”; kliknięcie filtruje karty osoby, aktywny awatar z obramowaniem `--primary`), przycisk „Sort” (styl subtle). Filtry działają od M3.
- Podpowiedź skrótu nad kolumnami (tekst `--text-subtle`, klawisze w ramce `kbd`).
- Kolumny w poziomym przewijanym rzędzie z odstępem 12 px.

---

## 2. Kolumna (`BoardColumn`)
- Szerokość 280 px, tło `--column`, zaokrąglenie 6 px, padding 8 px, wysokość do dołu ekranu (przewijanie wewnętrzne kart).
- Nagłówek: nazwa **wielkimi literami** (12 px, 600, `--text-subtle`, letter-spacing 0.04em) + licznik „(n)”; przy limicie WIP „(n / limit)”.
  - Przekroczony limit WIP: licznik i górna krawędź kolumny w `--danger`, tło kolumny `--danger-subtle`.
- Kategoria kolumny jako mała kropka/ikona przy nazwie (BACKLOG szary, TODO niebieski, IN_PROGRESS żółty, DONE zielony), nie jako duży badge.
- Przycisk „+” po prawej w nagłówku — szybkie dodanie karty (M3); menu „⋯” kolumny: „Rename”, „Settings”, „Delete” (wg uprawnień).
- Przeciąganie kolumny za **dedykowany uchwyt** (`GripVertical`, `role=button` nadawany przez bibliotekę dnd); nagłówek z nazwą i licznikiem nie jest uchwytem, bo inaczej czytniki ekranu pomijałyby nazwę i licznik (ADR-040). W trakcie — cień `--shadow-raised` i lekkie obrócenie (1–2°).
- Pusty stan: ikona `Ban`/`CircleSlash` (lucide) + „NO CARDS” (12 px, wielkie litery, `--text-subtle`), wyśrodkowane.
- „+ Add column” jako ostatnia, przerywana „kolumna-widmo”.

## 3. Karta (`CardTile`, M3)
```
┌──────────────────────────────┐
│ Tytuł karty w maksymalnie    │  14 px, 500, 2 linie (line-clamp)
│ dwóch liniach…               │
│                              │
│ ☑ SL-12   🏷 etykiety  ⚑  (NB)│  typ/klucz, etykiety, priorytet, awatar
└──────────────────────────────┘
```
- Tło `--surface`, zaokrąglenie 4 px, padding 12 px, cień `--shadow-card`, odstęp między kartami 8 px.
- Hover: tło `--surface-hover`; fokus klawiatury: obramowanie 2 px `--primary`.
- Dół karty: ikona typu + **klucz karty** `{KLUCZ_TABLICY}-{numer}` (12 px, 600, `--text-subtle`), etykiety (małe kolorowe pigułki), **flaga priorytetu** (lucide `Flag`, wypełniona), awatar osoby przypisanej (24 px).
- Priorytety: `LOW` — zielony, `MEDIUM` — żółty/pomarańczowy, `HIGH` — czerwony, `URGENT` — ciemnoczerwony + ikona `ChevronsUp`.
- Karta „starzejąca się” (M6, WF-FLOW-04): mała ikona zegara w `--warning` z podpowiedzią „In progress for N days”.
- Przeciąganie: cień `--shadow-raised`, obrót 2°, miejsce docelowe — przerywany prostokąt.

> Wymaganie dla M3: tablica ma krótki **klucz** (np. „SL”, 2–5 wielkich liter, generowany z nazwy, edytowalny),
> a karta — **numer kolejny w obrębie tablicy**; razem tworzą czytelny identyfikator jak w Jirze.

## 4. Szczegóły karty (M3/M5)
- Modal na trasie `/boards/:boardId/cards/:cardKey` (adres do udostępnienia), szerokość 900 px, animacja pojawienia.
- Układ dwukolumnowy: lewa (65%) — tytuł edytowalny w miejscu, opis, komentarze; prawa (35%) — status (kolumna), osoba przypisana, priorytet, etykiety, termin, daty utworzenia/aktualizacji.
- Pasek akcji u góry: klucz karty, przyciski „Copy link”, „Delete”, „Close” (Esc).
- Asystent AI (M7) jako przycisk „✨ Assistant” w pasku akcji, wyniki w panelu z akceptacją.

## 5. Strony logowania i rejestracji
- Tło `--canvas`; wyśrodkowana karta 400 px na `--surface` z cieniem `--shadow-raised`.
- Nad kartą logo i nazwa „StreamLine”; tytuł „Log in to StreamLine”.
- Pola o wysokości 40 px, przycisk primary na pełną szerokość, link do drugiego formularza pod spodem.

---

## 6. Tokeny kolorów

Definiowane jako zmienne CSS w `apps/web/src/index.css` i mapowane na tokeny shadcn
(`--primary`, `--background`, `--card`, `--border` itd.), aby komponenty shadcn przejęły styl.

| Token | Jasny | Ciemny | Zastosowanie |
|---|---|---|---|
| `--canvas` | `#FFFFFF` | `#1D2125` | tło strony |
| `--surface` | `#FFFFFF` | `#22272B` | karty, modale, pasek górny |
| `--surface-hover` | `#F7F8F9` | `#2C333A` | hover kart |
| `--sidebar` | `#F7F8F9` | `#1D2125` | panel boczny |
| `--column` | `#F1F2F4` | `#161A1D` | tło kolumn |
| `--text` | `#172B4D` | `#B6C2CF` | tekst podstawowy |
| `--text-subtle` | `#626F86` | `#8C9BAB` | tekst pomocniczy, nagłówki kolumn |
| `--border` | `#DCDFE4` | `#38414A` | krawędzie |
| `--primary` | `#0C66E4` | `#579DFF` | akcent, przyciski, linki, aktywne elementy |
| `--primary-hover` | `#0055CC` | `#85B8FF` | hover akcentu |
| `--primary-subtle` | `#E9F2FF` | `#1C2B41` | tło aktywnej pozycji menu, przycisku „subtle” |
| `--success` | `#22A06B` | `#4BCE97` | DONE, priorytet LOW |
| `--warning` | `#E2B203` | `#F5CD47` | IN_PROGRESS, priorytet MEDIUM, starzejąca się praca |
| `--danger` | `#E34935` | `#F87168` | przekroczony WIP, priorytet HIGH, akcje destrukcyjne |
| `--danger-subtle` | `#FFECEB` | `#42221F` | tło kolumny z przekroczonym WIP |

Cienie: `--shadow-card: 0 1px 1px rgb(9 30 66 / 25%), 0 0 1px rgb(9 30 66 / 31%)`;
`--shadow-raised: 0 8px 12px rgb(9 30 66 / 15%), 0 0 1px rgb(9 30 66 / 31%)` (w motywie ciemnym czarne o wyższym kryciu).

**Tekst w kolorze błędu:** `--danger` (`#E34935`) nie spełnia WCAG AA na białym tle, dlatego służy tylko do wypełnień i obramowań (górna krawędź kolumny). Czerwony **tekst** (licznik przekroczonego WIP, komunikaty błędów) używa `--destructive`: `#AE2E24` w motywie jasnym, `#F87168` w ciemnym (ADR-042). W motywie ciemnym przyciski primary i destructive mają ciemny tekst. Kropka kategorii: TODO w `--primary`, BACKLOG w `--text-subtle`.

Kontrast tekstu do tła — minimum WCAG AA (WNF-05).

## 7. Typografia, ikony, odstępy
- Font: **Inter** (licencja SIL OFL) z pakietu `@fontsource-variable/inter` (lokalnie, bez CDN); fallback `system-ui, sans-serif`.
- Skala: 12 / 14 (bazowa) / 16 / 20 / 24 / 29 px; H1 strony 24 px 600; tekst `--text`.
- Ikony: **lucide-react**, 16 px w treści, 20 px w nawigacji, `stroke-width` 2.
- Siatka odstępów: wielokrotności 4 px; zaokrąglenia: 4 px (karty, pola), 6 px (kolumny), 8 px (modale).
- Awatary: koło, inicjały na kolorze z palety 8 barw wybieranym deterministycznie z `userId`; stos awatarów z nakładaniem −8 px i obwódką 2 px w kolorze tła.

## 8. Motywy
- Jasny (domyślny), ciemny, systemowy; przełącznik w pasku górnym; wybór w `localStorage`; klasa `dark` na `<html>` (zgodnie z `@custom-variant dark`).
- Brak migotania przy starcie: skrypt w `index.html` ustawia klasę przed renderem Reacta.

## 9. Interakcje i dostępność
- Skróty: `Shift + N` — nowa karta (M3), `/` — fokus na filtrze, `Esc` — zamknięcie modala, `?` — lista skrótów.
- Aktualizacje optymistyczne bez blokowania UI; błąd → toast (shadcn `sonner`) i wycofanie zmiany.
- Komunikaty toast w prawym dolnym rogu; potwierdzenia destrukcyjne w `AlertDialog`.
- Stan ładowania: szkielety (skeleton) kolumn i kart zamiast spinnerów na całą stronę.
- Wszystkie akcje dostępne z klawiatury; widoczny fokus; etykiety `aria-label` dla przycisków-ikon.
