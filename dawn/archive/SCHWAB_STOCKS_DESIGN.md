# Charles Schwab Stocks Tool — Design Document

**Created**: September 2026
**Implemented**: September 2026 (incremental, four rounds)
**Status**: Complete — read-only. Trading is a separate, deferred, dangerous-gated tool.
**CMake option**: `DAWN_ENABLE_SCHWAB_TOOL` (OFF by default; ON in the default/full/debug presets)
**Latest commit**: `6e4b620` (transactions round)

---

## Overview

A read-only market-data and brokerage tool for DAWN, backed by the Charles Schwab
developer API. One LLM tool, `stocks`, exposes six actions:

| Action | What it answers | Schwab endpoint |
|--------|-----------------|-----------------|
| `quote` | Live prices for one or more symbols | `GET /marketdata/v1/quotes` |
| `portfolio` | Holdings + balances + unrealized P/L across all linked accounts | `GET /trader/v1/accounts?fields=positions` |
| `accounts` | Balances only (no per-position detail) | same, formatted without positions |
| `history` | Price trend + return/volatility/drawdown/SMAs for one symbol | `GET /marketdata/v1/pricehistory` |
| `fundamentals` | P/E, EPS, market cap, dividend yield, 52-week range, beta | `GET /marketdata/v1/instruments?projection=fundamental` |
| `transactions` | Recent account activity (trades, dividends, deposits/withdrawals, fees) | `GET /trader/v1/accounts/{hash}/transactions` |

Everything is read-only — the stored OAuth token is never used to place a trade.
It's built to voice-first constraints (compact, LLM-facing result strings) and to
DAWN's tool conventions (metadata-driven registration, three-layer split).

The tool was built in four incremental rounds, each following the same
plan → master-plan-review → implement → correctness+security review → live-verify
loop: (1) quotes + portfolio, (2) price history + fundamentals, (3) custom date
ranges for history, (4) transactions + portfolio unrealized P/L. This document is
the consolidated record, with the transactions round covered in the most depth
because it is the only one that touches an **undocumented** Schwab endpoint.

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Architecture | Three layers (`schwab_tool` → `schwab_service` → `schwab_client`) + pure helpers | Mirrors the email/calendar tools; the service owns orchestration + LLM-facing formatting, the client owns the raw authenticated JSON GET |
| Auth | Shared `oauth_client.c` (OAuth 2.0 auth-code + PKCE), tokens encrypted via `crypto_store` | Same machinery as Google/CalDAV; no vendor-specific token code |
| Enrollment | One-time-per-week CLI step (`dawn-admin schwab auth`), not the WebUI callback | Schwab pins the OAuth redirect to a loopback HTTPS URL registered in the developer app; DAWN's WebUI callback can't be used |
| Account numbers | Masked to last 4 everywhere they egress (`…1234`) | Never surface a full brokerage account number to the LLM, logs, or TTS |
| Pure analytics | `schwab_stats.c` (price math) and `schwab_txn.c` (transaction classify) are `math`/`json-c`-only, unit-tested | Isolates the parts most likely to be wrong from the network/OAuth layer; each has a Unity test with a fixture |
| Caching | None — live queries | Quotes/positions/transactions change constantly; a one-shot voice query doesn't benefit |
| Read-only | No trading in any round | Trading is irreversible and financial; it will be a separate tool with a dangerous-action gate |
| Transaction category filter | User-language ENUM (`trades`/`dividends`/`deposits`/`withdrawals`/`fees`/`all`) mapped server-side to Schwab's raw type set | The LLM never sees Schwab's 15 raw enum strings; the service whitelists (ENUM params are not validated at dispatch) |
| Return metrics | Deferred, not attempted | Schwab serves only a ~1-year transaction window and no historical valuations, so lifetime performance is infeasible; the classifier tags external-vs-internal cash flows now so a future round can compute it |

## Architecture

```
┌─────────────────┐   metadata + dispatch only
│  schwab_tool.c  │   tool_metadata_t "stocks", 6 actions, param extraction
└───────┬─────────┘
        │ action + packed value ("::field::value")
┌───────▼─────────┐   orchestration + LLM-facing formatting
│ schwab_service.c│   bearer acquisition, retries, error mapping, strbuf output,
│                 │   account masking, date validation, per-account fan-out
└───┬─────────┬───┘
    │         │ pure, unit-tested
    │   ┌─────▼───────────┐   ┌──────────────────┐
    │   │ schwab_stats.c  │   │  schwab_txn.c    │
    │   │ price analytics │   │ txn classify +   │
    │   │ (math only)     │   │ aggregate (json) │
    │   └─────────────────┘   └──────────────────┘
┌───▼─────────────┐   thin authenticated JSON GET over libcurl
│ schwab_client.c │   Authorization: Bearer, 4 MB cap, TLS pinned, no redirects,
│                 │   response body scrubbed from heap (sodium_memzero)
└─────────────────┘
```

**Return codes** (`schwab_rc_t`): `OK`, `ERROR`, `NOT_LINKED`, `AUTH`, `HTTP`,
`RATE_LIMITED`, `TOO_LARGE`. The client maps HTTP 401 → `AUTH` (drives one forced
refresh + retry), 429 → `RATE_LIMITED`, a 200 whose body exceeds the 4 MB cap →
`TOO_LARGE` (distinct so the user is told to narrow the window, not that the
network failed), and everything else non-200 → `HTTP`. The service's
`schwab_req_err` additionally turns a 400/404 into a "check the symbol/dates"
message.

**Two host bases**: `SCHWAB_MARKETDATA_BASE` (`/marketdata/v1`) for quotes,
history, fundamentals; `SCHWAB_TRADER_BASE` (`/trader/v1`) for accounts and
transactions.

**Enrollment / token lifecycle**: `dawn-admin schwab auth` prints an authorize
URL; the operator approves in a browser and pastes the redirected URL back. The
access token lasts ~30 min (auto-refreshed); the **refresh token has a fixed
7-day lifetime and cannot be extended**, so re-linking is a weekly step. See
`docs/SCHWAB_SETUP.md` in the dawn repo for the operator walkthrough.

## Price history + analytics (`schwab_stats.c`)

`schwab_history_stats(close[], high[], low[], n, base, freq, &out)` is pure math:
period % change (vs the first candle's close — Schwab's `previousClose` is not
reliably the pre-window close), period high/low, annualized log-return volatility
(Welford, √252/√52/√12 by interval), max drawdown, and 20/50-bar SMAs. The
`history` action supports both a preset range (`1mo…5y/ytd`, with illegal
range+interval auto-corrected) and an explicit `start`/`end` date window (dates
are **epoch-milliseconds** on the pricehistory endpoint), trimmed client-side so
both boundary days are inclusive. `data=series` emits compact strided arrays for
charting via `render_visual`.

## Transactions (`schwab_txn.c` + the two-call flow)

The transactions endpoint is the first to require Schwab's **account-hash
indirection** and the first whose response schema is entirely undocumented.

### Account-hash indirection

Transactions are per-account and keyed by an opaque hash, not the account number:

1. `GET /trader/v1/accounts/accountNumbers` → `[{accountNumber, hashValue}]`.
2. `GET /trader/v1/accounts/{hashValue}/transactions?startDate=…&endDate=…&types=…`.

The `hashValue` is validated (`[A-Za-z0-9]`, ≤128 chars) **before** it is
interpolated as a URL path segment — a stray `/` or `?` would change the endpoint.
The accountNumbers response holds plaintext account numbers, so it is parsed for
`(masked number, hash)` and `json_object_put` immediately, never logged.

### Date window

Unlike history (epoch-ms), transactions want **ISO-8601 datetime** strings
(`YYYY-MM-DDT00:00:00.000Z` / `T23:59:59.000Z`, confirmed accepted live). The
service reuses the strict `YYYY-MM-DD` validator, adds a **~1-year lookback
ceiling** (Schwab serves at most about a year; `SCHWAB_TXN_MAX_LOOKBACK_DAYS =
365` is a conservative default — the exact edge, 365 vs 366 vs 400, was not
probed and is a tracked follow-up), a future-start guard, an `end < start`
reject, and a future-`end` clamp. With no dates it defaults to the last 60 days.
`startDate`/`endDate`/`types` are effectively required, so all three are always
sent (the category ENUM maps to a comma-joined Schwab type list).

### Pinned response schema (from a live-capture spike)

Because the schema is undocumented, it was pinned by a **live spike**: an
env-gated raw-body dump hook (removed before commit) captured real responses to a
0600 scratch file, which a PII-redacting script summarized. The confirmed row
shape:

```
{ type, status, subAccount, netAmount, time ("YYYY-MM-DDThh:mm:ss+0000"),
  tradeDate, settlementDate, description, activityId, orderId, positionId,
  accountNumber (PII), qualifiedDividend?,
  transferItems: [ { amount, cost, price, feeType?, positionEffect?,
                     instrument: { symbol, assetType, description, closingPrice,
                                   instrumentId, status, type, uniformSymbol } } ] }
```

### Classification is leg-and-sign, not `type` alone

The critical finding: Schwab's outer `type` covers economically opposite events,
and key facts live in the legs, not the row. `schwab_txn_classify(row → schwab_txn_t)`:

- **Trade** (`TRADE`): the security lives in the one **non-CURRENCY** leg
  (`instrument.symbol`, `amount` = signed shares → buy/sell, `positionEffect`).
  The CURRENCY legs carrying a `feeType` (COMMISSION/SEC_FEE/OPT_REG_FEE/TAF_FEE)
  are the fees — fees are legs *inside* a trade, not a transaction type.
- **Dividend/interest** (`DIVIDEND_OR_INTEREST`): a single CURRENCY leg (the
  cash). The paying security is **only in the prose `description`** — there is no
  symbol field — so a symbol filter cannot reliably match a dividend, and the
  payer is surfaced from `description`.
- **Deposit/withdrawal** (`ACH_*`/`WIRE_*`/`CASH_*`/`ELECTRONIC_FUND`): external
  cash flow, direction by the sign of `netAmount`.
- **Internal** (`JOURNAL`/`MONEY_MARKET`/`SMA_ADJUSTMENT`/`MARGIN_CALL`/
  `MEMORANDUM`/`RECEIVE_AND_DELIVER`): internal moves and sweeps.

Each record carries an explicit **`external`** flag (deposit/withdrawal = external;
everything else internal). This is deliberately built now, unused, so a future
money-weighted-return round can consume "external flow on date D" without a
reclassification pass.

### Aggregation, listing, and reconciliation

Per account, the service aggregates over **every** returned row (`schwab_txn_agg_t`:
counts and net cash per category, income, fees), then renders a newest-first
listing capped at 50 rows/account (sorted lexicographically on the uniform `time`
string — no `strptime`). Internal sweeps are hidden from the `all` view but
counted, and an "Internal moves" summary line is always emitted so the per-category
counts **reconcile to the header total** even in a filtered view (a `deposits`
filter also returns `JOURNAL` rows).

## Portfolio unrealized P/L

The portfolio view sums per-position `longOpenProfitLoss` into an account-level and
combined **"Unrealized P/L (long positions)"** line with a % against summed cost
basis. Labeled "long" because short-position P/L lives in a different field;
realized P/L is not attempted (the transaction rows carry no per-lot basis).

## Security

- **Account numbers** are masked to the last 4 before any egress; the raw
  `/accountNumbers` root and per-row `accountNumber` are never rendered or logged.
- **Free-text memos** are passed through `schwab_txn_mask_numbers`, which masks any
  digit token carrying ≥5 digits — including groups split by a single separator
  (`1234 5678`, `123-45-6789`) — so a transfer memo can't leak an account/routing/
  SSN-shaped number. Short numbers (dates, share counts) pass through.
- **Injection surface**: every attacker-influenceable component of the request URL
  is either whitelisted (`type` → static Schwab type set), charset-validated
  (`symbol` via `clean_symbols` `[A-Z0-9.]`; `hashValue` `[A-Za-z0-9]`), or fully
  regenerated by the daemon from a validated integer (dates → fixed `strftime`).
  No untrusted byte reaches the URL.
- **Transport**: TLS verify on, protocol pinned to https, redirects disabled
  (`FOLLOWLOCATION=0`), response body `sodium_memzero`'d from heap after parse.
- **Data class**: `tool_get_current_user_id()` falls back to user 1 with no
  session/scheduled context, so a context-less caller reads user 1's activity — the
  documented, accepted per-surface model. Transaction/cost-basis data is more
  sensitive than a balance snapshot; `THREAT_MODEL.md` item 10 records that tighter
  gating on shared/always-on voice surfaces is a deliberately-accepted, deferred
  risk.

Both the transactions round and the whole change were reviewed by the
correctness-reviewer and security-auditor (0 critical/high; every medium/low
applied).

## Testing

- `tests/test_schwab_stats.c` — 7 Unity tests over the price analytics (bad
  inputs, single candle, monotone rise, drawdown/vol, base selection, SMAs,
  annualization).
- `tests/test_schwab_txn.c` — 9 Unity tests over the classifier and memo masker,
  against a **scrubbed synthetic fixture** shaped from the live capture (trade with
  fee legs, sell, dividend-with-no-symbol, journal, ACH deposit/withdrawal), plus
  the digit-masking cases. If Schwab's schema drifts, these go red.
- Live-verified end-to-end on a real linked account (102 transactions classified,
  filters reconcile, account masked).

## Scope boundaries / deferred

- **Trading** — a separate future tool with a dangerous-action confirmation gate.
- **Money-/time-weighted return (MWR/TWR)** — infeasible for lifetime performance
  (no inception cash flows past the ~1-year window, no historical valuations). A
  *trailing-window* Modified-Dietz is reconstructable in a future round from
  current holdings + this round's external-flow tags + one pricehistory call per
  symbol; fragile (splits/ACAT-in arrive as `RECEIVE_AND_DELIVER cost:0`), so its
  own round.
- **Daily-snapshot foundation** — persisting a daily portfolio valuation is the
  real unlock for true performance-over-time and benchmark comparison; it is what
  the API cannot provide retroactively.
- **Exact lookback ceiling** — confirm 365 vs 366/400 days on a live session and
  adjust `SCHWAB_TXN_MAX_LOOKBACK_DAYS`.

## Files

| File | Role |
|------|------|
| `src/tools/schwab_tool.c` | LLM tool metadata + action dispatch |
| `src/tools/schwab_service.c` | Orchestration, formatting, masking, date validation, per-account fan-out |
| `src/tools/schwab_client.{c,h}` | Thin authenticated JSON GET; `schwab_rc_t` |
| `src/tools/schwab_stats.{c,h}` | Pure price-history analytics |
| `src/tools/schwab_txn.{c,h}` | Pure transaction classifier + aggregates + memo masking |
| `include/tools/schwab_service.h` | Service prototypes |
| `docs/SCHWAB_SETUP.md` (dawn repo) | Operator setup + enrollment walkthrough |
