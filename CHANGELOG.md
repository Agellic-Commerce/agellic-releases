# Changelog

All notable changes to agellic-mcp are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
