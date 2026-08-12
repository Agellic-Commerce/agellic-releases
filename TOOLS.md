# Tool Reference

This document is the practitioner reference for the 12 MCP tools exposed by
agellic-mcp v2.0.1. Each section covers what the tool does, what
it costs in Keepa tokens, what inputs it accepts, what it returns, and the
operating rules worth knowing before you turn it loose on a candidate set.
All Keepa token costs are concrete numbers measured against current
runtime behavior, no hedging.

agellic-mcp pulls every figure from Keepa (`keepa.com`) and your Keepa
subscription supplies the token bucket. If you don't have a Keepa account
yet, grab one at [keepa.com/#!api](https://keepa.com/#!api) before you
start. The default Keepa subscription is **20 tokens per minute** (1,200
token bucket); several calls in this document note where that ceiling
decides whether a call answers inline or runs itself in the background.

## Quick scan

| Tool | One-line summary | Token cost |
|------|------------------|------------|
| `execute_keepa_finder` | Discover ASINs by category / brand / price / rank / competition filters; optional market-insights stats. | 10 base + 1 per started 100 of `perPage` (stats: +30 base + 1 per whole million total matches) |
| `screen_products` | Bulk-screen up to 500 ASINs per call into a pipe-delimited Core-10 signals table (BSR, sellers, Buy Box, OOS, etc.), staged higher across calls. | 3 per uncached ASIN |
| `get_product_details` | Deep per-product analysis: offers, Buy Box rotation, stock depth, calibrated demand, insights. Also resolves UPC/EAN/ISBN → ASIN. | 16 per uncached ASIN quoted worst-case, ~6-8 actual (code lookup: 1 per candidate up to `codeLimit`) |
| `resolve_cross_border` | Map source ASINs from one marketplace to equivalents on another via product codes; returns price gaps. | 12 per source ASIN quoted flat (3 source + 9 target), settled lower |
| `resolve_codes` | Bulk-resolve supplier UPC/EAN/GTIN/ISBN codes (up to 500 rows) to candidate ASINs at the identity tier. | ~1 per returned candidate (≈1/code typical) |
| `get_product_chart` | Fetch a price/BSR history PNG chart for one ASIN on one marketplace. | 1 flat per call (Keepa 90-min server cache) |
| `check_token_balance` | Show current Keepa token balance, refill rate, and cache-aware per-tool cost projections. | 0 (local) |
| `confirm_work_order` | Authorise a work order that was quoted but not started, because it costs more than ~1 hour of token refill. | 0 (local) |
| `check_job_status` | List, poll, cancel, or page through the rows of a `wo_…` work order. The status report carries progress, spend, and a live queue-aware ETA. | 0 (local) |
| `get_finder_result` | Page through ASINs from a stored finder result set by id. | 0 (local) |
| `get_cross_border_result` | Retrieve a cached cross-border analysis by id, or list recent ones. | 0 (local) |
| `get_codes_result` | Page through a stored code-resolution result set (the per-row candidate table). | 0 (local) |

The 6 free tools (`check_token_balance`, `confirm_work_order`,
`check_job_status`, `get_finder_result`, `get_cross_border_result`,
`get_codes_result`) work against local state only and never charge Keepa
tokens. Cached re-reads of the paid tools are also free for 24 hours.

Domain values referenced throughout: `1`=US, `2`=UK, `3`=DE, `4`=FR,
`5`=JP, `6`=CA, `8`=IT, `9`=ES, `10`=IN, `11`=MX, `12`=BR.

---

## Every call is a work order

This is the biggest change in v2.0.0. Every call to the five data-pulling
tools (`execute_keepa_finder`, `screen_products`, `get_product_details`,
`resolve_cross_border`, `resolve_codes`) now becomes a durable **work
order** with its own `wo_…` id. The order is a record of exactly what was
asked for, what it may spend, and what it has settled so far, which is what
makes a large run survivable: it funds itself batch by batch as tokens
refill, resumes after a dropped session, and never re-charges a row it
already fetched.

A balance too small to cover the work is no longer an error. Every such call
returns exactly one of three answers, and it is worth knowing which one you
got:

1. **The result, inline.** The balance covered the quote, so the work ran in
   the call. You get the usual output plus an `orderId` and a result-set
   handle you can re-read later at 0 tokens.
2. **A background acceptance.** There weren't enough tokens yet, so the
   order was accepted anyway and runs itself as they refill. No results yet:
   you get the `orderId`, a `Cost:` line, and a `Queue:`/`ETA:` pair. Poll
   with `check_job_status` and read rows as they land with its `fetch`
   action. Do not re-issue the original call, which would create a second
   order for the same work and pay for it twice.
3. **A consent quote.** The work would take more than roughly one hour of
   token refill, so nothing starts and nothing is charged. You get the
   `orderId`, a `Quote:` line, a one-time `confirmToken`, and the same
   `Queue:`/`ETA:` pair framed as "If you confirm now:". The order starts
   only once you agree and `confirm_work_order` is called, and confirming
   **queues** it, which is not the same as starting it.

`execute_keepa_finder` is the exception to answer 3: one finder query is one
indivisible Keepa call, so there is no consent gate. It runs inline, waits
in the background, or (when the query would cost more than the bucket can
ever make available to a single call) is refused at the door with nothing
created and nothing charged.

`get_product_chart` is outside this model entirely. At 1 flat token it is
never a work order, and on a drained bucket it just asks you to retry in a
few seconds.

### Cost and time are two separate claims

The `Cost:` / `Quote:` line is tokens only (`worst case N tokens at R/min`)
and says nothing about how long the work takes. The wait is the separate
`Queue:`/`ETA:` pair beneath it: how many orders are ahead and what they
still owe, then a **bound**, "done within ~X of token refill".

That ETA is recomputed from scratch every time it is rendered and is never
stored, so it counts **down** across polls as the queue drains and the
balance refills. Three conditions ride with it, and they are worth repeating
whenever you pass the number on to someone else:

- It bounds the work **as currently quoted**. Every order ahead is priced at
  its own quote, and an order can outgrow its quote (cached entries expiring
  mid-run, for instance), so this is a bound rather than an unconditional
  ceiling.
- It assumes **no other work is running on this Keepa key**. Queued orders
  are the only competition it can see; a second machine or a second edition
  spending the same key is invisible to it.
- Its wall-clock figure assumes an Agellic session is open **~8 hours a
  day**. An always-open session finishes roughly 3× sooner.

Quotes are worst-case ceilings; charges settle at Keepa's own figure as each
batch commits, so an order quoted at 400 tokens routinely settles for a
fraction of that. Mid-run, the charged total on a status report also
includes tokens reserved for requests still in flight. That reserve releases
when they settle, so the figure can go **down** before the order ends.

### Running, watching, stopping

- Work advances **only while an Agellic session is open**. Quit the app or
  let the machine sleep and the queue pauses; relaunch and it resumes where
  it left off with nothing lost. There is no cloud worker.
- **Polling is free and speeds nothing up.** Only the passage of time and
  token refill move an order along, and the ETA counts down with them, not
  with how often you look.
- **Any order can be cancelled** with
  `check_job_status({ jobId: "<orderId>", action: "cancel" })`. One that has
  not started stops instantly; a running one stops at its next dispatch
  boundary, and everything it settled stays readable with `fetch`.
- Results persist as result-set handles: `screen:<orderId>`,
  `lookup:<orderId>`, `finder:<orderId>`, `codes:<orderId>`, and `xb_…` for
  cross-border. Finished orders are kept for **14 days**, and a result-set
  view whose own 24-hour TTL has lapsed is rebuilt on demand, for free, by
  `check_job_status`.

---

## `execute_keepa_finder`

Discovery search across Amazon's catalog. Returns ASINs matching
category / brand / price / rank / competition filters, plus optional
market-insights stats when you ask for them.

**You speak in natural language.** Describe what you're after ("kitchen
items under $40, rating ≥ 4.2, 2–5 sellers, BSR < 10K") and the
assistant picks the Keepa filter names, scales (cents not dollars, mm
not inches), and time windows for you. The filter reference below is
what the LLM consults to pick names and units, not a syntax you type
yourself. The "Natural-language to filter mappings" subsection at the
end of the section shows sample translations.

### What it's good for

- The user has no ASINs yet and wants to find products matching criteria.
- The user wants to **size a market**: fetch `includeStats=true` on a
  follow-up call after a filter-only recon to get avg landed Buy Box
  price (`avgBuyBox`: item + shipping, the same amount Amazon charges
  its referral fee on), seller counts, Amazon share, brand
  fragmentation, FBA share, avg rating, and avg review count across the
  matched set.

Not the right tool when the user already has ASINs: go straight to
`screen_products` or `get_product_details`.

### Token cost

- **Base: 10 tokens** + **1 token per started 100 ASINs returned**.
  `perPage=100` costs 11, `perPage=1000` costs 20.
- **Stats (`includeStats=true`):** +30 base + 1 per whole completed million
  of `totalResults` (`⌊totalResults / 1,000,000⌋`, so under 1M matches the
  surcharge is exactly the flat 30).
- `perPage` is a **ceiling on cost, not a fixed charge**. Keepa bills
  `10 + ⌈min(perPage, totalResults)/100⌉`. So `perPage=10000` on a
  4,231-match query costs **53 tokens**, not 110. Even the worst case
  (`perPage=10000` returning a full 10,000 ASINs) is 110 tokens.
- **`perPage` has an effective floor of 100.** Keepa charges per *started*
  100 results, so `perPage=1`, `50` and `100` all cost the same 11 tokens.
  Asking for fewer than 100 buys nothing and forfeits ASINs you already paid
  for.
- A broad **single-filter** search can match hundreds of millions of
  products and cost 300+ tokens. Always combine ≥ 2 filters.

**Pricing insights accurately: `expectedTotalResults`.** The stats surcharge
is charged on the count the call actually matched, which nobody knows before
it returns, so this is the one call that can charge more than it quoted. The
fix is to run insights as a pair: the same filters once with `includeStats`
omitted (cheap), then again with `includeStats: true` **and**
`expectedTotalResults` set to the `totalResults` that first run reported.
The hint is local, used only to price the quote, and never sent to Keepa.

### Inputs

All filters are ANDed; array filters are ORed (max 50 entries each). At
least one real filter is required besides `domain` / `page` / `perPage`
/ `sort` / `stats`.

Common numeric range filters (use `_gte` / `_lte` suffixes):

- `current_BUY_BOX_SHIPPING`: Landed Buy Box price, item + shipping, in
  cents ($40 → 4000). The default when the user says "price".
- `current_SALES`: Best Sellers Rank (lower = better).
- `current_RATING`: 0–50 scale (4.5 stars → 45).
- `current_COUNT_NEW`: Listed new-seller count. Default when the user
  says "sellers".
- `delta30_BUY_BOX_SHIPPING`, `delta90_BUY_BOX_SHIPPING`: Landed Buy Box
  price delta vs avg (positive = price dropped).
- `deltaPercent30_SALES`, `deltaPercent90_SALES`: BSR delta percent
  (positive = rank improved).
- `packageWeight` (grams; 1 lb = 454g), `packageHeight` / `packageLength`
  / `packageWidth` (mm; 1 in = 25.4mm).
- `outOfStockPercentage90`: Percent of time out of stock.
- `buyBoxStatsAmazon`: Percent of time Amazon holds the Buy Box.
- `buyBoxStatsSellerCount30/90/180/365`: Unique Buy Box winners.
- `buyBoxStatsTopSeller`: Top seller's BB win share (low = rotating).
- `buyBoxIsAmazon`, Boolean: Amazon currently holds the Buy Box.

String / array filters:

- `brand`, `title`, `manufacturer`: prefer `brand` unless the user
  explicitly says "manufacturer".
- `rootCategory`, `categories_include`: numeric IDs only. Use only when
  you have a trusted ID from a prior tool result or explicit user input.
- `categoryHint`: text description (e.g. `"kitchen"`, `"Bücher"` on
  DE). Resolved server-side via fuzzy match. Single match → injected as
  `rootCategory` / `categories_include`; multiple → ambiguity error with
  candidates; zero → actionable error. Never pass `categoryHint`
  together with `rootCategory` / `categories_include`.

Price stability and flip filters:

- `buyBoxStandardDeviation30/90/365`: Landed Buy Box price volatility in
  cents. `_lte` for stable, `_gte` for volatile.
- `flipability30/90/365` (0–255): dip+rebound score. `_gte` for
  swing-prone, `_lte` to exclude.
- Timeframes: 30d = recent, 90d = typical sourcing window, 365d =
  seasonal patterns.
- High flipability + moderate std dev often reflects stable pricing with
  occasional dips, a useful sourcing signal.

Competition and seller field routing:

- "How many sellers?" → `current_COUNT_NEW` (listed). Not
  `buyBoxStatsSellerCount{30,90,180,365}` (BB winners).
- "Is Amazon on this?" → `buyBoxIsAmazon` or `buyBoxStatsAmazon` (%
  over time). Not `outOfStockPercentage90`.
- "Amazon OOS?" → `outOfStockPercentage90`. Not `buyBoxStatsAmazon`.
- "Competitive Buy Box?" → `buyBoxStatsTopSeller_lte` (low = rotating).
- "Buy Box winners?" → `buyBoxStatsSellerCount{30,90,180,365}` (default
  90).
- Seller-count delta: `delta = avg − current`. `delta30_COUNT_NEW_gte: 3`
  means 3 sellers LEFT (avg was higher than current).

### The full filter surface

The fields above are a sampler. The strict validator accepts over a
thousand named filters, generated from a handful of patterns crossed
with Keepa's price types and time windows:

- `current_<TYPE>_{gte,lte}`: current value
- `avg{7,30,90,180,365}_<TYPE>_{gte,lte}`: windowed averages
- `delta{1,7,30,90,Last}_<TYPE>_{gte,lte}`: absolute delta vs that
  window's avg (or previous value for `deltaLast`)
- `deltaPercent{1,7,30,90}_<TYPE>_{gte,lte}`: percent delta
- `backInStock_<TYPE>`: was OOS in last 60d, now has an offer
- `isLowest_<TYPE>` / `isLowest90_<TYPE>`: current is the all-time / 90d low
- `lastPriceChange_<TYPE>_{gte,lte}`: timestamp filter (Keepa minutes)

`<TYPE>` covers the 24 base Keepa types (AMAZON, NEW, USED,
BUY_BOX_SHIPPING, NEW_FBA, NEW_FBM_SHIPPING, COUNT_NEW, COUNT_USED,
COUNT_REVIEWS, RATING, SALES, LISTPRICE, COLLECTIBLE, REFURBISHED,
WAREHOUSE, LIGHTNING_DEAL, TRADE_IN, …); `avg`, `deltaPercent`,
`isLowest`, and `lastPriceChange` also accept 5 extras
(EBAY_NEW_SHIPPING, EBAY_USED_SHIPPING, PRIME_EXCL, RENT,
BUY_BOX_USED_SHIPPING).

Anything that doesn't fit a top-level pattern goes in `advancedFilters:
{ ... }`, a passthrough map of `Keepa-field → value`. The strict layer
validates either way: bad names come back as actionable errors, not
silent drops, so the working posture is **try the filter and adapt from
the response** rather than refuse because a field isn't named in the
section above.

### What it returns

- **Complete result set** (`asins.length === totalResults`): emits a
  bracketed handle line: `[handle: <id> · <count> products · expires
  24h]`. The id is machine metadata; the assistant talks in counts and
  filters, not raw ids. Downstream tools accept the id directly.
- **Partial slice** (`asins.length < totalResults`): emits
  `[partial: <id> · <fetched> of <total> · <advisory>]`. The slice is
  cached too, but the label says refine, or get explicit user
  acceptance, before any downstream tool call uses it.
- **Not funded yet:** no ASINs at all. The search is accepted as a
  background work order and returns an `orderId` (`wo_…`), its cost, and a
  queue-aware ETA bound. Poll with `check_job_status`; the handle arrives
  through its `status` or `fetch` action once the query lands. A partial
  slice is handed off the same way, with the same "refine first" caveat.
- **Refused at the door:** the query would cost more than this Keepa bucket
  can ever make available to a single indivisible call. The error names the
  cost and the cheaper query to run (usually `perPage=100`, or dropping
  `includeStats`, or refining below the fundable match count). Nothing is
  created and nothing is charged. A finder query is never subject to the
  consent gate, so `confirm_work_order` never sees one.
- **Zero results:** plain-text message including the actual charge (a
  zero-match query still bills); the model suggests filter adjustments.

### perPage

`perPage` defaults to **1000**. Range: 50 to 10,000. For queries with
`totalResults ≤ 1000`, the default produces a complete handle on the
first call. For 1,000–10,000 `totalResults`, re-call with
`perPage=10000` to upgrade the partial to a complete handle. Above
10,000, refine: no `perPage` value reaches a complete result set, and a
partial slice of an over-cap query isn't actionable downstream without
explicit user acceptance.

### Operating discipline

- **Iterative workspace.** Finder is a loop, not a one-shot. Start
  broad, narrow with the user, accept the slice the user actually wants.
  Exit condition is user satisfaction, could be 20 products, could be
  9,000.
- **10,000 is a ceiling, not a target.** Keepa caps a single page at
  10,000 ASINs; queries returning more than that cannot be captured as a
  complete result set without refinement. Above 10K, the right move is
  tighter filters (tighter price band, tighter BSR band, higher OOS
  threshold, a sub-category, etc.), not a top-N slice, unless the user
  explicitly prefers a slice.
- **`includeStats=true` is the primary refinement instrument** when
  intuition isn't enough. Insights summarize the matched set so the next
  filter move is obvious. Never set on the first call: run filters-only
  first to learn `totalResults`, then decide whether stats are worth the
  cost. Above 50M matches the cost gets steep; refine before requesting.

### Data and unit invariants

- Prices: integer smallest currency unit. `$50` → `5000`.
- Weights: grams. "Under 2 lbs" → `packageWeight_lte: 907`.
- Dimensions: millimeters. "Under 6 inches" → `packageHeight_lte: 152`.
- Ratings: 0–50 (stars × 10). 4.5 stars → `45`.
- Percentages: integers 0–100 unless noted.
- Delta sign: positive delta = price DROPPED (price-like) or rank
  IMPROVED (BSR).

### Natural-language to filter mappings

- "under $40" → `current_BUY_BOX_SHIPPING_lte: 4000`
- "$20 to $50" → `current_BUY_BOX_SHIPPING_gte: 2000,
  current_BUY_BOX_SHIPPING_lte: 5000`
- "at least 4.2 stars" → `current_RATING_gte: 42`
- "BSR under 10,000" → `current_SALES_lte: 10000`
- "2 to 5 sellers" → `current_COUNT_NEW_gte: 2, current_COUNT_NEW_lte: 5`
- "under 2 lbs" → `packageWeight_lte: 907`
- "under 6 inches tall" → `packageHeight_lte: 152`
- "price dropped by at least $10 vs 30d avg" →
  `delta30_BUY_BOX_SHIPPING_gte: 1000`
- "rank improved by at least 20% vs 30d avg" →
  `deltaPercent30_SALES_gte: 20`

---

## `screen_products`

Bulk screening tool. Compresses each product into 10 Core screening
signals plus 3 identity columns, returned as a pipe-delimited table, one
row per ASIN.

### What it's good for

- The user has a candidate set (typically 50–500 ASINs from a finder
  result, an external list, or a prior screen) and wants to rank, sort,
  or filter by Core-10 signals like BSR, Buy Box price, seller count,
  Amazon presence, OOS percentage, or 30-day price drops.
- Cheaper than `get_product_details` (3 tokens vs 16 quoted) and 10× the
  capacity per call (500 vs 50). Use it whenever you don't need offer
  ladders or stock depth.

### Token cost

- **3 tokens per uncached ASIN** (lite fetch: history + stats + Buy Box;
  no offers / stock / rating).
- **0 tokens for cached ASINs** (24h product cache).
- Duplicates in the input list are auto-removed before billing.
- Tokens are spent batch by batch as the order runs, and each batch settles
  at Keepa's own figure. A screen that stops early or is cancelled is
  charged only for the rows it actually fetched.

### Inputs

Mutually exclusive, pass exactly one:

- **`resultSetId`**: id from a prior `execute_keepa_finder`,
  `get_product_details`, or `screen_products` call. The tool resolves
  ASINs server-side so you don't burn context dumping them inline.
  Still capped at 500 per call. Its ASINs are copied into the order at
  once, so the order outlives the source set's own 24h expiry.
- **`asins`**: up to 500 ASINs. Use when you have an explicit list.

Optional on any call:

- **`maxTokenSpend`**: a hard ceiling. The order stops rather than exceed
  it, keeping everything it settled. Worth setting whenever token spend is
  a concern.
- **`deadlineMinutes`**: give up this long after **authorisation**. The
  clock runs while the order waits in the queue, not from its first Keepa
  request.
- **`orderId` + `stage`**: staging, below.

Both ceilings are fixed on the call that creates the order. Passing a
different value on a staging continuation is refused, not silently ignored.

### Staging: manifests larger than 500

The 500 cap bounds one request's payload, not the screen. To screen 1,200
ASINs, send three `stage: true` calls (500, 500, 200) chained by the
`orderId` the first one returns, then a final call with no ASINs that seals
the manifest, quotes it, and starts the work. Nothing is priced or fetched
until that final call, and an order left staged is discarded after 24 hours.

### What it returns

A pipe-delimited text table. The 13-column header is:

```
ASIN|BSR|Sold|BB(c)|Trend|Sellers|Amz|FBA(c)|Ref%|Drops30|OOS90|Brand|Title
```

| Column  | Description                                          |
|---------|------------------------------------------------------|
| ASIN    | Amazon ASIN (identifier)                             |
| BSR     | Current Best Sellers Rank (primary category)         |
| Sold    | Estimated monthly sales                              |
| BB(c)   | Landed Buy Box price (incl. shipping), cents         |
| Trend   | Landed Buy Box price trend (30d vs 90d landed avg)   |
| Sellers | New offer count (incl. Amazon + brand)               |
| Amz     | Whether Amazon itself sells the product              |
| FBA(c)  | FBA pick-and-pack (fulfillment) fee in cents         |
| Ref%    | Referral fee percentage                              |
| Drops30 | Sales-rank drops in last 30 days (sales proxy)       |
| OOS90   | Amazon out-of-stock percentage over the last 90 days |
| Brand   | Brand name (a gibberish brand is the private-label tell) |
| Title   | Truncated core of the Amazon title (≤70 chars), which separates rows on a branded screen where Brand is uniform |

Values are plain numbers (Brand and Title are text). Nulls render as
dashes. A summary line at the top reports
`requested:N fetched:N cached:N failed:N tokens:N`, followed by the
`resultSetId` and the `orderId`.

The result set is stored under `screen:<orderId>` (24-hour TTL, rewritten
after every batch) so downstream tools can reference a screen that is still
running.

### Which of the three answers you got

A screen returns the table inline when the balance covers the quote, a
background acceptance when it doesn't, or a consent quote when the work
exceeds ~1 hour of refill (see
[Every call is a work order](#every-call-is-a-work-order)). A `PARTIAL —`
line above the table means the run stopped early and the rest continues in
the background: poll it rather than re-calling, since a second call creates
a second order for the same ASINs.

### Limitations

- **500 ASINs per call.** Beyond that, either narrow via finder filters, use
  `get_finder_result({ resultSetId, limit: 500 })` to take a BSR-sorted
  top-500 slice, or stage the manifest across several calls.
- **Big screens on low-TPM Keepa plans run in the background.** The default
  Keepa subscription is 20 TPM (bucket capacity 1,200). A 500-ASIN screen
  costs 1,500 tokens, roughly 75 minutes of refill, so on that plan it comes
  back as a background acceptance or a consent quote rather than a table; on
  100+ TPM plans (capacity 6,000+) the same call usually runs inline. Read
  the settled rows with `check_job_status({ jobId, action: 'fetch' })` as
  they land.
- Force-refreshing cached data is not supported. The Keepa cache is used
  automatically when it satisfies screening requirements.

### When to use vs `get_product_details`

Screening gives 10 signals plus ASIN, Brand and a truncated Title per row,
usually enough to rank and filter a candidate set. `get_product_details` is a separate
choice for a small shortlist that needs per-seller offers, Buy Box
rotation, stock depth, or calibrated demand. The two are independent
choices, not sequential steps: sometimes a screen is all you need.

---

## `get_product_details`

Deep product analysis: offers, Buy Box rotation, stock depth, calibrated
demand range, sales rank history, trend analysis,
review velocity, insights, and economics. Also resolves a single
UPC/EAN/ISBN code to ASINs on one specified marketplace.

### What it's good for

- **Deep dive after screening.** The user picked a shortlist from a
  `screen_products` table and wants per-seller offers, Buy Box rotation,
  stock depth, or the calibrated demand range.
- **Single-product identification by code**: UPC/EAN/ISBN on one
  specified marketplace. (For marketplace-to-marketplace comparison, use
  `resolve_cross_border` instead.)

### Token cost

**Enriched fetch (`asins` / `resultSetId` modes):**

- **16 tokens per uncached ASIN, worst case.** That is the measured
  enriched ceiling the order quotes and authorises per row (two 6-token
  offer pages, stock, rating, graph). Every dispatch reserves it, then
  settles at Keepa's own charge, so **measured actuals land around 6-8
  tokens per ASIN**. Quote the ceiling, report the ledger figure as the
  cost.
- **0 tokens for cached ASINs** (24h product cache). They are excluded from
  the quote entirely.
- Partial cache hits: only uncached ASINs are quoted and charged.

**Code lookup (identification tier):**

- **`codeLimit` tokens per call (default 3).** Keepa charges 1 token per
  returned candidate, capped at `codeLimit`.
- No offers / stock / rating enrichment in code mode.

### Inputs

Mutually exclusive, pass exactly one:

- **`resultSetId`**: id from a prior `execute_keepa_finder`,
  `screen_products`, or `get_product_details` call. The tool resolves
  ASINs server-side. Max 50 ASINs in the stored set (narrow first if
  larger).
- **`asins`**: up to 50 ASINs with full enrichment. Use when you have
  an explicit list.
- **`codes`**: resolve exactly one UPC/EAN/ISBN/GTIN to matching ASINs
  on the specified marketplace. The schema accepts an array for forward
  compatibility, but >1 is rejected at runtime.

Pass `update=null` unless the user explicitly asks for fresh data.
`update=0` forces a live Keepa crawl even if the data was fetched minutes
ago, and because a force-refresh treats every ASIN as uncached, it is quoted
at the full 16 tokens per ASIN. The 24h product cache is correct for almost
every workflow.

Every order materializes a `lookup:<orderId>` result set as its rows settle,
so a follow-up call can re-read the same ASINs by id at 0 tokens.

### What it returns

Per resolved ASIN:

- **`identity`**: title, brand, category, manufacturer, `productCodes`
  (UPC/EAN/GTIN), Amazon URL, image URL.
- **`pricing`**: current lowest new price, list price, Buy Box price,
  30/90/180d averages, trend direction/strength, and
  the **sell-price read** (`pricing.sellPrice`): observed sale-price
  bands (`moveFastCents` / `marketCents` / `stretchCents` = p25 / median
  / p75 of prices in force at inferred sale moments), the current
  price's position inside them, sales skew, drift, and caveats. Bands
  describe the observed market, never a recommendation.

  Every Buy Box price in this block is **landed** (item + shipping),
  Keepa's native `csv[18]` basis: `pricing.buyBox.currentCents`, the
  30/90/180/365d averages, and the 365d min/max.
  `pricing.buyBox.shippingBasis: 'landed'` marks that contract, and its
  absence marks a stale pre-landed payload. When shipping is greater
  than zero, `itemCents` and `shippingCents` show how the landed price
  splits; on free-shipping items both are absent.
  `pricing.sellPrice` declares its own `shippingBasis` (`landed`,
  `item-only`, or `mixed-window`) and carries
  `marketState: 'suppressed'` when the Buy Box is suppressed.
- **`sales`**: current and historical sales rank, primary + leaf BSR
  with category names, 30/90/180d drops, Amazon-reported monthly sold
  badge (when available; the model's range estimate lives in `demand`).
- **`demand`**: Demand Read v3, a velocity-implied monthly-sales estimate
  (not a verified figure), discriminated on `mode`. **`read`** carries the
  model estimate: `centerLikely` (the conservative plan-on number),
  `rangeLow` / `rangeHigh`, a ready-to-print `hero` range, confidence
  (`high` / `medium` / `low`), and `source`. **`badge`** means Amazon
  reports an "X+ bought past month" figure, which is ground truth, so the
  model is suppressed and the badge value is echoed. **`no-read`** means
  nothing usable, with a `reason` saying why. All three carry `caveats`,
  free-text qualifiers like "estimate from cross-marketplace baseline".
- **`seasonality`**: whether a recurring seasonal peak exists and how
  sure the detector is. `confirmationLevel` separates `confirmed` (the
  peak recurred across 2+ years) from `candidate` (one season observed,
  do not act on it alone); also the peak week window, calendar label
  (Q4, Summer, etc.), current phase (`pre-peak` / `peak` / `post-peak` /
  `off-season`), and concrete sourcing-window / lead-out dates.
- **`competition`**: individual seller offers (FBA/FBM, prices, stock
  depth), Buy Box current winner, dominant seller + win %, rotation
  table, historical avg seller count. Offers are ordered by **landed**
  price, so the offer at the top is the cheapest to actually receive,
  not the one with the lowest sticker and a shipping charge behind it.
  `competition.buyBox.priceCents` is the same landed value and basis as
  `pricing.buyBox.currentCents`.
- **`reviews`**: rating, review count, velocity (added 30/90/180d),
  trend (accelerating/steady/slowing), historical avg.
- **`supply`**: Amazon OOS 30/90/180d, marketplace OOS 90d.
- **`insights`**: trend signals, Buy Box volatility,
  effective competition (sellers within 5% of BB), IP risk,
  race-to-bottom warning. See
  [`COMPUTED-INSIGHTS.md`](COMPUTED-INSIGHTS.md) for the algorithms
  behind every field in `demand`, `seasonality`, `pricing.sellPrice`,
  and `insights`, what each measures, the constants, and how to read
  it.
- **`economics`**: referral fee percent, FBA pick & pack fee, return
  rate.
- **`metadata`**: listing age, Subscribe & Save eligibility.
- **`tokensUsed`**: the order's ledger, meaning what Keepa actually charged
  for the rows settled so far, never the worst-case quote.

Like every other Keepa-calling tool, a lookup returns the products inline, a
background acceptance, or a consent quote (see
[Every call is a work order](#every-call-is-a-work-order)). A reply marked
**PARTIAL** names the `orderId` and the rest continues in the background:
poll it and page the settled products with
`check_job_status({ jobId, action: 'fetch' })`. **INCOMPLETE** and
**CANCELLED** are final, so the products under them are everything the order
settled.

### Cost discipline

State the worst-case spend before running on a stored result set
(`uncached_asins × 16` tokens, with actuals typically ~6-8 and the cache
absorbing anything fetched in the last 24h). Per-product output is **~5–10 KB** depending on offer count
(up to 20 offers per product) and insight verbosity. 10 ASINs ≈ ~75 KB
/ ~20K tokens of context; 50 ASINs ≈ ~375 KB / ~100K tokens. There's
no fixed "batch size target": state the spend, then let the user decide
the batch size.

### Limitations

- **50 ASINs per call, hard cap.** For larger sets use
  `screen_products` (500-cap, 3 tok/ASIN) and pick a shortlist.
- **`codes` mode is single-marketplace only.** It does not compare
  across marketplaces: `resolve_cross_border` is the right tool for
  that.
- **ASINs are NOT globally unique.** The same ASIN can mean different
  products on different marketplaces. Code-mode is for identification on
  one specified marketplace, not for cross-marketplace lookup.

### Data and unit invariants

- **Prices** are integers in the smallest currency unit (e.g. `1999` =
  $19.99 on US domain). Divide by 100 to render.
- **Ratings** use a 0–50 scale (stars × 10). `45` = 4.5 stars. Divide by
  10.
- **Keepa time values** are minutes since 2011-01-01 00:00 UTC. Convert
  via `new Date((keepaTime + 21564000) * 60 * 1000)`.

---

## `resolve_cross_border`

Map source ASINs from one Amazon marketplace onto equivalent listings on
another marketplace via EAN/UPC/GTIN product-code matching, with a
same-ASIN fallback for products that lack codes. Returns price gaps in
the source currency, sorted by gap descending (largest arbitrage
opportunity first).

### What it's good for

ASINs are **not globally unique**: the same ASIN can refer to different
products on different marketplaces. For source-ASINs →
target-marketplace comparison workflows, this is the correct tool, even
within regional groups (US/CA/MX, EU) where ASINs sometimes match,
because the tool verifies via product codes and title similarity.

Typical triggers:

- "Find the UK equivalent of these US ASINs and compare prices"
- "Which of my CA products are cheaper in the US market?"
- "Show me US→MX price gaps for this finder result"
- "Compare these 40 ASINs between Amazon US and Amazon DE"

Use `get_product_details` code lookup INSTEAD when the task is
single-product identification by UPC/EAN/ISBN on one marketplace
(without comparison to a second marketplace).

### Token cost

| Stage | Tokens |
|-------|--------|
| Source lite+buybox fetch | 3 per ASIN |
| Target code resolution (CODE_LIMIT=3 candidates × 3 tok each) | 9 per ASIN |
| **Worst-case total** | **12 per source ASIN** |

**The order is quoted flat at 12 per source ASIN, with no cache discount.**
A source already in the product cache has only had its source half done and
still owes up to 9 tokens of target-side resolution, so pricing treats every
row as uncached. A 50-ASIN batch is therefore authorised at 600 tokens
whether or not those sources are cached.

Actual cost is frequently much lower, and it settles as each batch commits:
cached sources skip their 3-token fetch, products with no codes skip stage 2
for a ≤3-token same-ASIN fallback, and fewer than CODE_LIMIT candidates
costs proportionally less. `summary.tokensUsed` on the result is the only
real charge. Quote the ceiling, then report that number.

### Inputs

Exactly one of `asins` or `resultSetId` must be non-null. `sourceDomain`
and `targetDomain` must differ (equal values are rejected).

- **`asins`**: explicit source ASIN list (hard maximum 500).
- **`resultSetId`**: id from a prior `execute_keepa_finder`,
  `screen_products`, or `get_product_details` call.

### What it returns

A **JSON-encoded text payload**: the outer MCP envelope is a text
content block; the text itself is `JSON.stringify(result)`. The parsed
object carries:

**`summary`**
- `sourceCount`, `matchedCount`, `unmatchedCount`. `sourceCount` is what the
  order set out to resolve (the whole manifest), not what has settled.
- `pendingCount`: source ASINs **not yet resolved**, present while rows are
  outstanding. Treat a pending source as one that has not been looked at,
  never as one that found no arbitrage.
- `orderEnded`: `true` once the order has reached a terminal state, which
  means its pending rows are final. Absent means the result is still filling
  in, so poll; present means stop polling and re-run those sources if you
  still want them.
- `avgGapPercent`: average % gap across priced matches (null if none
  priced)
- `sourceDomain` / `targetDomain`: human-readable marketplace labels
- `exchangeRate` / `exchangeRateDate`: the conversion rate applied and
  the date of the rate table in effect. Rates are bundled defaults
  (approximate, for ballpark comparison); Claude Code users can refresh
  them on demand with `node install.mjs --refresh-rates`, which writes a
  shared `~/.agellic-mcp/exchange-rates.json` override. CD-only installs
  stay on the bundled defaults until the next release. See
  [INSTALL.md](./INSTALL.md#refreshing-exchange-rates).

**`matches`** (array, sorted by `gapPercent` descending)
- `sourceAsin` / `targetAsin`, `sourceTitle` / `targetTitle`
- `sourcePrice` (local cents), `targetPrice` (target-currency cents),
  `targetPriceConverted` (source currency)
- `gapPercent` = `(sourcePrice − targetPriceConverted) / sourcePrice ×
  100`. Positive = cheaper on target.
- `confidence`: scored from match source + match count + title
  similarity. The similarity metric is **Dice-Sørensen on normalized
  titles** (threshold 0.8):
  - **`high`**: single code match AND similarity ≥ 0.8.
  - **`medium`**: single code match with low similarity, OR
    multi-candidate code match with high similarity, OR same-ASIN match
    with high similarity.
  - **`low`**: multi-candidate code match with low similarity, OR
    same-ASIN match with low similarity. Treat with caution.
- `sourceCurrency` / `targetCurrency`

**`unmatched`** (array)
- `sourceAsin` / `sourceTitle`, `reason` (`no_target_match` /
  `fetch_error`)

Results are stored under an `xb_` result id for 7 days. Retrieve later
with `get_cross_border_result`.

### Limitations and operating notes

- **500 ASINs per call, hard maximum.**
- **Inline, background, or consent.** A call runs inline as long as the
  balance covers the flat `12 × source_asins` quote. When it doesn't, the
  call is accepted as a background work order (or, over ~1 hour of refill,
  quoted for consent) rather than refused: poll with `check_job_status` and
  read settled rows with its `fetch` action, or retrieve the whole analysis
  with `get_cross_border_result`. **Rough sizing on the default 20 TPM Keepa
  plan (capacity 1,200 tokens): ~100 source ASINs is the inline ceiling**;
  higher TPM plans push it much higher. Because the quote carries no cache
  discount, a warm cache raises what the order *settles* for, not what it
  needs up front to run inline.
- **The `xb_` id is stable for the order's whole life** and the blob behind
  it is rewritten after every batch, so an id handed over mid-run still
  resolves against whatever the order ends up with. That is why the result
  can be read while it is still incomplete: check `pendingCount` and
  `orderEnded` before drawing conclusions from the match count.
- **Exchange rates** are baked-in approximations suitable for spotting
  arbitrage opportunities. Not suitable for forex or accounting.
  `exchangeRateDate` in the summary indicates when rates were last
  updated.

---

## `resolve_codes`

Bulk UPC/EAN/GTIN/ISBN-13 → candidate-ASIN resolution for supplier
manifests (wholesale price lists, arbitrage CSVs). Accepts up to 500 rows
per call, batches the codes to Keepa, and attributes every returned
product back to your codes by scanning each product's full code list with
GTIN-14 normalization (the UPC-12 and EAN-13 forms of one item collide
correctly).

### What it's good for

- You have a supplier manifest (codes, maybe titles / brands / pack sizes)
  and need the candidate Amazon ASINs for each line.
- Identity only: no buy-box, offers, stock, or rating data. Feed the
  chosen ASINs into `screen_products` / `get_product_details` for that.

### Token cost

- **~1 Keepa token per returned candidate** (typically ≈1 per code, since
  most codes match a single ASIN). `codeLimit` (default 3, max 20) caps
  candidates per code.
- Codes not in Keepa's database cost ~nothing. The order is **quoted at the
  worst case** (`uniqueCodes × codeLimit`) and **settled at Keepa's actual
  charge** as each dispatch commits. There is no whole-call reservation to
  refund.
- There is no cache discount here: the identity tier never writes the
  product cache, and the question a code asks is which ASINs it maps to.

### Inputs

- **`rows`**: up to 500 manifest rows. Each row is a `code` plus optional
  `supplierTitle` / `supplierBrand` / `cost` / `qty` / `packSize`. Pass
  them when you have them: supplier title vs candidate title is the
  strongest disambiguator.
- **`domain`**: the marketplace to resolve on.
- **`codeLimit`**: candidates per code (default 3, max 20).
- **`excludeBrands`**: hard filter; candidates whose brand matches
  (case-insensitive) are dropped before storage.

### What it returns

A **compact summary only**: counts (`rows / resolved / multiCandidate /
notFound / invalid / unattributed`), tokens used, a `codes:` resultSetId,
the notFound code list, and per-row validation errors. **The per-row
candidate table is NOT returned here**: it is cached (24 h) and read via
`get_codes_result`. `multiCandidate` is your action signal: those rows
need disambiguation; single-candidate rows are done.

The `codes:` id is derived from the work order and stable for its whole
life, so an id handed over mid-run still resolves against whatever the order
ends up with. If the stored view expires before the order does, a
`check_job_status` poll rebuilds it for free rather than re-resolving.

When a run stops short, the reply is **two blocks with the note first and
the summary last**, so read the last block for the counts. The note's
opening word tells you whether more is coming: **PARTIAL** and **NOT
STARTED** leave a live order to poll, while **INCOMPLETE** and
**CANCELLED** are terminal and nothing resumes.

### Disambiguation: the tool states facts, you judge

It never auto-picks a winner. Each cached row carries the echoed supplier
inputs plus mechanical **matchSignals** (`candidateCount`,
`primaryCodeMatch`, `qtyMatch`, `brandMatch`) and per-candidate identity
fields (title, brand, full product codes, packageQuantity, BSR, lowest-new
price, `ambiguityFlags`). Compare and choose yourself.

### Notes

- Lenient validation (8–14 digits, formatting stripped). Placeholder /
  junk codes (≥10 identical consecutive digits) are rejected before spend.
- The resultSetId holds the **union of all candidates across rows**: for
  large manifests, disambiguate first and pass the chosen subset
  downstream, not the raw union (it can exceed a 500-ASIN cap).
- A call the balance can't fund is accepted as a background work order
  instead of being refused: poll `check_job_status`, which hands back the
  `codes:` resultSetId once anything settles. Do not re-call `resolve_codes`
  to retry, which pays for the same resolution twice.
- **A zero-candidate row is not proof the code is unknown to Keepa.** A code
  the run never reached renders identically to a genuine miss. The
  `PARTIAL:` line in the summary is what tells the two apart, so confirm
  with `check_job_status` before treating a `notFound` code as dead.

---

## `get_product_chart`

Fetches a PNG chart image from Keepa's `/graphimage` endpoint for a
single ASIN on a single marketplace domain. Renders inline in Claude
Desktop's regular chat and in Claude Code, and the model receives the
same image so it can analyse the chart and answer follow-up questions.

### What it's good for

- The user asks to **see** a price history, BSR trajectory, or Buy Box
  pattern.
- The user wants to **confirm a visual signal** (seasonality, trend
  direction) that a numeric field doesn't capture.
- A specific candidate from `get_product_details` or `screen_products`
  is worth a closer visual look.

Not a batch primitive, it's a visualization tool for specific products.
Don't ask for charts on every product in a result set.

### Token cost

**1 Keepa token reserved per call, flat.** Regardless of curves, image
size, or time range. Keepa caches identical requests **server-side for
90 minutes**: identical repeat requests on the Keepa side are free.

Note: the local token bucket doesn't reconcile against PNG responses
(they carry no `tokensConsumed` field), so the local available-tokens
count drops by 1 per call regardless of Keepa-side cache hits.

### Inputs

- **`asin`**: required. Single ASIN per call. Don't batch-chart a
  whole result set.
- **`domain`**: required. **No default.** If the user hasn't specified
  a marketplace, the assistant has to ask.
- **`rangeDays`**: defaults to 90 (last 90 days). Common values:
  30 (recent), 90 (typical sourcing window), 365 (seasonality).
- **`width` × `height`**: defaults to **800 × 400**.
- **Curve toggles**: see below.

### Curves

Tuned for resellers. **Default ON:** `amazon` (Amazon's own price),
`new` (3rd-party new offer low), `buyBox`, `salesRank` (BSR), `fba`
(lowest FBA offer).

**Default OFF (toggle on per user request):** `used`, `fbm`,
`buyBoxUsed`, `lightningDeals`, `warehouseDeals`, `primeExclusive`.

At least one curve must be enabled. Setting all 11 toggles to false is a
validation error: an empty chart would still cost 1 Keepa token for
nothing.

### What it returns

Every successful call returns the same two-part base:

1. **TextContent**, metadata line: ASIN, domain, range, dimensions,
   curves enabled, and a token-cost note.
2. **ImageContent**: base64-encoded PNG (default 800 × 400). This is the
   channel the model sees (so it can analyse the chart), and it renders
   inline in Claude Code and Claude Desktop chat.

On Claude Desktop (regular chat and Cowork) the server additionally
mounts an MCP Apps view to present the chart. In the ChatGPT desktop
app the chart renders as an inline MCP Apps card when Codex's
`enable_mcp_apps` feature flag is on (the installer offers to set it;
see [INSTALL.md](./INSTALL.md#inline-chart-cards-in-chatgpt-desktop-optional))
and you are signed in to ChatGPT. The Codex CLI terminal does not
render images; the model still receives the chart for analysis.

### Critical notes

- **Cowork renders inline as of v1.7.1.** Earlier releases could not
  display the chart in Cowork (Claude Desktop's agent-mode surface);
  it now arrives as the same MCP Apps view regular chat uses. If
  Cowork still shows text-only charts after an upgrade, quit Claude
  Desktop fully (Cmd-Q) and reopen: Cowork keeps the previous server
  process alive until a full quit.
- **Non-PNG responses are rejected.** If Keepa returns an HTML error
  page with a 200 status, the tool surfaces `KEEPA_ERROR` instead of
  treating the bytes as an image.
- **API key stays server-side.** The URL with your Keepa API key is
  built in-process and never returned. Only rendered image bytes flow
  back.
- **Validation rejects fabricated ASINs** (`B0SNOW0001`-style
  sequences, `B00000001`, keyboard-mashed patterns) before hitting
  Keepa. Use ASINs from prior tool results or explicit user input.

---

## `check_token_balance`

Check Keepa token availability and per-tool cache-aware cost
projections. Free, reads locally cached bucket state and never calls
Keepa.

### What it's good for

- Before running an expensive call (a 500-ASIN screen, a 50-ASIN deep
  dive, a 100-ASIN cross-border), check whether it will answer inline or
  be accepted as background work.
- Find out what fraction of a candidate set is already cached, so you
  can quote an accurate cost to the user instead of the worst-case.
- Get current balance and refill rate (tokens/min) for situational
  awareness.

### Token cost

**0 Keepa tokens.** Local bucket read.

### Inputs (all nullable)

Three call modes:

- **`asins` + `forTool` (+ `domain`)**: **Cache-aware per-tool check.**
  `forTool` is REQUIRED when `asins` is provided (one of
  `screen_products`, `get_product_details`, or `resolve_cross_border`).
  `domain` is REQUIRED in this branch, no silent default to US; if the
  user hasn't specified a marketplace, the assistant has to ask. Returns
  cached vs uncached counts, the actual per-tool cost, and a proceed /
  wait recommendation.
- **`estimatedCost`**: Simple affordability check. Returns whether
  enough tokens are available, or wait time.
- **All null**: Returns current balance and refill rate.

### What it returns

- Current token balance and refill rate (tokens/min).
- When `asins` provided: cached count, uncached count, actual cost,
  proceed/wait recommendation.
- When insufficient: tokens needed, estimated wait time in minutes.

### Per-tool cost reference

| Tool | Cost |
|------|------|
| `execute_keepa_finder` | 10 base + 1 per started 100 ASINs; stats add 30 + 1 per whole completed million of `totalResults` |
| `screen_products` | 3 per uncached ASIN |
| `get_product_details` (ASIN / resultSetId modes) | **16 per uncached ASIN**, the authorised worst case; actuals settle at ~6-8 |
| `resolve_cross_border` | **12 per source ASIN** (3 source + up to 9 target, CODE_LIMIT=3), quoted flat with no cache discount; actuals settle lower |
| `resolve_codes` | ~1 per returned candidate, quoted at `uniqueCodes × codeLimit` (default 3, max 20) |
| `get_product_details` code lookup | 1 per returned candidate, up to `codeLimit` (default 3) |
| `get_product_chart` | 1 flat (cached by Keepa 90 min) |
| `confirm_work_order`, `check_job_status`, `check_token_balance`, `get_finder_result`, `get_cross_border_result`, `get_codes_result` | free (local reads) |

### Timing expectations

- Small lookups (≤20 ASINs) usually complete immediately.
- Medium lookups (50–200 ASINs) may take several minutes.
- Large lookups become background work orders: poll with
  `check_job_status` (`action: 'status'`) and read settled rows with
  `action: 'fetch'`. They advance only while an Agellic session is open.

### Notes

- Cached ASINs cost 0 tokens for single-domain lookups. Cross-border is
  only *partially* cache-aware: a source-cached ASIN saves the 3-token
  source fetch but still owes **up to** 9 for target resolution on the other
  marketplace, so this tool prices it at 9 rather than 0. Both figures are
  ceilings, and `resolve_cross_border` itself quotes the flat 12 per ASIN
  regardless. Read the cross-border number here as *the most a warm cache
  will still cost you*.
- **A cost above the bucket's capacity is not a refusal** for work that
  splits. `screen_products`, `get_product_details`, `resolve_codes` and
  `resolve_cross_border` all become background work orders that fund
  themselves batch by batch, so there is no need to split a list by hand.
  `execute_keepa_finder` is the exception: one query is one indivisible
  Keepa call, so an over-ceiling quote is refused with advice to lower
  `perPage`.
- **A refill rate of 0/min is a hard refusal**, not a wait. It means Keepa
  product tracking is consuming the whole plan's rate, so the tokens can
  never accrue and a full bucket doesn't rescue it. Shrink the tracked
  product list or upgrade the plan.
- **This tool's `Wait ~N minutes` estimate is not an order's real wait.** It
  is balance over refill rate for a hypothetical call right now, and it
  cannot see other work already queued on the key. Once a call has created
  an order, `check_job_status` reports the real, queue-aware bound.

---

## `confirm_work_order`

The consent gate. When a tool quotes work costing more than **60 minutes of
Keepa token refill**, it does not start it: it returns an `orderId`, a
cost-only `Quote:` line, a queue-aware ETA bound, and a one-time
`confirmToken`. Passing the id and the token back here authorises the order.

### What it's good for

- Starting a large screen, lookup, code resolution, or cross-border run that
  came back awaiting consent, once the user has seen the numbers and agreed.

### Token cost

**0 Keepa tokens.** It reads local state and writes a consent record; the
Keepa spend happens later, as the order runs.

### Inputs

- **`orderId`**: the id returned alongside the quote (`wo_<timestamp>_<suffix>`).
- **`confirmToken`**: the one-time token issued with that quote. It is bound
  to that quote, so any re-quote invalidates it.

### What happens

| Order state | Result |
|---|---|
| `quoted`, refill rate steady | Authorised and **queued** for the background scheduler, not started. It runs behind anything already queued. |
| `quoted`, rate dropped >20% since the quote | **Re-quoted.** A new quote and a new token come back in the same response, and the old token is dead. This is a success, not an error: show the new numbers and ask again. |
| `quoted`, rate now 0/min | Refused with an explanation. Your token stays valid, because the rate can recover. |
| `draft` | Error: the manifest was never sealed. Finish staging it. |
| `confirmed` | Error: already authorised, which is not the same as started. Use `check_job_status`. |
| terminal | Error: already finished. Read its rows with `check_job_status` `action: 'fetch'`. |
| wrong or expired token | Error: re-read the current quote with `check_job_status`, which reports the live token. |

### Notes

- **Nothing is charged at confirmation time.** Tokens are spent per batch as
  the order executes, so a partially-run order costs exactly what it
  consumed.
- **Authorised is not started.** Confirming queues the order; the `started:`
  line on `check_job_status` is what tells you a Keepa request has actually
  been dispatched for it.
- **This tool never runs the work inline.** An over-threshold order is by
  definition an hour or more of refill, so confirmation hands it to the
  background scheduler and returns immediately.
- It confirms any kind of work order, so its response carries no rows at
  all. Read the work with `check_job_status`.
- A confirmed order can still be cancelled, and its settled rows stay
  readable.

---

## `check_job_status`

The hub for work orders. Every id it accepts is a `wo_…` id: list them, poll
one, cancel one, or page through the rows one has already settled. The tool
name and the `jobId` input field are historical, kept because hosts have
already learned them, and there is no legacy job queue behind them any more.

### What it's good for

- A prior tool call returned an `orderId`: poll with `action='status'`
  (passing it as `jobId`).
- An order is running and you want the rows it has so far:
  `action='fetch'`, mid-run or after it finishes.
- Survey in-flight or recent work with `action='list'`.
- Stop an order with `action='cancel'`.
- The user mentions an order without providing the id: `list` first to
  discover it, then proceed.

### Token cost

**0 Keepa tokens.** Local read; never calls Keepa. Polling costs nothing and
speeds nothing up.

### Actions

- **`list`**: every recent order, newest first. No `jobId` required.
- **`status`** (default): the progress report for one order, described
  below.
- **`fetch`**: page through the rows an order has already settled, rendered
  the way the creating tool renders them: a screen order as the 13-column
  pipe table, a lookup order as product blocks, a finder order as a one-line
  match count plus its handle, a codes order as per-code candidate counts,
  and a cross-border order as per-source match and gap lines under an
  exchange-rate header. `offset` (default 0) and `limit` (default 100, max
  500) page in the order the rows were requested.
- **`cancel`**: stop an order. One that has **not started yet** cancels
  instantly. A **running** one accepts the request and stops at its next
  dispatch boundary (a Keepa request already in the air is paid for either
  way), and everything it settled stays readable with `fetch`.

### Work-order states

- **`draft`**: still being staged, so not priced and not started.
- **`quoted`**: priced and waiting on consent. Nothing charged. The status
  response carries the live `confirmToken` for `confirm_work_order`.
- **`confirmed`**: authorised, which is not the same as started. It is
  queued for the background scheduler and may sit behind other authorised
  orders before its first Keepa request.
- **`completed`**: every row settled.
- **`partial`**: stopped with rows unfetched (a spend ceiling, a deadline,
  an exhausted retry budget, or repeated upstream refusals) with everything
  it did settle intact. Read them with `fetch`.
- **`failed`**: stopped before settling anything.
- **`cancelled`**: stopped on request, keeping its settled rows the same way
  a `partial` order does.

### What `status` reports

- **Progress**: rows settled vs total, with the not-found / unreadable
  split.
- **Tokens**: what Keepa itself billed, separated from what is reserved for
  requests still in flight. That reserve releases when they settle, so the
  charged total is a high-water mark mid-run.
- **The original quote**, cost only, with no time claim attached, plus a
  **`DRIFT`** note when re-pricing the remaining work now needs more refill
  than the whole order was quoted at (cached entries expire, refill rates
  move). The quote is never silently rewritten, and a declared
  `maxTokenSpend` still binds hard.
- **`authorised:` and `started:`**, which are different events. The first is
  when the user agreed to the spend; the second says only whether a Keepa
  request has actually been dispatched. An order still queued reports
  `not yet started — N orders ahead (~X tokens)`.
- **A `Queue:`/`ETA:` pair** while rows are still outstanding: how many
  orders are ahead with their remaining cost, then a bound on the wait.
  See [Cost and time are two separate claims](#cost-and-time-are-two-separate-claims)
  for the three conditions that ride with it.
- **A hand-off** naming the id to read the results with:
  `screen:<orderId>`, `lookup:<orderId>`, `finder:<orderId>`,
  `codes:<orderId>`, or `xb_<suffix>` for cross-border.

**The ETA is withheld** for four kinds of order, because none of them
finishes on refill alone and a completion time would contradict the message
it sits in: a `draft`, one that has been asked to cancel, one that is
unschedulable, and one whose deadline has passed. For an order still
awaiting consent, the pair is kept but framed "If you confirm now:" and
counts every authorised order ahead of it.

### Notes

- Work orders are created automatically by `screen_products`,
  `get_product_details`, `execute_keepa_finder`, `resolve_cross_border` and
  `resolve_codes` on every call they accept. A short balance comes back as
  an `orderId`, not an error.
- **Finished orders are kept for 14 days**, and a result-set view whose own
  24-hour TTL lapsed is rebuilt on demand at 0 tokens. Unsealed drafts are
  discarded after 24 hours.
- Do not re-issue the original call to "retry" a background order: that
  creates a second order for the same work and pays for it twice.
- An order waiting on refill holds no tokens at all, and one still awaiting
  consent has been charged nothing.

---

## `get_finder_result`

Fetch a page of ASINs from a stored finder result set by id. Pairs with
`execute_keepa_finder` (which emits the id). Use `offset` and `limit` to
page through the full set without re-running the finder query.

### What it's good for

- After `execute_keepa_finder` returns a complete result set, retrieve
  the ASINs.
- After `check_job_status` reports a background finder order finished and
  names its `finder:<orderId>` handle. Same id, same read.
- Page through a large result set, or take a top-N slice. The slice can
  be passed to whichever downstream tool the user has chosen.
- The user asks for "next page" of results from a previous finder
  search.

### Token cost

**0 Keepa tokens.** Local read against the result-set cache.

### Inputs

- **`resultSetId`**: the `finder:<orderId>` id from a prior
  `execute_keepa_finder` call or a `check_job_status` hand-off.
- **`offset`**: defaults to 0. The starting index into the stored set.
- **`limit`**: defaults to 100. Number of ASINs to return.

### What it returns

A single human-readable text block:

- **Header line**: `Finder result <id> (domain <d>): returning K ASINs
  [offset A..B] of N stored (M total matches).` Carries: id echo, stored
  domain, count in this page, offset range, total count stored, and
  total finder matches.
- **ASIN line**: `ASINs: B001,B002,...` (comma-separated), or `ASINs:
  <none in range>` when `offset` is past the end.

### Notes

- **Result sets expire after 24 hours, but the view is derived, not the
  record.** On "not found or expired", poll the order the id names with
  `check_job_status`: a retained order (14 days) rebuilds its view for free.
  Re-running `execute_keepa_finder` is the fallback once the order itself is
  gone, and it pays for the search a second time.
- **Pagination bounds.** If `offset >= totalResults`, the tool returns
  the header + `ASINs: <none in range>`, success with empty range, not
  an error.
- **Kind mismatch.** Only finder result sets work here. Passing a
  `screen_products` or `cross_border` id returns an error pointing at
  the correct tool.
- **Malformed stored set (rare).** If a stored finder set lacks a
  recorded domain, the tool returns a "malformed: no domain recorded"
  error and the assistant re-runs the finder query.

---

## `get_cross_border_result`

Retrieve a cached cross-border analysis by id (`xb_...`), or list recent
analyses for this install. Pairs with `resolve_cross_border` (which
emits the id).

### What it's good for

- After `resolve_cross_border` has run and its inline output has been
  compacted out of the conversation, use `get` with the `xb_` id to
  recover the full result.
- After a background `resolve_cross_border` work order has settled
  anything: `check_job_status` hands back the `xb_` id, and it stays
  readable here for the rest of the order's life and after it completes.
- Redisplay or re-analyze a previous run: use `list` to discover
  available ids, then `get` to fetch.

### Token cost

**0 Keepa tokens.** Local store lookup; no Keepa API calls.

### Actions

- **`get`** (default): looks up a specific `xb_` result in the local store
  and returns the full cross-border analysis.
- **`list`**: scans the local store for all cross-border analyses held
  under this install's Keepa key hash. No id required.

### What it returns

**`action='get'`**: a **JSON-encoded text payload** carrying the full
stored `CrossBorderResult`. The text is `JSON.stringify(result)`; parse
for field access. The parsed object contains:

- `summary`: counts (matched / unmatched), avg gap %, source and
  target domain labels, exchange rate and date, the run's actual
  `tokensUsed`, plus `pendingCount` and `orderEnded` when relevant (below).
- `matches`: array of source↔target ASIN pairs with prices (local
  cents), converted target price, gap %, and confidence level.
- `unmatched`: source ASINs that could not be resolved, with reason
  codes.

### Reading an incomplete result

The `xb_` blob is rewritten after every batch settles, so an id can be read
before its order is done. What comes back then is a real but **incomplete**
comparison, and two fields keep you from misreading it:

- `summary.pendingCount` counts source ASINs that are **not resolved**.
  Those have not been looked at, so reporting "1 of 5 matched" as though the
  other four found no arbitrage is exactly the misread these fields prevent.
- `summary.orderEnded` is `true` only once the order has reached a terminal
  state, which makes its pending rows final.

| | `orderEnded` absent | `orderEnded: true` |
|---|---|---|
| What it means | still filling in | ended short, those rows were never resolved |
| `action='list'` label | `— STILL RUNNING, N of M not resolved yet` | `— ENDED INCOMPLETE, N of M never resolved` |
| What to do | poll `check_job_status`, re-read here | stop polling; re-run those sources if you still want them |

**`action='list'`**: a human-readable text block, newest first. Header
line counts cached analyses; one line per analysis carries id, source →
target marketplace codes, matched/total count, average price gap, and
the best individual match. Example line:

```
xb_a1b2c3: CA → US: 80/81 matched, avg gap +8.1% (best: B0BESTSRC1 → B0BESTTGT1 +22.3%)
```

### Notes

- **7-day TTL.** After that the record is evicted and `get` returns a
  not-found error. Re-run `resolve_cross_border` to regenerate.
- **Missing or expired id** returns a clear error pointing at
  `action='list'` for discovery and `resolve_cross_border` to create a
  new analysis.
- **Id is opaque.** The `xb_` id is a random token, cannot be guessed.
  Always obtain it from a prior `resolve_cross_border` response or via
  `action='list'`.

---

## `get_codes_result`

The sole reader of cached `resolve_codes` resolutions. `resolve_codes`
returns only a summary; the per-row candidate table lives here: fetch
exactly the rows you need instead of receiving the whole table.

### What it's good for

- After `resolve_codes`, pull the rows that need disambiguation
  (`multiCandidateOnly: true`, the canonical move; single-candidate rows
  are already resolved).
- After a background `resolve_codes` work order has settled anything:
  `check_job_status` hands back the `codes:` id, and it stays readable here
  for the rest of the order's life and after it completes.
- Fetch specific rows by supplier code, or page through the full set.
- Discover cached resolutions (`action: 'list'`).

### Token cost

**0 Keepa tokens.** Local cache read. The stored view expires after 24 h,
but the work order behind it does not, so an expired id is usually
recoverable for free: poll `check_job_status` on the order (its id is the
`codes:` id minus the prefix) and the view is rebuilt from the order's own
record at 0 tokens. Re-run `resolve_codes` only once the order itself has
aged out, since that pays for the whole resolution again.

### Actions

- **`get`** (default): page through one resolution's rows. Requires `id`
  (`codes:...`).
- **`list`**: enumerate cached resolutions for this install, newest
  first, with summary counts. No id required.

### Inputs (get)

- **`id`**: the `codes:` id from `resolve_codes`.
- **`multiCandidateOnly: true`**, return only rows with >1 candidate.
- **`codes: [...]`**, fetch specific rows by supplier code (formatting
  and zero-pad variants are tolerated).
- **`offset` / `limit`**: row pagination over the filtered view (default
  50 rows/page).

### What it returns

Each row renders as a header line (the supplier code, the mechanical
matchSignals, the echoed supplier inputs, and an `excludedByBrand` count
when the filter dropped candidates) followed by a pipe table of
candidates: ASIN, title, brand, packageQuantity, items, part, model, BSR,
lowest-new price (cents), ambiguity flags (`variation` / `multipack` /
`inactive_listing`), and the primary-code match. Zero-candidate rows
render `(no candidates)`.

### Notes

- The price column is the current lowest-new price in cents, identity
  tier, so no buy-box data exists in this table by design.
- Compare the supplier title / brand / pack size in the header against
  each candidate and pick the ASIN yourself; the tool never auto-picks.
  Wildly heterogeneous candidates on one row signal a junk supplier code.
