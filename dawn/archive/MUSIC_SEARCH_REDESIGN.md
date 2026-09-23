# Music Search Redesign — Design Document

**Created**: September 22, 2026
**Status**: Shipped — data layer (`e902ab8`) + relevance-ranked query core (`828b196`), both 2026-09-22. Round 2 (punctuation-insensitive matching, pagination with totals, `album:`, album inventory) implemented 2026-09-23 — see §2. Remaining phases shelved with triggers (see §6).
**Scope**: Full redesign of music search across all surfaces (mixed local + Plex library).

---

## 1. Problem

Music search was one of the earliest tools and hadn't kept pace. A single flat query
routed through `music_db_search()` → `SQL_SEARCH`, which OR'd one `LIKE` pattern across
`title/artist/album/genre/path`, deduped across sources, and returned rows **alphabetically
ordered, not by relevance**, truncated by a tiny `LIMIT`. Consequences seen live:

1. **Genre was being silently erased for local files.** The DB had genre on 96% of local
   rows (written by the unmerged `dawn_satellite` commit `a4c45a0`), but HEAD's
   `insert_track()` never wrote the genre column, so any file whose mtime changed was
   re-inserted with `genre=NULL`. Active data loss.
2. **No year/date** column, extraction, or result field — decade/era queries impossible.
3. **No relevance ranking.** "Prince" returned "Princeton"/"Princess Leia" (alpha-first,
   window of 5) and never the 57 real Prince tracks; "Queen" surfaced "Queen Of Mars".
4. **Path-bleed.** Matching the file path meant "Ramin Djawadi" returned "DJ Boborobo"
   (folder name) and "1980" matched a plex path fragment.

The architecture was otherwise sound; the problem was missing/erased data + a flat query
model.

---

## 2. What shipped

### Phase 1 — data layer (`e902ab8`)

- **Local decoders** (FLAC/Ogg/MP3) now extract **genre** and **release year** into
  `audio_metadata_t`. FLAC/Ogg read `GENRE`/`DATE` (multi-genre comma-joined);
  MP3 reads ID3 genre with `(NN)`/`(NN)Text`/bare-`NN` normalization via the ID3v1 table,
  and year via a `TDRC`→`TYER`→`TDOR` frame scan (libmpg123's `v2->year` is NULL for
  ID3v2.4). Pure helpers live in `src/audio/audio_metadata_util.c` (unit-tested,
  `test_audio_metadata`).
- **Schema**: `year INTEGER` column + `idx_music_year`; genre column already existed. NULL
  stored for unknown year so `year BETWEEN` excludes untagged tracks.
- **Erasure fix**: the local insert now writes genre + year.
- **Plex** parses `parentYear` into the year column.
- **Backfill**: `PRAGMA user_version` (`MUSIC_META_VERSION`) triggers a one-time forced
  local re-scan (`UPDATE … SET mtime=0 WHERE source=0`) so existing rows pick up genre/year.
  Documented in `UPGRADING.md`.
- `SQL_SELECT_BY_PATH` widened to the shared 8-column projection (also fixed a latent bug
  where `plex:` tracks reported LOCAL source). Shared `populate_result_from_row` reader +
  `MUSIC_ROW_COLS` macro keep every query on one column contract.

### Phase 2a — ranked query core (`828b196`)

- **`music_db_query(music_query_t*, …)`** — the single query core every surface uses. Free
  text is ranked across artist/title/album/genre; fielded `artist`/`title`/`album` filters +
  `genre` + `year_min`/`year_max`. **The file path is no longer matched** (kills the bleed).
- **`src/audio/music_rank.c`** — pure, unit-tested scorer (`test_music_rank`): tiers
  exact > prefix > whole-word > substring, field-weighted artist/title > album > genre.
  Prefix requires a word boundary, so "Prince" does not prefix-match "Princess".
- **Model — breadth by default, precision when fielded**:
  - Free-text search returns **ranked breadth**: best-first, partials kept lower, capped at
    the limit, only true non-matches dropped. The LLM sees options and chooses. (An earlier
    hard relative-score threshold was tried and removed — ranking already solves the original
    crowding, and dropping partials removed the caller's judgment.)
  - A fielded `artist:`/`title:`/`album:` filter is the **precision lever**: candidates must
    match that field at a **word boundary or better**, so `artist:"Prince"` excludes
    "…Princeton Nassoons" entirely. Genre is a broad substring filter; year is numeric.
  - Ranking term falls back text → artist → title → album; genre/year-only queries return
    alpha order.
- **New tool params** `artist` / `title` / `genre` / `year_min` / `year_max`, parsed once via
  `music_query_parse_filters` and assembled via `music_query_from_filters` (shared by the
  voice + WebUI executors).
- **Every surface unified** on the core: voice + WebUI tools, browser search box and
  play-by-query, `dawn-admin` CLI, and the play/resolve picker
  (`music_db_pick_best_match` is now whole-word aware, so "play Prince" matches
  "search Prince"). `music_db_search` retained as a text-only wrapper.
- Candidate window `MUSIC_QUERY_CANDIDATE_WINDOW = 300` (heap, off-lock), C-side rank/cap
  after releasing `g_db_mutex`.

Reviewed across correctness / architecture / security / efficiency lenses at each step;
live-validated with the voice assistant.

### Round 2 — one predicate, pagination, album inventory (2026-09-23)

Triggered by a live discography question ("which Ben Folds albums am I missing?") that got a
confidently wrong answer. Four root causes, all fixed:

1. **Punctuation broke matching.** SQL tokenized the query on spaces
   (`%Rockin%the%Suburbs%`), but the scorer demanded the whole needle contiguously, so
   "Rockin the Suburbs" vs stored "Rockin' the Suburbs" scored 0 and every candidate was
   dropped. Same for AC DC/AC/DC, Guns N Roses, `&`/and, "Dont"/"Don't".
2. **No pagination or totals** on `search`, and ranking only saw a 300-row alphabetical
   window — Ben Folds alone is ~760 rows.
3. **No `album:` field** — the model used `title:` for album names.
4. **No inventory view** — "which albums do I have" meant wading through duplicate-heavy
   track rows (every album exists 2–4×: local FLAC, Amazon MP3, Plex indexed twice).

**Design.**

- **Folded token matcher** (`music_rank.c`, pure). Fold = ASCII lowercase, apostrophes
  (incl. U+2018/2019) removed, `&` → "and", other ASCII and common Unicode punctuation
  (dashes, curly quotes, ellipsis, NBSP) → separator. Tiers on folded text:
  EXACT 1000 > PREFIX 700 > WORD 500 (contiguous phrase at word boundaries) > ALL_WORDS 400
  (every token a whole word, any order) > SUBSTRING 200 > ALL_PRESENT 100. Row score =
  best field-weighted tier; if no single field holds every token but the row does
  ("Ben Folds Rockin" = artist + title) → CROSS tiers 50/25, below any single-field match.
  Filler words (the, a, of, by, and, feat, song, album…) help phrase matching but aren't
  *required* in free text; tokens ≤ 2 chars count only as whole words.
- **The matcher lives in SQL.** It is registered as deterministic SQLite functions
  (`music_score`, `music_match`, `music_album_key`, `music_fold`) on the music DB connection
  only (re-registered each init; never used in schema objects, since the Plex sync connection
  and the CLI lack them). `music_db_query_page()` filters, orders (`music_score DESC`, then
  alpha, then `id` for stable paging) and counts (`COUNT(*) OVER()` in the same statement,
  one snapshot) with that single predicate. The candidate window, its 754 KB malloc, and the
  "SQL matched but scorer dropped it" bug class are gone by construction; totals are exact.
- **Fielded filters are strict**: every word required (filler included, so `artist:"The Band"`
  ≠ Dave Matthews Band), whole-word, punctuation/order-insensitive (quality ≥ ALL_WORDS).
- **Typo tolerance is opt-in and labeled.** `music_query_t.allow_partial`: only when a strict
  pass finds nothing, rows missing one word of a 3+-word query (at least one whole-word hit)
  are returned, flagged `approximate`. Only the LLM-facing search/batch text sets it and says
  "No track matches every word … closest matches"; play/enqueue/resolve/browser/admin stay
  strict, so a near-miss is never *played* silently.
- **Dedup** additionally collapses same-source copies (a library indexed twice) when artist/
  album/title match and genre, year and a known duration (±2 s) agree — so a copy is never
  hidden by one that a genre/year filter would reject. Written as one `idx_music_dedup` range
  seek. Copies that disagree on tags (e.g. a local FLAC tagged 1997 and an MP3 tagged 2005)
  deliberately remain separate. Under a genre/year filter the cross-source rule also requires matching genre/year,
  so an untagged local copy can't hide the tagged Plex copy the filter is looking for.
- **Album inventory**: `music_db_list_albums_by_artist()` groups by an edition-folded album key
  (strips trailing `(Expanded Edition)`, `(EP)`, `[Clean]`, ` - Disc 1`, ` - <artist>` … but
  not content qualifiers like `(Live)`/`(Acoustic Version)`), counts distinct folded titles,
  earliest year, edition and artist counts, shortest display name; oldest first. Grouped by
  key only (not artist), so a compilation credited to many "Ben Folds Presents: X" artists
  lists once — the trade-off is that same-named albums by two matching artists merge (shown
  as "+N other artists").
- **Tool surface** (`src/tools/music_search.c`, shared by both executors):
  every `search` result leads with the total and page count —
  `Found 423 tracks for artist 'Ben Folds' - showing 26-50 (page 2 of 17). Use page:3 for
  more.` — rows show album + year + path; empty/past-the-end results echo the query. Batch
  `items[]` gives per-query totals (capped at 50 queries). New `album:` param; `page` applies to
  `search`. `library query:'albums' artist:'X'` returns the discography view. Tag-derived text
  has control characters flattened so a hostile tag can't forge a result line or header.
- **Split**: query code moved from `music_db.c` to `music_db_query.c` (`music_db_internal.h`
  shares the connection/lock/projection/dedup macro). Legacy path-matching `music_db_search`
  removed; the resolver picker is now `music_rank_pick_best` (whole-word artist match).

---

## 3. Key files

`src/audio/music_db.c` (lifecycle, scan, browse, `music_db_populate_row`),
`src/audio/music_db_query.c` (`music_db_query_page`, `music_db_query`,
`music_db_list_albums_by_artist`, SQL function registration), `include/audio/music_db_internal.h`,
`src/audio/music_rank.{c,h}` (fold, tiers, `music_rank_pick_best`, album key),
`src/audio/audio_metadata_util.c`, `src/audio/{flac,mp3,ogg}_decoder.c`,
`src/audio/plex_{client,db}.c`, `src/tools/music_search.{c,h}` (filters, page/batch/album text),
`src/tools/music_tool.c`, `src/webui/webui_music_tool.c`, `src/webui/webui_music_handlers.c`,
`src/auth/admin_socket_music.c`. Tests: `tests/test_music_rank.c`, `test_music_db.c`,
`test_music_search.c`, `test_audio_metadata.c`.

---

## 4. Non-goals (deliberate)

External metadata enrichment (MusicBrainz/Last.fm); recommendations/taste models;
audio-content mood models (CLAP/Essentia); playback/queue/streaming changes; queue-entry
metadata; a separate `exact`-match mode (fielded + ranking covers it).

---

## 5. Measurements (Jetson, this box, 2026-09-22)

- Local tag coverage: genre ~99%, year ~93% of sampled files.
- Plex genre coverage: 46% at track level, 88% at album level (album-genre fallback is a
  possible future win — see §6).
- Keyword timing on a 6,616-row DB copy: 5-column `LIKE`-OR + dedup ≈ 8–9 ms. FTS5 trigram
  `MATCH` < 1 ms (the basis for the shelved Phase 2b).
- Candidate buffer: `sizeof(music_search_result_t)` ≈ 2,572 B → 300-row window ≈ 754 KB
  transient per query (removed in Round 2).
- Round 2 (6,616 rows, -O2): 2–10 ms per query through the SQL scorer (fielded ≈ 2–6 ms,
  free text ≈ 6–10 ms, album inventory ≈ 6 ms). An efficiency review extrapolated to a
  ~53k-row copy at ~60–120 ms for text queries after its fixes, with the dedup `NOT EXISTS`
  the dominant remaining cost.

---

## 6. Shelved with triggers

The high-value core shipped. Everything below is optimization, polish, or speculative
capability — none addresses a problem hit in practice. Deliberately shelved 2026-09-22; each
names the signal that should un-shelve it. (Pointer lives in `docs/TODO.md`.)

| Item | Trigger to revisit |
|---|---|
| **Phase 2b — FTS5 trigram candidate generation.** Swap the LIKE candidate scan for an FTS5 external-content table (keep the C scorer). Pure speed/token-AND; correctness fully handled by 2a. Cost: sync triggers across all write paths (local scan, Plex bulk upsert, Plex `sync_gen` delete) + a ≥3-char trigram constraint needing a 2-char LIKE fallback. | Library crosses **~50k rows/source** AND search latency shows in telemetry. |
| **Phase 4 — semantic mood search.** Per-track embeddings (reuse `embedding_engine`) + vector store + hybrid merge for "play something upbeat/chill". Text embeddings capture lexical/genre priors, not acoustic mood; marginal lift over "genre filter + the LLM's own world knowledge" is unproven. | Real, repeated mood requests that genre/year + LLM reasoning **demonstrably can't** satisfy — then gate the build on a ~20-query eval showing lift over that baseline. |
| **Phase 3 — WebUI genre/year filter UI + genre browse.** Filter dropdowns and rendering genre/year in browser result rows. Polish for the browser (secondary to voice); core search already works there. | A decision to make the browser a first-class music-browsing surface. |
| **Plex album-genre fallback.** One `type=9` album fetch mapping `parentRatingKey`→album genre when a track's genre is empty (raises Plex genre coverage 46%→~88%). | Plex genre gaps become a felt problem in use. |
| **Write-time dedup flag.** Replace the per-row `NOT EXISTS` (now the top query cost, also paid by stats/browse) with a `dup_hidden` column recomputed by one set-based UPDATE at the end of the local scan and Plex sync (builtins only, so both connections can run it). | Library passes **~20k rows** or search/browse latency shows. |
| **Read-only search connection off `g_music_db_mutex`.** Searches hold the global lock for the full scan (the browser search box runs on the lws thread). A WAL reader connection with the functions registered would leave the mutex to writers. | Library passes **~20k rows**. |
| **Single-pass typo fallback.** A miss with `allow_partial` scans twice (strict, then partial); one pass at the PARTIAL floor with `SUM(score > PARTIAL) OVER()` as the strict count would halve miss latency. Also: score is evaluated in both WHERE and ORDER BY, and `COUNT(*) OVER()` materializes the match set. | Miss latency shows in telemetry, or alongside the ~20k-row items. |
