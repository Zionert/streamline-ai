# Review: M2a — LexoRank-style ordering keys (packages/shared)

## Reviewer report — round 1
**Verdict:** APPROVE
**Checks:** lint PASS, typecheck PASS (all 3 packages), test PASS for shared (226) and web (77); api NOT RUN: Docker daemon is down, so Postgres on localhost:5433 is unreachable (P1001 in globalSetup). This is an environment problem, not caused by this change: api code is untouched and api typecheck passes against the new shared exports. prettier --check PASS. `pnpm --filter @streamline/shared bench` runs in about 4.3 s and prints valid Markdown.

### BLOCKER
- none

### MAJOR
- none

### MINOR
- `packages/shared/src/rank/keys.ts:69-76` (rebalance contract trap, for M2b to know about): `rankBetween` can return a 33+ char key while `rankSchema` caps input at 32. Nothing is wrong today, but it puts two requirements on callers. (a) The server must compute the new key itself, check `key.length > RANK_MAX_LENGTH`, and in the **same transaction** replace the whole list with `rebalanceRanks(n + 1)` before writing anything. It must never persist the long key first. A `VARCHAR(32)` column would make this a hard failure. (b) If the client ever sends a precomputed `rank` (for example from optimistic UI), the request fails with 400 exactly at the threshold. → Expand the `rankSchema` / `RANK_MAX_LENGTH` JSDoc with one sentence: "Only server-computed keys are persisted; when `rankBetween` returns a key longer than RANK_MAX_LENGTH, rebalance the list in the same transaction instead of storing it." The M2b API should accept neighbour IDs or a target index, not raw ranks.
- `packages/shared/src/rank/lexorank.ts:172` (contract note for M2b): `rankBetween` is deterministic. Two concurrent inserts into the same gap produce the **same** key, and after that `rankBetween(k, k)` throws `INVALID_RANK_ORDER`, so nothing can be inserted between the twins. This is correct for this module, but the M2b repository needs a unique `(columnId, rank)` / `(boardId, rank)` constraint with retry, or row locking. The service also needs to map `RankError` to an HTTP error (400 or 409), because stale neighbours give `low >= high`. Add a JSDoc line documenting the determinism.
- `packages/shared/bench/rank.bench.ts:126-131,164-177` (benchmark methodology): the µs/op columns time individual sub-microsecond operations with `performance.now()` and add them up. At 0.06-0.5 µs per op, timer overhead and quantisation are a large share of the result. Also, the integer timing covers only an in-memory loop, not database writes. → For the thesis, base the argument on **rows/op**, which is exact and deterministic, and state in the header text that the µs columns cover key computation only, not database cost. Optionally switch to `process.hrtime.bigint()` or time a whole batch.
- `packages/shared/bench/rank.bench.ts:76-78,123,167` (benchmark fairness): in the `random` scenario `from === to` can happen. The integer strategy then writes 0 rows while ranks still count 1 write and compute a key. The bias works against ranks, so the conclusions still hold, but either skip no-op moves for both strategies or mention it in the header. A real API should treat a no-op move as a no-op anyway.
- `packages/shared/bench/rank.bench.ts:70-72` (benchmark wording): the `middle` scenario always targets the same index, so it is effectively the "right after a fixed neighbour" worst case: 5 rebalances, and 501 rows/op at n=100k. The data is fine (still about 50x fewer writes than integers), but the title "Move a random card to the middle" undersells it. Rename it, for example "... to the same middle position", so the thesis reader understands why it rebalances.
- `packages/shared/bench/rank.bench.ts:36-46`: duplicates mulberry32 from `test-utils/prng.ts`. This is understandable, because plain Node type stripping cannot import `.js`-suffixed TS paths without extra tsconfig flags. Add a one-line comment saying why it is duplicated so it does not look like dead duplication.
- Thesis caveat, not a code issue: the baseline is dense integer re-indexing only, as the brief required. A thesis reviewer may ask about gapped integers or float positions. A sentence in section 5.3 explaining why those were rejected (gaps also run out and need renumbering; floats run out of precision after about 50 halvings) would pre-empt that.
- `packages/shared/src/rank/collation.test.ts`: byte order is checked with `Buffer.compare`, which is the right proxy at this layer. M2b should add one integration test with real Postgres (`ORDER BY rank COLLATE "C"`) on mixed-case keys to close the loop.

### Correctness verification (requested focus)
I traced `shortestBetween` by hand and checked the invariants:
- Termination: once `high` stops constraining (highActive=false), the only failing case is lowDigit = 'z', and low is finite, so a padded 0 eventually gives room. While highActive, `lowDigit > highDigit` and "prefix equals all of high" are both impossible given `low < high` and canonical keys.
- Canonical form plus a byte-ordered alphabet means string order equals numeric order, so the produced key is strictly inside the interval and never ends in '0'.
- Edge cases confirmed: `a` is a prefix of `b` (`a`/`a1` gives `a0V`, `V`/`V1` gives `V0V`); `b` starts with zeros (`rankBefore('001')` gives `000z`, `rankBefore('1')` gives `0z`); `a` ends at 'z' (`z` gives `z1`, `ay`/`b` gives `az`); length-32 adjacent keys give a 33-char key (correct; this is when callers must rebalance).
- `rebalanceRanks`: step = floor(62^L/(n+1)) >= 1 because 62^L > n, the remainder accumulation is exact, and the last value is < 62^L. Keys are distinct, ascending, canonical, and leave room at both ends. Safe-integer limits only matter above about 2e14 elements, which is irrelevant.
- The numbers in the `RANK_MAX_LENGTH` comment (~160 adversarial inserts, ~1 900 appends) match the benchmark output (148-187 and 1 771-1 869).

### Done well
- An exhaustive oracle (`shortestPossibleLength` with BigInt) proves the **shortest key** property on 5 000 random bound pairs and on all single-char pairs. These are strong behavioural tests, not smoke tests.
- `rebalanceRanks` is checked against the exact BigInt formula, plus minimal-length boundaries (61/62, 3843/3844).
- The canonical "no trailing 0" invariant keeps the code simple and guarantees room before any key. It is well explained in JSDoc.
- Rebalance writes are included in the benchmark's rows/op (amortised). Operations are seeded and deterministic, with warm-up and the median of 5 runs. The "inserts until rebalance" table is a strong thesis artefact.
- Strict TS, no `any`, no Prisma/API/web changes, no third-party ranking packages. Privacy check is clean: no mentions of development tools, `docs/` or ADRs. `test-utils`/`bench` are kept out of `dist`.
- `rankSequenceBetween` is a small addition beyond the brief. It is tested and useful for bulk insert/seed, so it is acceptable scope.
