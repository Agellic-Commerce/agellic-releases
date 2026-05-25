# Changelog

All notable changes to agellic-mcp are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
