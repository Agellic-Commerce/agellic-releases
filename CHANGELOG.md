# Changelog

All notable changes to agellic-mcp are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [2.0.0] - 2026-08-11

Every Keepa-calling tool call is now a durable work order. Work that
used to wait invisibly (or fail) when tokens ran short now accepts,
survives restarts, runs itself as tokens refill, and reports honest
progress and an honest wait. Nothing expensive starts without your
consent. Your existing v1.0.0 license token works as-is, no reissue
needed.

### Added

- **`confirm_work_order`, the consent gate.** Work quoted above roughly
  one hour of token refill does not start. The call returns the order
  id, a cost-only quote, a queue-aware ETA, and a one-time confirm
  token; nothing is charged until you agree and the assistant passes
  the token back. Confirming queues the order behind anything already
  authorised (authorised is not started, and the status view tells you
  which is true).
- **Staged screen manifests.** `screen_products` now builds manifests
  larger than one call's 500-ASIN cap across staging calls, then seals
  and quotes the whole thing at once.
- **Spend ceilings and deadlines.** `maxTokenSpend` puts a hard cap on
  what a screen may ever charge. `deadlineMinutes` gives up that many
  minutes after the order is authorised (the clock runs while it waits
  in the queue), keeping whatever settled.
- **Mid-run row access.** `check_job_status` action `fetch` pages
  through rows an order has already settled, mid-run or after it
  finishes.

### Changed

- **Every call returns one of three answers.** The result right away
  when the token balance covers the quote; a background acceptance when
  it does not (the order then runs itself as tokens refill); or a quote
  plus confirm token when the work is big enough to need consent. All
  three name the order id and the result-set handle your next call can
  reuse.
- **Cost and time are two separate claims.** Acceptances state the flat
  token cost on one line and the wait on another: a bound ("done within
  ~X of token refill"), recomputed on every poll so it counts down as
  the queue drains. The bound covers the work as currently quoted,
  assumes no other work on your Keepa key, and its wall-clock figure
  assumes a session open about 8 h/day (an always-open session finishes
  about 3x sooner).
- **`check_job_status` is the hub.** Modes list, status, cancel, and
  fetch. Status separates `authorised:` (you agreed) from `started:`
  (a Keepa request has actually been dispatched), and withholds the
  ETA for orders that will not finish on refill alone: drafts,
  cancelling orders, unschedulable orders, and orders whose deadline
  has passed. An order awaiting consent shows its ETA conditionally,
  prefixed "If you confirm now:".
- **Charges settle at Keepa's own figure.** Quotes are worst-case
  ceilings, and most orders settle under them. Mid-run, the charged
  total can include tokens reserved for requests still in flight;
  those release when the request settles.
- **Cancel keeps what you paid for.** Cancelling a not-yet-started
  order stops it outright at zero charge. Cancelling a running order
  stops it at its next dispatch boundary, and everything settled so
  far stays readable.

### Removed

- **The rate-limit-wait job queue.** Nothing queues invisibly anymore.
  Anything big enough to wait is a work order you can see, poll, and
  cancel, and it survives a session restart.

### Upgrade notes

- CLI installs (Claude Code, Codex) clear the local product cache on
  upgrade, so previously cached products refetch once at their normal
  token price. Claude Desktop (.mcpb) upgrades keep the cache.
- Background work only advances while an Agellic session is open.

## [1.8.0] - 2026-08-01

Buy Box prices now include shipping. Every Buy Box number this server
reports is the LANDED price (item + shipping), which is what a buyer
actually pays and what Amazon charges its referral fee on. Expect a
one-time value shift on shipped items: yesterday's $10.99 may read
$15.98 today with no offer change at all. The item price did not move,
the label got honest. Items with free shipping are unchanged. Your
existing v1.0.0 license token works as-is, no reissue needed.

### Changed

- **Buy Box prices are landed (item + shipping) on every surface.**
  This is Keepa's native `csv[18]` series, which carries shipping across
  the full 1095 days of history, so this lane has no cutover date and no
  mixed-convention window. The change reaches `pricing.buyBox.*`
  (current, the 30/90/180/365d averages, and the 365d min/max),
  `competition.buyBox.priceCents`, the `screen_products` `BB(c)` column
  and its `Trend`, the Buy Box lane of the sell-price read, the price
  position insight, and the finder's `avgBuyBox` insight.
- **Re-baseline any saved Buy Box thresholds.** A price filter, ROI
  target, or alert level tuned against pre-1.8.0 item-only Buy Box
  numbers is now being compared against a larger number on every shipped
  item. This is the one thing worth checking after you upgrade.
- **Offers sort by landed price.** The offer list in
  `get_product_details` now orders by what you would actually pay, so
  the offer at the top is the cheapest to receive rather than the one
  with the lowest sticker and a shipping charge hidden behind it.
- **Finder price filters were always landed, and now say so.**
  `current_BUY_BOX_SHIPPING`, `delta30/90_BUY_BOX_SHIPPING`, and
  `buyBoxStandardDeviation30/90/365` are Keepa fields that have always
  included shipping (the `_SHIPPING` suffix is the tell). Only the
  wording changed here, no filter behavior did.

### Added

- **`pricing.buyBox.shippingBasis: 'landed'`** marks the new contract.
  Its absence marks a stale pre-landed payload, so anything reading
  these fields programmatically can tell the two apart without guessing.
- **Reconciliation components.** `pricing.buyBox.itemCents` and
  `pricing.buyBox.shippingCents` show exactly how a landed price splits.
  They appear only when shipping is greater than zero, so free-shipping
  items carry neither.
- **Basis disclosure on the read surfaces.** `pricing.sellPrice` and the
  price position insight now declare a `shippingBasis` of `landed`,
  `item-only`, or `mixed-window`. The last one means the window straddles
  the date Keepa started including shipping in its lowest-new series, so
  the prices inside it mix both conventions and are not comparable to
  each other.
- **`pricing.sellPrice.marketState: 'suppressed'`** on listings whose
  Buy Box is suppressed. A normal market has no state worth reporting,
  so the field is present only when there is something to disclose.

### Fixed

- **The lowest-new shipping-inclusion date was a week early.** Keepa
  began including shipping in its lowest-new series on 2026-02-23, not
  2026-02-16. The earlier date comes from a comment bug in Keepa's own
  Java client. Reads whose window crosses that boundary now caveat
  against the right instant.
- **Honest wording when a listing only ever held two prices.** The
  time-at-price hero used to print the entire coverage window as though
  a single price had held for all of it. It now reports the dominant
  price with its real dwell time, or the range across both levels.
- **Chart captions scope the landed claim to the Buy Box curve.** The
  new-offer curve is item-only before 2026-02-23, and the caption now
  says so instead of implying every curve on the chart is landed.

## [1.7.1] - 2026-07-31

This release puts the price/BSR chart in front of you on two more
surfaces: the ChatGPT desktop app (as an inline card, opt-in via the
installer) and Cowork, which renders charts inline for the first time.
Your existing v1.0.0 license token works as-is, no reissue needed.

### Added

- **Inline chart cards in the ChatGPT desktop app.** `get_product_chart`
  now renders as an MCP Apps card in the conversation when Codex's
  `enable_mcp_apps` feature flag is on and you are signed in to ChatGPT.
  Without the flag the chart still reaches the model; ChatGPT just shows
  it inside the tool-call expander.
- **The codex installer offers that flag, never flips it silently.**
  Interactive `--host codex` installs and upgrades ask (default No);
  non-interactive runs only act on the new `--enable-mcp-apps` flag. The
  enable goes through `codex features enable` (Codex's own config
  editor; your `config.toml` is never hand-edited), and a failure there
  never fails the install. An explicit `enable_mcp_apps = false` or
  `apps = false` in your config is always respected, and uninstall
  leaves the flag alone. Expect a harmless "under-development features"
  notice at codex startup once enabled. See
  [INSTALL.md](./INSTALL.md#inline-chart-cards-in-chatgpt-desktop-optional).

### Fixed

- **Cowork renders charts inline.** Charts now arrive in Cowork (Claude
  Desktop's agent-mode surface) as the same MCP Apps view regular chat
  uses; earlier releases could only deliver a text readout there. Chart
  display in Claude Desktop chat also survives a recent Claude Desktop
  update that changed how tool results reach the chart view. If Cowork
  still shows text-only charts after upgrading, quit Claude Desktop
  fully (Cmd-Q) and reopen: Cowork keeps the previous server process
  alive until a full quit.

### Changed

- **Chart summaries no longer include a Keepa browser URL line.** The
  inline render (or the tool-call expander on surfaces without one) is
  the chart channel; the URL line was a leftover fallback.

## [1.7.0] - 2026-07-29

This release answers a question agellic could not answer before, "what can
I actually sell this for?", and rebuilds the seasonality detector so that a
confirmed season means the peak genuinely recurred across years. Your
existing v1.0.0 license token works as-is, no reissue needed.

### Added

- **Sell-price read in `get_product_details`.** Every product now carries
  `pricing.sellPrice`: observed sale-price bands (`moveFastCents` /
  `marketCents` / `stretchCents`, the 25th / 50th / 75th percentiles of the
  prices in force at inferred sale moments), the current price's position
  inside them, sales skew, 30-day drift, and honest caveats. Three methods
  ladder from transaction-quality on down: Buy Box prices at sale events,
  the lowest-new floor at sale events when the Buy Box is suppressed, and
  duration-weighted time-at-price when sale events are too thin; below
  that, plain window averages. The bands describe the observed market,
  never a pricing recommendation. See
  [COMPUTED-INSIGHTS.md §3](./COMPUTED-INSIGHTS.md#3-sell-price-read).
- **`insights.recentDemandDeviation`.** The trailing-year average rank
  divided by the trailing-30-day average: a one-number read on whether
  demand is currently running above or below the product's own baseline.

### Changed

- **Seasonality detection rebuilt around recurrence.** A seasonal peak is
  now `confirmed` only when the same peak window recurred across at least
  two separate years of history AND beat a statistical null test that
  rejects slow rank drift masquerading as seasonality. A single observed
  season reports as `candidate` with explicit "do not act on this alone"
  framing. Detection itself moved to a scale-free cluster split of the
  weekly rank profile. Deliberately conservative; verified against two
  blind holdout sets with zero false confirmations. See the rewritten
  [COMPUTED-INSIGHTS.md §2](./COMPUTED-INSIGHTS.md#2-seasonality).
- **Three years of history for the deep read.** `get_product_details` now
  fetches 1095 days of history (was 365), giving seasonality confirmation
  up to three full cycles to work with; per-ASIN token cost is unchanged.
  Seasonality computed from shorter screening windows (under 730 days) is
  explicitly capped at `candidate` and its summary says why.
- **Honest calendar labels.** A peak is labeled Q4 / Summer / Halloween
  etc. only when it genuinely matches the calendar template: the overlap
  bar tightened, plus a new containment path for a narrow peak sitting
  inside a wide season window. Otherwise the label is `Other` and the peak
  week range carries the identity.
- **The product cache resets on this upgrade** (new cache format for the
  three-year window): previously cached ASINs refetch at the normal token
  cost on first read. Credentials, token state, and logs are untouched.

## [1.6.0] - 2026-07-20

Agellic now installs into Codex CLI and the ChatGPT desktop app alongside
Claude Desktop and Claude Code: one command registers both Codex surfaces,
your credentials never enter Codex configuration, and every host on the
machine shares one credential cache, one data dir, and one job queue. Your
existing v1.0.0 license token works as-is, no reissue needed.

### Added

- **Codex CLI + ChatGPT desktop as installable hosts.** `node install.mjs
  --host codex` installs a Codex-owned copy of the server, probes it with a
  real MCP round-trip before touching any configuration, and registers it
  through the `codex` CLI. Codex CLI and ChatGPT desktop share MCP
  configuration, so the one command covers both. See
  [INSTALL.md](./INSTALL.md) for the walkthrough.
- **Credential-free Codex configuration.** Your license and Keepa key are
  written only to the per-machine credential cache; the Codex entry itself
  carries no secrets, and the server reads the cache at boot.
- **Promptless second-host install.** If any host already configured agellic
  on this machine, `node install.mjs --host codex --non-interactive`
  completes with no prompts, entirely off the shared cache.
- **One-command upgrades across hosts.** `node install.mjs --host all
  --upgrade` detects which scripted hosts are installed (Claude Code, Codex)
  and upgrades each in turn with a per-host report.
- **Per-host uninstall with a safe purge order.** Every uninstall level
  (config only, config + binary, full purge) now works per host, and the
  full purge refuses while another host is still installed, naming it, so
  you can't corrupt a live host's shared data. INSTALL.md documents the
  order that always works without `--force`.
- **A graceful path when the `codex` CLI is missing.** The installer still
  installs and probes the server and writes the credential cache, then
  prints two remedies: install the CLI and re-run (always safe, the
  registration is idempotent), or add the server manually in ChatGPT
  desktop settings.

## [1.5.0] - 2026-07-04

Accurate token pricing from the very first call, token accounting that tracks
Keepa's real refill rate, and clearer guidance when a batch is too big or your
budget has run dry. Your existing v1.0.0 license token works as-is, no reissue
needed.

### Changed

- **First-call token pricing is now accurate from a cold start.** At startup the
  server runs a free check of your Keepa token balance and aligns its local
  budget to it, so the very first cost estimate and funding decision match what
  Keepa will actually charge instead of assuming a full budget. The check runs
  after the connection is live, so it never delays startup.
- **Token accounting tracks Keepa's real refill rate.** The local token budget
  now refills and reconciles at the effective rate your Keepa key actually earns
  each minute, so cost estimates and wait times stay in step with what Keepa
  does rather than drifting optimistic.
- **Oversized batches and an empty budget now give you a next step.** When a
  request is larger than a tool's per-call cap, the message names the specific
  tool and limit to fall back to for that kind of request. When your token rate
  is effectively zero, the message names the reconnect step that recovers it
  instead of leaving you to wait indefinitely.

### Fixed

- **The balance estimator stops advising waits that can never finish.** A cost
  estimate for a batch bigger than your budget could ever hold used to suggest
  "wait N minutes," a wait that would never come true. It now recognises an
  impossible-to-fund request and tells you to split it instead.
- **The startup balance probe is hardened.** It follows a strict internal event
  contract and can never block or fail a boot, so it stays a quiet background
  accuracy step rather than a startup risk.

## [1.4.0] - 2026-06-25

Background jobs you can see into and cancel, token costs that account for what's
already cached, cross-border comparisons that price in your cache before
charging, and cleaner upgrades. Your existing v1.0.0 license token works as-is,
no reissue needed.

### Added

- **The background-job queue is now visible.** When a call is rate-limited and
  queued, `check_job_status` shows the job's position in line, the token balance
  it's waiting to reach before it can start, and an ETA based on your Keepa
  refill rate, so a queued screen or lookup is no longer a black box.
- **You can cancel a queued job before it starts.** `check_job_status` with
  `action: "cancel"` abandons a pending job that hasn't begun. A job that has
  already started finishes on its own.

### Changed

- **Token costs now subtract what's already cached.** Cost checks
  (`check_token_balance`) and the job funding gate price in the ASINs you've
  already pulled, so a re-run that's mostly cached leases immediately instead of
  waiting to afford the full uncached price. Cached re-reads stay free.
- **Cross-border comparisons are cache-aware.** `resolve_cross_border` counts
  cached source products in its affordability check and ETA and only charges for
  the uncached ones, so a partially-cached batch gets an accurate, lower cost
  estimate instead of being quoted, or made to wait for, the full price.
- **Duplicate requests collapse onto the one already in flight.** Submitting an
  identical request while a matching job is still queued or running now reuses
  that job instead of minting a second one that would double-spend tokens.
- **A cancelled job reads as cancelled, not failed.** `check_job_status` now
  distinguishes a job you cancelled from one that errored out.
- **Oversized batches get a concrete split size.** When a request is too large to
  ever fund within your token budget, the message now tells you the exact maximum
  number of ASINs to split it into instead of a vague "too big."

### Fixed

- **Token accounting self-heals after a rate-limit.** On a real Keepa 429, the
  local token tracking reconciles against Keepa's actual rate-limit headers, so
  it can't drift out of sync and make you wait when you don't need to.
- **Upgrades clear cached results cleanly.** Re-running the installer with
  `--upgrade` now clears cached lookups, screens, finder and cross-border results
  and the job queue (your credentials, rate-limit state and saved exchange rates
  are kept). A result id from before an upgrade now re-runs cleanly instead of
  silently re-charging against a stale cache.

## [1.3.0] - 2026-06-23

Sharper demand reads for products that only map to a broad department, plus a
finder guardrail against wrong-marketplace category ids. Your existing v1.0.0
license token works as-is, no reissue needed.

### Changed

- **Demand reads now anchor to the specific subcategory, not the whole
  department.** When a product's sales-rank data only places it in a top-level
  department (e.g. "Toys & Games"), the demand estimate used to blend that entire
  department. It now follows the product's own category breadcrumb down to the
  most specific matching subcategory (e.g. "Action Figures"), giving a tighter,
  more accurate range. Products that already mapped to a specific subcategory
  read the same.
- **The product finder catches wrong-marketplace category ids before spending
  tokens.** `execute_keepa_finder` rejects a category id that belongs to a
  different Amazon marketplace than the one you're searching, a mistake that
  used to slip through and return a plausible-but-wrong match count. It checks
  every category field (include, exclude, and the sales-rank reference), and the
  rejection costs zero tokens.

## [1.2.0] - 2026-06-21

Sharpens demand reads that come from velocity (rank-drops or reviews). Your
existing v1.0.0 license token works as-is, no reissue needed.

### Changed

- **Velocity-based demand estimates are no longer understated.** When a demand
  read is built from rank-drop or review velocity rather than a direct category
  match, the per-category conversion rates were anchored to Keepa's censored
  "50+/mo" badge floor, which quietly pushed those estimates too low, most of
  all for the ~40% of products sitting right at that floor. The rates now use
  the middle of each Keepa badge band instead, lifting typical velocity-based
  estimates by roughly 1.3–1.4×. Direct category-table reads are unchanged, so
  most products read the same; the lift shows up on low-volume and
  thinner-category products.

## [1.1.0] - 2026-06-21

Corrects the demand read for low-volume products. Your existing v1.0.0
license token works as-is, no reissue needed.

### Changed

- **Demand reads now reflect real volume below 50/mo.** The v1.0.0 engine
  floored every estimate at 50 units/month, so a product that moves only a few
  units a month could read "50–100." Demand is now a two-sided range built from
  rank-drop and review velocity and can read below 50 (e.g. "~3"), with
  confidence scaled to how well the two signals agree. A reported Amazon
  "X+ bought" badge is still treated as truth and shown as-is.
- **New "likely stalled" read** for a product with recent reviews but no rank
  movement (reviews lag sales, so it's most likely stale past demand) instead of
  inventing a current number.
- **`--upgrade` now refreshes the product cache** (Claude Code), so the new
  demand math takes effect on upgrade with no manual cache step.

### Fixed

- The sub-50 demand floor that could overstate slow sellers, the costly
  direction of error for a sourcing decision.

## [1.0.0] - 2026-06-19

First stable release. Graduates the `0.5.0-beta.*` series: the beta
designation is dropped from the version, docs, and license (the
distribution model is unchanged: a free, invited, revocable license).
Highlights since `0.5.0-beta.4`:

### Added

- **Two new tools, `resolve_codes` and `get_codes_result`**, bulk
  resolve UPC / EAN / GTIN codes to candidate ASINs from a supplier
  manifest (up to 500 rows), with a cached per-row candidate table you
  page through. The tool surface grows from **9 to 11**.
- **Inline Keepa charts in Claude Desktop chat and Claude Code.**
  `get_product_chart` now renders the price/BSR chart inline, and the
  model receives the same image so it can analyse the chart and answer
  follow-ups. (Cowork still shows text + URL only: see TROUBLESHOOTING.)
- **Cost receipts + category echo** on `execute_keepa_finder` and the
  cross-border tools: each result reports the Keepa tokens it consumed.
- **`screen_products` Title + Brand columns** in the screen output.

### Changed

- **Demand model recalibrated**: a reported demand badge is treated as
  truth, a new engine replaces the older heuristic, and cross-marketplace
  fallbacks demote to a no-read rather than guessing.
- **`get_product_details` output ~50% smaller** with no loss of signal.

### Fixed

- Keepa token over-refund (a billing-accuracy bug), job-queue durability,
  and a broad correctness + hardening pass across demand, insights, and
  analytics.

## [0.5.0-beta.4] - 2026-05-25

### Added

- **On-demand FX rate refresh for `resolve_cross_border`.** New installer
  mode `node install.mjs --refresh-rates` (Claude Code only) fetches
  current ECB reference rates from Frankfurter and writes a per-machine
  override at `~/.agellic-mcp/exchange-rates.json`. The server resolves
  rates **cache-over-bundled**: the override wins when present and valid;
  a missing / corrupt / partial cache falls back to the baked-in defaults
  silently. `summary.exchangeRateDate` reflects whichever source is live.
  The override is shared via `~/.agellic-mcp/`, so a Claude Code refresh
  also applies to Claude Desktop on the same machine. Claude Desktop-only
  installs have no terminal step and stay on the bundled defaults until
  the next release.

### Changed

- **Refreshed the bundled exchange-rate defaults to 2026-05-22 ECB rates**
  (were 2026-04-09), the zero-config fallback used when no
  `exchange-rates.json` override is present.

### Fixed

- **Claude Desktop now inherits the per-machine Keepa tokens-per-minute
  instead of silently defaulting to 20.** The `.mcpb` manifest gave the
  tokens-per-minute field a `default: 20`, so Claude Desktop always injected
  `20` even when the field was left blank, overriding the per-machine value
  configured elsewhere (e.g. `60` from Claude Code) and throttling CD-only
  users on higher Keepa tiers to the 20-tier rate. The field now matches the
  license/key fields: leave it blank to inherit from the shared credential
  cache, or enter a value to set it explicitly.
- **Corrected 5 mislabeled `screen_products` column descriptions** (the
  Core-10 column labels) in TOOLS.md.
- **The server now reports its real version.** Its internal version string
  was a hand-maintained literal that had never been bumped past
  `0.5.0-beta.1`, so the startup log and the version handed to Claude
  Desktop / Claude Code identified the server as `beta.1`. It is now derived
  from the package version at build time and can no longer drift. No change
  to behavior: this only corrects the version shown in logs and used for
  support diagnostics. (The `beta.4` download was re-cut to include this fix;
  re-download if you installed before 2026-05-25.)

## [0.5.0-beta.3] - 2026-05-23

### Fixed

- **Claude Code installer's credential-cache write step decoded the
  license token as a 3-part JWT.** Agellic licenses are a 2-part
  `<base64url-payload>.<base64url-signature>` envelope (ed25519, not
  JWT: see `src/license/verify.ts`). The decode in beta.2 rejected
  every real license with `malformed license token (expected 3 JWT
  parts)` and skipped the cache write, defeating the headline fix
  from beta.2 (the CC-first → CD-second credential handoff). The
  decode now matches the real format and includes a whitespace-strip
  pass for parity with the server's verifier.

## [0.5.0-beta.2] - 2026-05-23

### Fixed

- **Claude Code installer now prompts for TPM interactively.** Was
  silently defaulting to 20 even in interactive mode when no `--tpm`
  flag, existing config entry, or credential cache supplied a value.
  INSTALL.md's promise of an interactive TPM prompt now matches
  behavior. Workaround on beta.1 was passing `--tpm <value>` on the
  command line.
- **Claude Code installer writes the credential cache after a
  successful probe.** Previously the per-machine cache at
  `~/.agellic-mcp/credentials.json` was only populated by an actual
  server boot (CD spawn, or a tool call in CC). That meant the "leave
  fields blank, cache fills in" UX in Claude Desktop only worked when
  CD was installed first. The installer now writes the cache directly
  after the probe validates credentials, so CC-first → CD-second picks
  up credentials from cache too.

## [0.5.0-beta.1] - 2026-05-22

### Added

- **Public release distribution.** Pre-built `.mcpb` (Claude Desktop) and
  `.zip` (Claude Code) artifacts now ship from
  [Agellic-Commerce/agellic-releases](https://github.com/Agellic-Commerce/agellic-releases).
  Both files are byte-identical; just two filenames so each host's unzip
  tool recognises the extension.
- **Bundled Claude Code installer.** `install.mjs` ships inside the
  release artifact at root. After `unzip`, run `node install.mjs`: no
  source checkout, no `pnpm install`. Auto-discovers the sibling
  `./server/server.js` bundled in the same artifact.

### Fixed

- **Claude Desktop first-install red banner.** Brand-new testers dragging
  the `.mcpb` in for the first time no longer see "could not find a valid
  license" before they've had a chance to enter credentials. The server
  detects first-install state (no env creds AND no on-disk credential
  cache) and boots a placeholder MCP server with one tool
  (`_configure_agellic`). When credentials are saved, Claude Desktop's
  auto-reconnect respawns the extension into normal full-tool mode.

### Internal

- `resolveCredentials` returns a discriminated union
  `ResolveResult = { kind: 'resolved', credentials } | { kind: 'configuration-pending', reason: 'first-install' }`.
  Configured-but-broken paths (`NoCredentialsError`,
  `LicenseBootError`) propagate as today.

## [0.4.0] - 2026-05-22

### Added

- **Persistent offers across the cache boundary.** Cached
  `get_product_details` responses now include the `## offers (N=…)`
  table, matching fresh-fetch output. `CachedProductData` gains an
  optional `offers?` slot.
- **Hourly cache eviction sweep.** `ProductCache.pruneExpired` is now
  wired into a startup microtask + an unref'd hourly `setInterval` in
  `server.ts`. Expired entries are removed from disk, not just hidden at
  read time.

### Fixed

- **OOS products no longer force-miss the cache.** `hasOffers` semantics
  changed from "got non-empty offers" to "we requested offers": an
  empty offers array now satisfies subsequent `needsOffers` requests,
  saving 9 Keepa tokens per OOS lookup.

### Internal

- `format-lookup` output compression: insights inline, buyBox.rotation
  + offers as pipe tables.
- `vendored/` → `core/` rename; `get-product-details` +
  `resolve-cross-border` refactored into folder modules.
- `SERVER_VERSION` resync (0.3.0 → 0.4.0; was lagging behind package.json).

## [0.3.1]

- `check_job_status` chain-priming: finder branch routed through shared
  formatter; cross-border gained match summary.

## [0.3.0]

- (earlier; pre-changelog cutover.)

## [0.2.0] - 2026-05-15

### Added

- Credential cache model: env-first / cache-fallback resolution with a
  per-machine cache at `~/.agellic-mcp/credentials.json` (mode 0600).
  Second host on the same machine can leave credential fields blank
  and pick everything up from the cache.

## [0.1.0]

- Initial closed-beta release.
