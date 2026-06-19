# Install guide

How to install, upgrade, and uninstall `agellic-mcp` in Claude Desktop
and Claude Code.

For things that can go wrong, see [TROUBLESHOOTING.md](./TROUBLESHOOTING.md).
For example prompts once installed, see [USAGE.md](./USAGE.md).

---

## Requirements

- **Agellic license token** — delivered in the email you received when
  you joined the early-access program. Starts with `eyJ…`.
- **A Keepa API key** — get one at
  [keepa.com/#!api](https://keepa.com/#!api).
- **Node.js 22.22.2+ (LTS) or 24.15.0+** (Claude Code only — Claude
  Desktop ships its own Node runtime). Node 20 reached end of life on
  2026-04-30 and is no longer supported. If you're on older Node,
  install a current LTS from
  [nodejs.org/en/download](https://nodejs.org/en/download) (or via
  `nvm`, `fnm`, `volta`, etc.).
- **One of:**
  - **Claude Desktop** (macOS or Windows — Linux is not supported by
    CD itself).
  - **Claude Code** (any platform: macOS, Linux, Windows).

You can install into both hosts on the same machine. They share
credentials at `~/.agellic-mcp/credentials.json` (mode 0600) — after
the first host is configured, the second host's install can leave
credential fields blank and pick everything up from the cache. See
[Where files live](#where-files-live).

---

## Install in Claude Desktop

1. Download `agellic-mcp.mcpb` from the
   [latest release](https://github.com/Agellic-Commerce/agellic-releases/releases/latest).
2. Open Claude Desktop → **Settings** → **Extensions**, then install the
   `.mcpb` one of two ways:
   - **Drag and drop** the `.mcpb` file into the drop target; or
   - If your build doesn't show a drop target, click **Advanced settings**
     → **Install extension…** and select the downloaded `.mcpb`.
3. Paste your **Agellic license key** and **Keepa API key** into the
   credential form. Set **Tokens per minute** to match your Keepa
   subscription tier (default `20` is calibrated to Keepa's minimum
   paid tier; raise it for higher tiers).
4. Claude Desktop installs the extension and auto-restarts it. After
   ~2-5 seconds the 9 Agellic tools appear in the tool list.

The `.mcpb` file location doesn't matter — CD reads the file wherever
you drag from and copies the contents into its own extension directory
at `~/Library/Application Support/Claude/Claude Extensions/local.mcpb.agellic.agellic-mcp/`
(macOS) or `%APPDATA%\Claude\Claude Extensions\local.mcpb.agellic.agellic-mcp\`
(Windows). After install, you can delete or move the original `.mcpb`
freely.

### First-install behavior (Claude Desktop)

On a brand-new install with no credential cache on the machine, the
server boots in **configuration-pending mode**:

- The extension shows as connected; one placeholder tool named
  `_configure_agellic` appears in the tool list. Asking Claude what
  tools it has will surface that one tool with a description telling
  you exactly what to do.
- Paste credentials into the form and click **Save**. Claude Desktop
  restarts the extension automatically.
- After ~2-5 seconds, the placeholder is replaced with the full
  11-tool set.

You should **not** see a red "could not find a valid license" banner
on a brand-new install. If you do, see
[TROUBLESHOOTING.md → error #1](./TROUBLESHOOTING.md#1-red-could-not-find-a-valid-license-banner-on-a-brand-new-install).

If you already configured agellic in Claude Code on this machine,
leave the license and Keepa fields **blank** in the CD form — they
will be filled from the per-machine credential cache automatically.

---

## Install in Claude Code

### macOS / Linux

```bash
curl -fsSL -O https://github.com/Agellic-Commerce/agellic-releases/releases/latest/download/agellic-mcp.zip
unzip agellic-mcp.zip -d agellic-install
cd agellic-install
node install.mjs
```

### Windows (PowerShell)

```powershell
Invoke-WebRequest `
  https://github.com/Agellic-Commerce/agellic-releases/releases/latest/download/agellic-mcp.zip `
  -OutFile agellic-mcp.zip
Expand-Archive agellic-mcp.zip agellic-install
cd agellic-install
node install.mjs
```

The installer:

- Prompts interactively for your **Agellic license token**, **Keepa
  API key**, and **Tokens per minute** (TPM prompt accepts blank for
  default `20`). You can also pass them as flags:
  `--license <token>`, `--keepa-key <key>`, `--tpm <integer>`.
- **Probes the server with your credentials** before writing any
  config — if a credential is wrong, you see the error immediately
  instead of on the first tool call.
- Copies the server tree to the **canonical bin path** for your OS
  (see below) using a staging directory and atomic renames.
- Writes credentials to the per-machine cache at
  `~/.agellic-mcp/credentials.json` so a subsequent Claude Desktop
  install can leave its credential fields blank.
- Merges an `mcpServers.agellic` entry into `~/.claude.json`.

Restart Claude Code (`/restart` or quit the process) to pick up the
new MCP server. The 9 Agellic tools appear in the next session.

If you already configured agellic in Claude Desktop on this machine,
run `node install.mjs --non-interactive` — the installer reads
credentials from `~/.agellic-mcp/credentials.json` and skips the
prompts entirely.

### Canonical bin paths

| OS | Path |
|---|---|
| macOS | `~/Library/Application Support/Agellic/` |
| Windows | `%LOCALAPPDATA%\Agellic\` |
| Linux | `$XDG_DATA_HOME/agellic/` (typically `~/.local/share/agellic/`) |

The directory contains `server.js` + its runtime dependencies +
`version.json` (install metadata).

---

## Upgrade

### Claude Desktop

Drag the new `agellic-mcp.mcpb` into Settings → Extensions (or
**Advanced settings → Install extension…** if there's no drop target).
CD prompts for credentials again on the first launch after upgrade —
paste them, or leave blank to pick up from the credential cache.

### Claude Code

```bash
node install.mjs --upgrade
```

Re-reads credentials from `~/.claude.json` (or the cache if the
config was wiped), so you don't need to re-pass `--license` or
`--keepa-key` on every upgrade. Pass them explicitly only to rotate
a value.

The upgrade uses a sibling staging directory and atomic renames — an
interrupted upgrade leaves the previous install intact, never
half-installed.

---

## Refreshing exchange rates

`resolve_cross_border` converts target-marketplace prices into the
source currency using a rate table baked into the build — its date is
reported as `summary.exchangeRateDate` in every cross-border result.
The bundled defaults are fine for the ballpark gap analysis the tool
does, but they drift a few percent over weeks.

### Claude Code — refresh on demand

```bash
node install.mjs --refresh-rates
```

Fetches current ECB reference rates (via Frankfurter — no API key) and
writes `~/.agellic-mcp/exchange-rates.json`, which the server uses **in
preference to the bundled table**. A missing or invalid file falls back
to the bundled defaults silently, and a failed fetch (network down,
partial data) leaves any existing file untouched — never a partial
write. This doesn't reinstall or upgrade anything; it only writes the
rate file. Restart your Claude session to pick up the new rates.

Because `~/.agellic-mcp/` is shared between hosts, a refresh run from
Claude Code also applies to Claude Desktop on the same machine.

### Claude Desktop — limitation

The `.mcpb` install has no terminal step, so **Claude Desktop-only users
can't run `--refresh-rates`.** You get fresher rates by either:

- **upgrading to a newer release** — each `.mcpb` ships refreshed bundled
  defaults; or
- **also installing the Claude Code build on the same machine** and
  running `--refresh-rates` once — CD then reads the shared override.

A pure-CD install otherwise stays on the bundled defaults, which is fine
for the ballpark comparison `resolve_cross_border` is designed for.

---

## Uninstall

### Claude Desktop

Settings → Extensions → click the agellic extension → remove.

### Claude Code

Three levels of removal:

**Config only** (default) — removes the `mcpServers.agellic` entry
from `~/.claude.json`, leaves the binary + data dir in place:

```bash
node install.mjs --uninstall
```

**Config + binary** — also removes the binary at the canonical bin
path. Data dir preserved:

```bash
node install.mjs --uninstall --remove-bin
```

**Full purge** — removes everything: config entry, binary, AND
`~/.agellic-mcp/` (credential cache, cached products, logs, jobs):

```bash
node install.mjs --uninstall --purge
```

The full purge **refuses by default if the Claude Desktop extension
is still installed** — the data dir is shared between hosts, and
purging while CD is active would corrupt CD's runtime state.
Uninstall CD first, or pass `--force` to override (only do this if
you understand the consequences).

---

## Where files live

| Location | What lives there |
|---|---|
| `~/.claude.json` | `mcpServers.agellic` entry (Claude Code only) — license + Keepa key + TPM in the env block, mode 0600 |
| `~/.agellic-mcp/credentials.json` | Shared per-machine credential cache (mode 0600). Both hosts read; written after every successful server boot AND after every successful installer probe |
| `~/.agellic-mcp/exchange-rates.json` | Optional FX-rate override for `resolve_cross_border`, written by `--refresh-rates` (Claude Code). Shared between hosts; absent = bundled defaults. Mode 0644 (rates aren't secret) |
| Canonical bin path | `server.js` + dependencies + `version.json` (Claude Code only) |
| CD extension dir | Same server tree, managed by Claude Desktop (Claude Desktop only) |
| `~/.agellic-mcp/` | Shared data dir: credential cache, cached products, result sets, token bucket state, job queue, logs |

The data dir can be overridden via the `AGELLIC_DATA_DIR` environment
variable. The bin paths and CD extension paths are platform
conventions and not configurable.

---

## Logs

Per-day rotated logs at `<dataDir>/logs/agellic-<YYYY-MM-DD>.log`.
License tokens and Keepa API keys are redacted from logs.

Two env vars control log volume:

- `AGELLIC_LOG_LEVEL` — `error | warn | info | debug` (default `info`).
- `AGELLIC_LOG_DEBUG` — comma-list of module names to elevate to
  debug (e.g. `tool-call,token-bucket`).

The server prints its log path to stderr on boot. Claude Code surfaces
this in the MCP server status panel; Claude Desktop also writes its
own per-server log at `~/Library/Logs/Claude/mcp-server-agellic.log`
(macOS) or `%APPDATA%\Claude\Logs\mcp-server-agellic.log` (Windows).

---

## Verifying an install

Ask Claude (in either host) to call **`check_token_balance`**. This
exercises the license check, Keepa key load, and storage layer
without spending any Keepa tokens.

If `check_token_balance` returns your current balance and refill rate,
the install is healthy.

If it fails, see [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) — the
error message names the specific failure mode.
