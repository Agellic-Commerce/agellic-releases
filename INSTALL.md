# Install guide

How to install, upgrade, and uninstall `agellic-mcp` in Claude Desktop,
Claude Code, and Codex (Codex CLI + the ChatGPT desktop app).

For things that can go wrong, see [TROUBLESHOOTING.md](./TROUBLESHOOTING.md).
For example prompts once installed, see [USAGE.md](./USAGE.md).

---

## Requirements

- **Agellic license token**: delivered in the email you received when
  you joined the early-access program. Starts with `eyJ…`.
- **A Keepa API key**: get one at
  [keepa.com/#!api](https://keepa.com/#!api).
- **Node.js 22.22.2+ (LTS) or 24.15.0+** (for the scripted installs:
  Claude Code and Codex. Claude Desktop ships its own Node runtime).
  Node 20 reached end of life on 2026-04-30 and is no longer supported.
  If you're on older Node, install a current LTS from
  [nodejs.org/en/download](https://nodejs.org/en/download) (or via
  `nvm`, `fnm`, `volta`, etc.).
- **One of:**
  - **Claude Desktop** (macOS or Windows, Linux is not supported by
    CD itself).
  - **Claude Code** (any platform: macOS, Linux, Windows).
  - **Codex CLI and/or the ChatGPT desktop app** (they share one MCP
    configuration; the scripted install also needs the `codex` CLI on
    PATH, see [If `codex` is not installed](#if-codex-is-not-installed)).

You can install into any combination of hosts on the same machine. They
share credentials at `~/.agellic-mcp/credentials.json` (mode 0600):
after the first host is configured, the next host's install can leave
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
   ~2-5 seconds the 11 Agellic tools appear in the tool list.

The `.mcpb` file location doesn't matter, CD reads the file wherever
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

If you already configured agellic in Claude Code or Codex on this
machine, leave the license and Keepa fields **blank** in the CD form:
they will be filled from the per-machine credential cache
automatically.

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
  config: if a credential is wrong, you see the error immediately
  instead of on the first tool call.
- Copies the server tree to the **canonical bin path** for your OS
  (see below) using a staging directory and atomic renames.
- Writes credentials to the per-machine cache at
  `~/.agellic-mcp/credentials.json` so a later install into any other
  host (Claude Desktop's form, or a Codex install) can leave its
  credential fields blank.
- Merges an `mcpServers.agellic` entry into `~/.claude.json`.

Restart Claude Code (`/restart` or quit the process) to pick up the
new MCP server. The 11 Agellic tools appear in the next session.

### Adding Claude Code as a second host

Already configured agellic in Codex or Claude Desktop on this machine?
Run the installer again from **inside the unzipped `agellic-install`
folder** (the folder the first scripted install ran from). The shared
credential cache makes it promptless:

```bash
node install.mjs --non-interactive
```

If you no longer have that folder (you deleted it, or your first
install was the Claude Desktop `.mcpb`, which never used one),
download and unzip the archive first (the steps at the top of this
section), then run the command from inside `agellic-install`.

### Canonical bin paths

| OS | Claude Code | Codex |
|---|---|---|
| macOS | `~/Library/Application Support/Agellic/` | `~/Library/Application Support/Agellic-Codex/` |
| Windows | `%LOCALAPPDATA%\Agellic\` | `%LOCALAPPDATA%\Agellic-Codex\` |
| Linux | `$XDG_DATA_HOME/agellic/` (typically `~/.local/share/agellic/`) | `$XDG_DATA_HOME/agellic-codex/` |

Each directory contains `server.js` + its runtime dependencies +
`version.json` (install metadata). Each scripted host owns its own
tree, so removing one host never breaks another.

---

## Install in Codex CLI + ChatGPT desktop

Codex CLI, the ChatGPT desktop app, and the Codex IDE extension share
one MCP configuration, managed by the `codex` CLI. The installer
registers through it (it never edits `~/.codex/config.toml` by hand).

This section is self-contained: you do not need Claude Code or any
other host installed first. If another host is already installed on
this machine, skip to
[Adding Codex as a second host](#adding-codex-as-a-second-host).

### macOS / Linux

```bash
curl -fsSL -O https://github.com/Agellic-Commerce/agellic-releases/releases/latest/download/agellic-mcp.zip
unzip agellic-mcp.zip -d agellic-install
cd agellic-install
node install.mjs --host codex
```

### Windows (PowerShell)

```powershell
Invoke-WebRequest `
  https://github.com/Agellic-Commerce/agellic-releases/releases/latest/download/agellic-mcp.zip `
  -OutFile agellic-mcp.zip
Expand-Archive agellic-mcp.zip agellic-install
cd agellic-install
node install.mjs --host codex
```

It is the same release archive as the Claude Code install (one
archive covers every scripted host); `--host codex` is the only
difference. On a machine with no agellic install yet, the installer
prompts for your **Agellic license token**, **Keepa API key**, and
**Tokens per minute**, exactly as in the Claude Code install (the
same flags work too: `--license <token>`, `--keepa-key <key>`,
`--tpm <integer>`).

What the install does, in order:

1. Installs the server tree to the **Codex-owned** bin path
   (`Agellic-Codex`, see [Canonical bin paths](#canonical-bin-paths)).
2. Probes the binary with a real MCP `initialize` round-trip before
   touching any configuration.
3. Writes the shared credential cache at
   `~/.agellic-mcp/credentials.json` **strictly**: if this write fails,
   the install stops and no Codex registration is made.
4. Registers a **credential-free** entry via `codex mcp add`. Your
   license and Keepa key never enter Codex configuration; the server
   reads them from the shared cache at boot.

Then restart Codex CLI sessions and/or the ChatGPT desktop app, run
`codex mcp list` (expect `agellic`), and ask the assistant to call
`check_token_balance`.

### Adding Codex as a second host

Already configured agellic in Claude Code or Claude Desktop on this
machine? Run the installer again from **inside the unzipped
`agellic-install` folder** (the folder the first scripted install ran
from). The shared credential cache makes it promptless:

```bash
node install.mjs --host codex --non-interactive
```

If you no longer have that folder (you deleted it, or your first
install was the Claude Desktop `.mcpb`, which never used one),
download and unzip the archive first (the steps just above), then run
the command from inside `agellic-install`.

### If `codex` is not installed

The installer still installs and probes the server tree and writes the
credential cache, then exits with code 2 after printing two remedies,
preferred first:

1. Install the Codex CLI (see OpenAI's install instructions), then
   re-run `node install.mjs --host codex`. Registration is idempotent,
   so re-running is always safe, and the one command covers Codex CLI
   and ChatGPT desktop because they share configuration.
2. Or register manually in ChatGPT desktop: **Settings → MCP servers →
   Add server**, choose **STDIO**, set the command to `node` with the
   printed `server.js` path as its only argument, then restart the app.

### Custom data dir

If `AGELLIC_DATA_DIR` is set when you install, the installer records it
in the Codex entry (the one env value that is ever written there):
ChatGPT desktop doesn't inherit your shell environment, and every host
must use the same data dir.

---

## Upgrade

### Claude Desktop

Drag the new `agellic-mcp.mcpb` into Settings → Extensions (or
**Advanced settings → Install extension…** if there's no drop target).
CD prompts for credentials again on the first launch after upgrade:
paste them, or leave blank to pick up from the credential cache.

### Claude Code

```bash
node install.mjs --upgrade
```

Re-reads credentials from `~/.claude.json` (or the cache if the
config was wiped), so you don't need to re-pass `--license` or
`--keepa-key` on every upgrade. Pass them explicitly only to rotate
a value.

The upgrade uses a sibling staging directory and atomic renames: an
interrupted upgrade leaves the previous install intact, never
half-installed.

An upgrade also clears the shared derived stores (cached products,
result sets, and the job queue; credentials, rate-limit state, and
exchange rates survive), so restart your other connected hosts after
upgrading any one of them.

### Codex

```bash
node install.mjs --host codex --upgrade
```

Same staged-promote mechanics and derived-store clearing, against the
Codex-owned tree. Credentials come from the shared cache, and the
registration is re-applied idempotently (which also heals a moved
server path).

### All hosts at once

```bash
node install.mjs --host all --upgrade
```

Detects which scripted hosts are installed (Claude Code, Codex) and
upgrades each in turn with a per-host report. One host failing doesn't
stop the sweep, but the exit code is nonzero. Claude Desktop is
excluded: its upgrades ship via the `.mcpb` flow above. `--host all` is
valid only with `--upgrade`; install and uninstall stay explicit per
host.

---

## Refreshing exchange rates

`resolve_cross_border` converts target-marketplace prices into the
source currency using a rate table baked into the build. Its date is
reported as `summary.exchangeRateDate` in every cross-border result.
The bundled defaults are fine for the ballpark gap analysis the tool
does, but they drift a few percent over weeks.

### Claude Code: refresh on demand

```bash
node install.mjs --refresh-rates
```

Fetches current ECB reference rates (via Frankfurter, no API key) and
writes `~/.agellic-mcp/exchange-rates.json`, which the server uses **in
preference to the bundled table**. A missing or invalid file falls back
to the bundled defaults silently, and a failed fetch (network down,
partial data) leaves any existing file untouched, never a partial
write. This doesn't reinstall or upgrade anything; it only writes the
rate file. Restart your Claude session to pick up the new rates.

Because `~/.agellic-mcp/` is shared between hosts, a refresh run from
Claude Code also applies to Claude Desktop and Codex on the same
machine.

### Claude Desktop: limitation

The `.mcpb` install has no terminal step, so **Claude Desktop-only users
can't run `--refresh-rates`.** You get fresher rates by either:

- **upgrading to a newer release**: each `.mcpb` ships refreshed bundled
  defaults; or
- **also installing the Claude Code build on the same machine** and
  running `--refresh-rates` once: CD then reads the shared override.

A pure-CD install otherwise stays on the bundled defaults, which is fine
for the ballpark comparison `resolve_cross_border` is designed for.

---

## Uninstall

### Claude Desktop

Settings → Extensions → click the agellic extension → remove.

### Claude Code and Codex

Three levels of removal, per host. `--host claude-code` is the default;
add `--host codex` to act on the Codex registration and tree instead.

**Config only** (default): removes only that host's entry (the
`mcpServers.agellic` entry in `~/.claude.json`, or the `agellic` entry
in Codex configuration via `codex mcp remove`). Leaves the host's
binary and the shared data dir in place. Use this to disable agellic
in one host while keeping every other host installed:

```bash
node install.mjs --uninstall
node install.mjs --host codex --uninstall
```

**Config + binary**: also removes that host's own binary tree. Data
dir preserved (your token bucket state, cached products, and result
sets all stay). Other hosts' entries, trees, and shared data are
untouched:

```bash
node install.mjs --uninstall --remove-bin
node install.mjs --host codex --uninstall --remove-bin
```

**Full purge**: removes everything the acting host owns AND
`~/.agellic-mcp/` (credential cache, cached products, logs, jobs):

```bash
node install.mjs --uninstall --purge
```

The full purge **refuses by default while any other host is still
installed** (Claude Desktop, Claude Code, or Codex; the refusal names
them). The data dir is shared, and purging it under a live host would
corrupt that host's runtime state. The order that always works, no
`--force` needed:

1. Run `--uninstall` (or `--uninstall --remove-bin`) for each other
   host first. Those never touch shared data, so they are never
   blocked.
2. Run `--uninstall --purge` from the last remaining host. It finds
   no other presence and proceeds.

`--force` overrides the refusal if you understand the consequences.

If the `codex` CLI is missing during a Codex uninstall: config-only
`--uninstall` changes nothing (removing the entry needs the CLI) and
prints the manual removal command. `--remove-bin` and `--purge` still
perform their filesystem removals and print the same manual step for
the config entry, so a leftover Codex tree never blocks the purge
ordering above.

### Uninstall gotchas

**Lost your `install.mjs`?** The uninstaller ships only inside the
release archive: there's no separate download. If you deleted the
unzipped `agellic-install/` folder, you have nothing to run
`--uninstall` with. Either re-download `agellic-mcp.zip` from the
[latest release](https://github.com/Agellic-Commerce/agellic-releases/releases/latest),
unzip it, and run `node install.mjs --uninstall` (add `--remove-bin` or
`--purge`) from there, or remove agellic by hand:

1. Delete the `mcpServers.agellic` entry from `~/.claude.json` (Claude
   Code), and/or run `codex mcp remove agellic` (Codex; or remove the
   server in ChatGPT desktop → Settings → MCP servers).
2. Delete the canonical bin directories you installed (see
   [Canonical bin paths](#canonical-bin-paths)): macOS
   `~/Library/Application Support/Agellic/` and
   `~/Library/Application Support/Agellic-Codex/`, Windows
   `%LOCALAPPDATA%\Agellic\` and `%LOCALAPPDATA%\Agellic-Codex\`.
3. Optionally `rm -rf ~/.agellic-mcp` to remove the shared data dir
   (see [Where files live](#where-files-live)).

**Quit every host before `--purge`** (Claude Desktop, Claude Code
sessions, Codex CLI sessions, and the ChatGPT desktop app). The purge
deletes `~/.agellic-mcp/`, but a server process that's still running
recreates `~/.agellic-mcp/jobs/…` on its next job-queue tick, so the
data dir can reappear seconds after the purge reports success. On macOS
a Claude Desktop extension can keep running from memory even after you
remove it in Settings, so quit Claude Desktop fully (not just close the
window). If you already purged and `~/.agellic-mcp/` came back, quit
every host and `rm -rf ~/.agellic-mcp` once more.

---

## Where files live

| Location | What lives there |
|---|---|
| `~/.claude.json` | `mcpServers.agellic` entry (Claude Code only): license + Keepa key + TPM in the env block, mode 0600 |
| `~/.codex/config.toml` | `agellic` entry (Codex hosts): credential-free, written via `codex mcp add`, never hand-edited by the installer. `AGELLIC_DATA_DIR` is the only env value ever recorded there |
| `~/.agellic-mcp/credentials.json` | Shared per-machine credential cache (mode 0600). All hosts read; written after every successful server boot AND after every successful installer probe |
| `~/.agellic-mcp/exchange-rates.json` | Optional FX-rate override for `resolve_cross_border`, written by `--refresh-rates` (Claude Code). Shared between hosts; absent = bundled defaults. Mode 0644 (rates aren't secret) |
| Canonical bin paths | `server.js` + dependencies + `version.json`, one tree per scripted host: `Agellic` (Claude Code), `Agellic-Codex` (Codex) |
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

- `AGELLIC_LOG_LEVEL`: `error | warn | info | debug` (default `info`).
- `AGELLIC_LOG_DEBUG`: comma-list of module names to elevate to
  debug (e.g. `tool-call,token-bucket`).

The server prints its log path to stderr on boot. Claude Code surfaces
this in the MCP server status panel; Claude Desktop also writes its
own per-server log at `~/Library/Logs/Claude/mcp-server-agellic.log`
(macOS) or `%APPDATA%\Claude\Logs\mcp-server-agellic.log` (Windows).

---

## Verifying an install

Ask the assistant (in any host) to call **`check_token_balance`**. This
exercises the license check, Keepa key load, and storage layer
without spending any Keepa tokens.

If `check_token_balance` returns your current balance and refill rate,
the install is healthy.

For Codex hosts you can also check the registration first from a
terminal: `codex mcp list` should show `agellic` as enabled.

If it fails, see [TROUBLESHOOTING.md](./TROUBLESHOOTING.md): the
error message names the specific failure mode.
