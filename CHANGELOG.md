# Changelog

All notable changes to agellic-mcp are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.4.0] — 2026-06-25

Background jobs you can see into and cancel, token costs that account for what's
already cached, cross-border comparisons that price in your cache before
charging, and cleaner upgrades. Your existing v1.0.0 license token works as-is —
no reissue needed.

### Added

- **The background-job queue is now visible.** When a call is rate-limited and
  queued, `check_job_status` shows the job's position in line, the token balance
  it's waiting to reach before it can start, and an ETA based on your Keepa
  refill rate — so a queued screen or lookup is no longer a black box.
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
  estimate instead of being quoted — or made to wait for — the full price.
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

## [1.3.0] — 2026-06-23

Sharper demand reads for products that only map to a broad department, plus a
finder guardrail against wrong-marketplace category ids. Your existing v1.0.0
license token works as-is — no reissue needed.

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
  different Amazon marketplace than the one you're searching — a mistake that
  used to slip through and return a plausible-but-wrong match count. It checks
  every category field (include, exclude, and the sales-rank reference), and the
  rejection costs zero tokens.

## [1.2.0] — 2026-06-21

Sharpens demand reads that come from velocity (rank-drops or reviews). Your
existing v1.0.0 license token works as-is — no reissue needed.

### Changed

- **Velocity-based demand estimates are no longer understated.** When a demand
  read is built from rank-drop or review velocity rather than a direct category
  match, the per-category conversion rates were anchored to Keepa's censored
  "50+/mo" badge floor, which quietly pushed those estimates too low — most of
  all for the ~40% of products sitting right at that floor. The rates now use
  the middle of each Keepa badge band instead, lifting typical velocity-based
  estimates by roughly 1.3–1.4×. Direct category-table reads are unchanged, so
  most products read the same; the lift shows up on low-volume and
  thinner-category products.

## [1.1.0] — 2026-06-21

Corrects the demand read for low-volume products. Your existing v1.0.0
license token works as-is — no reissue needed.

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

- The sub-50 demand floor that could overstate slow sellers — the costly
  direction of error for a sourcing decision.

## [1.0.0] — 2026-06-19

First stable release. Graduates the `0.5.0-beta.*` series — the beta
designation is dropped from the version, docs, and license (the
distribution model is unchanged: a free, invited, revocable license).
Highlights since `0.5.0-beta.4`:

### Added

- **Two new tools — `resolve_codes` and `get_codes_result`** — bulk
  resolve UPC / EAN / GTIN codes to candidate ASINs from a supplier
  manifest (up to 500 rows), with a cached per-row candidate table you
  page through. The tool surface grows from **9 to 11**.
- **Inline Keepa charts in Claude Desktop chat and Claude Code.**
  `get_product_chart` now renders the price/BSR chart inline, and the
  model receives the same image so it can analyse the chart and answer
  follow-ups. (Cowork still shows text + URL only — see TROUBLESHOOTING.)
- **Cost receipts + category echo** on `execute_keepa_finder` and the
  cross-border tools — each result reports the Keepa tokens it consumed.
- **`screen_products` Title + Brand columns** in the screen output.

### Changed

- **Demand model recalibrated** — a reported demand badge is treated as
  truth, a new engine replaces the older heuristic, and cross-marketplace
  fallbacks demote to a no-read rather than guessing.
- **`get_product_details` output ~50% smaller** with no loss of signal.

### Fixed

- Keepa token over-refund (a billing-accuracy bug), job-queue durability,
  and a broad correctness + hardening pass across demand, insights, and
  analytics.

## [0.5.0-beta.4] — 2026-05-25

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
  (were 2026-04-09) — the zero-config fallback used when no
  `exchange-rates.json` override is present.

### Fixed

- **Claude Desktop now inherits the per-machine Keepa tokens-per-minute
  instead of silently defaulting to 20.** The `.mcpb` manifest gave the
  tokens-per-minute field a `default: 20`, so Claude Desktop always injected
  `20` even when the field was left blank — overriding the per-machine value
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
  to behavior — this only corrects the version shown in logs and used for
  support diagnostics. (The `beta.4` download was re-cut to include this fix;
  re-download if you installed before 2026-05-25.)

## [0.5.0-beta.3] — 2026-05-23

### Fixed

- **Claude Code installer's credential-cache write step decoded the
  license token as a 3-part JWT.** Agellic licenses are a 2-part
  `<base64url-payload>.<base64url-signature>` envelope (ed25519, not
  JWT — see `src/license/verify.ts`). The decode in beta.2 rejected
  every real license with `malformed license token (expected 3 JWT
  parts)` and skipped the cache write, defeating the headline fix
  from beta.2 (the CC-first → CD-second credential handoff). The
  decode now matches the real format and includes a whitespace-strip
  pass for parity with the server's verifier.

## [0.5.0-beta.2] — 2026-05-23

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

## [0.5.0-beta.1] — 2026-05-22

### Added

- **Public release distribution.** Pre-built `.mcpb` (Claude Desktop) and
  `.zip` (Claude Code) artifacts now ship from
  [Agellic-Commerce/agellic-releases](https://github.com/Agellic-Commerce/agellic-releases).
  Both files are byte-identical; just two filenames so each host's unzip
  tool recognises the extension.
- **Bundled Claude Code installer.** `install.mjs` ships inside the
  release artifact at root. After `unzip`, run `node install.mjs` — no
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

## [0.4.0] — 2026-05-22

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
  changed from "got non-empty offers" to "we requested offers" — an
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

## [0.2.0] — 2026-05-15

### Added

- Credential cache model: env-first / cache-fallback resolution with a
  per-machine cache at `~/.agellic-mcp/credentials.json` (mode 0600).
  Second host on the same machine can leave credential fields blank
  and pick everything up from the cache.

## [0.1.0]

- Initial closed-beta release.
