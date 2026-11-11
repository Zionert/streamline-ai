# Baza danych

Stan: M2b — modele `User`, `RefreshToken`, `Board`, `BoardMember`, enum `BoardRole`; dwie migracje. Wstępny model docelowy: `docs/TECH_STACK.md`, sekcja 5.

## Diagram ER
```mermaid
erDiagram
  User ||--o{ RefreshToken : "ma"
  RefreshToken |o--o| RefreshToken : "replacedBy (rotacja)"
  User ||--o{ BoardMember : "jest członkiem"
  Board ||--o{ BoardMember : "ma"
  User {
    string id PK
    string email UK
    string passwordHash
    string name
    datetime createdAt
    datetime updatedAt
  }
  RefreshToken {
    string id PK
    string userId FK
    string tokenHash UK
    datetime expiresAt
    datetime revokedAt
    string replacedById FK
    datetime createdAt
  }
  Board {
    string id PK
    string name
    datetime createdAt
    datetime updatedAt
  }
  BoardMember {
    string id PK
    string boardId FK
    string userId FK
    BoardRole role
    datetime createdAt
    datetime updatedAt
  }
```

## Konfiguracja
- PostgreSQL 18 uruchamiany przez `docker-compose.yml` (`postgres:18-alpine`); dane dostępowe z root `.env` (`POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`, `POSTGRES_PORT`).
- Prisma 7.10.1: adres bazy w `apps/api/prisma.config.ts` (zmienna `DATABASE_URL` z `apps/api/.env`).
- Generator `prisma-client`, wyjście `apps/api/src/generated/prisma` (poza repozytorium; generowany w `postinstall` pakietu API).
- Połączenie przez `@prisma/adapter-pg`; klient tworzony w `apps/api/src/repositories/db.ts` (`createDbClient`), połączenie leniwe; w `server.ts` trafia do `createApp({ config, logger, db })`.
- Tylko repozytoria korzystają z Prisma.
- Testy integracyjne API używają osobnej bazy `streamline_test` (ADR-015). Adres: `DATABASE_URL_TEST` albo `DATABASE_URL` z dopiskiem `_test` do nazwy bazy; baza jest tworzona i migrowana automatycznie (`prisma migrate deploy` w `globalSetup`), a tabele są czyszczone (`TRUNCATE`) przed każdym testem. Nazwa bazy testowej musi kończyć się na `_test`. CI przekazuje `DATABASE_URL_TEST` i używa usługi PostgreSQL.

## Modele

### User
| Pole | Typ | Uwagi |
|---|---|---|
| `id` | String | PK, `cuid()` |
| `email` | String | unikalny; zapisywany znormalizowany (przycięty, małe litery) |
| `passwordHash` | String | hash bcrypt (bcryptjs) |
| `name` | String | |
| `createdAt` | DateTime | domyślnie `now()` |
| `updatedAt` | DateTime | `@updatedAt` |

Relacje: `refreshTokens` (1:N), `memberships` (1:N, `BoardMember`). Indeks: unikalny `User_email_key`.

### RefreshToken
| Pole | Typ | Uwagi |
|---|---|---|
| `id` | String | PK, `cuid()` |
| `userId` | String | FK → `User.id`, `onDelete: Cascade` |
| `tokenHash` | String | unikalny; SHA-256 (hex) nieprzezroczystego tokena — sam token nie jest przechowywany |
| `expiresAt` | DateTime | |
| `revokedAt` | DateTime? | ustawiane przy rotacji, wylogowaniu lub masowym unieważnieniu |
| `replacedById` | String? | unikalny FK → `RefreshToken.id` (`onDelete: SetNull`); token wydany w zamian przy rotacji |
| `createdAt` | DateTime | domyślnie `now()` |

Indeksy: unikalne `tokenHash` i `replacedById`, zwykły `userId`. Para `replacedById` ustawione i `revokedAt` niepuste odróżnia token wymieniony przy rotacji od unieważnionego wylogowaniem — to podstawa wykrywania ponownego użycia (ADR-013).

### BoardRole (enum)
`OWNER`, `ADMIN`, `MEMBER`, `VIEWER` (kolejność deklaracji decyduje o sortowaniu członków). Uprawnienia ról: `packages/shared/src/permissions.ts` (macierz w `docs/API.md`).

### Board
| Pole | Typ | Uwagi |
|---|---|---|
| `id` | String | PK, `cuid()` |
| `name` | String | 1–100 znaków (walidacja Zod) |
| `createdAt` | DateTime | domyślnie `now()` |
| `updatedAt` | DateTime | `@updatedAt`; zmienia się tylko przy zmianie nazwy |

Relacja: `members` (1:N).

### BoardMember
| Pole | Typ | Uwagi |
|---|---|---|
| `id` | String | PK, `cuid()` |
| `boardId` | String | FK → `Board.id`, `onDelete: Cascade` |
| `userId` | String | FK → `User.id`, `onDelete: Cascade` |
| `role` | BoardRole | |
| `createdAt` | DateTime | domyślnie `now()`; data dołączenia |
| `updatedAt` | DateTime | `@updatedAt` |

Indeksy: unikalny `BoardMember_boardId_userId_key` (`boardId`, `userId`), zwykły `BoardMember_userId_idx`, oraz **częściowy unikalny** `BoardMember_one_owner_per_board` na `boardId` z warunkiem `WHERE "role" = 'OWNER'` — najwyżej jeden właściciel na tablicę. Indeks częściowy jest dopisany ręcznie w SQL migracji (Prisma go nie wyraża); co najmniej jednego właściciela zapewnia serwis (tablicę tworzy z właścicielem, właściciel nie może odejść ani zostać zmieniony). Patrz ADR-027.

Kaskady: usunięcie tablicy usuwa jej członkostwa; usunięcie użytkownika usuwa jego członkostwa (przy dodaniu usuwania kont trzeba zadbać o tablice bez właściciela).

## Migracje
| Migracja | Zawartość |
|---|---|
| `20261111010926_auth_users_refresh_tokens` | tabele `User` i `RefreshToken`, indeksy, klucze obce |
| `20261111143833_boards_members` | enum `BoardRole`, tabele `Board` i `BoardMember`, indeksy (w tym częściowy `BoardMember_one_owner_per_board`), klucze obce z kaskadą |

Polecenia: `pnpm --filter @streamline/api db:migrate` (tworzy i stosuje migrację w środowisku deweloperskim), `prisma migrate deploy` (stosuje istniejące migracje; używane przez testy).

## Znane braki
- Wygasłe i unieważnione wiersze `RefreshToken` nie są usuwane (brak zadania czyszczącego).
