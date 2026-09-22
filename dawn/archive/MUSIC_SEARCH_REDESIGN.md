# Music Search Redesign — Design Document

**Created**: September 22, 2026
**Status**: Shipped — data layer (`e902ab8`) + relevance-ranked query core (`828b196`), both 2026-09-22. Remaining phases shelved with triggers (see §6).
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

---

## 3. Key files

`src/audio/music_db.c` (`music_db_query`, `build_like_pattern`, `populate_result_from_row`,
`music_db_pick_best_match`), `src/audio/music_rank.{c,h}`, `src/audio/audio_metadata_util.c`,
`src/audio/{flac,mp3,ogg}_decoder.c`, `src/audio/plex_{client,db}.c`,
`src/tools/music_tool.{c,h}` (+ `music_query_parse_filters`/`music_query_from_filters`,
`music_search_filters_t`), `src/webui/webui_music_tool.c`, `src/webui/webui_music_handlers.c`,
`src/auth/admin_socket_music.c`. Tests: `tests/test_music_rank.c`, `test_audio_metadata.c`,
`test_music_db.c`.

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
  transient per query.

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
| **Candidate-buffer reuse across a batch resolve.** The ~754 KB window is malloc/freed per `music_db_query`; a batch resolve does it per item (bounded by `MAX_PLAYLIST_LENGTH`). Bounded and safe today. | Batch-resolve latency surfaces in a profile. |
| **Split `music_db_query.c` out of `music_db.c`.** `music_db.c` is at the 1,500-line soft flag. | Next substantial edit to `music_db.c`. |
| **Surface an `album:` tool param + echo the query in empty-result messages.** Minor symmetry/UX. | Next touch of the tool surface. |
