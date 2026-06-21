# Agellic — Amazon product intelligence for AI assistants

**Ask in plain English, get a decision — not a spreadsheet.** Demand
estimates, Buy Box and stock signals, cross-marketplace arbitrage, and
bulk screening, powered by your own [Keepa](https://keepa.com/) key.

agellic is a local [Model Context Protocol](https://modelcontextprotocol.io)
(MCP) server, so it runs in any MCP-compatible assistant. Today it ships
with one-click installers for **Claude Desktop** and **Claude Code**,
with more hosts as the MCP ecosystem grows.

> Source code lives in the private `Agellic-Commerce/agellic-mcp` repo;
> this repo hosts the published release artifacts.

## Why agellic

**Answers, not spreadsheets.** It reads Keepa's raw history and hands the
model the *conclusions* — Buy Box health, stock depth, rank trends,
out-of-stock patterns, demand stability, seasonality — so you ask in
plain English and get a judgment, not a CSV to squint at.

**Demand Read.** A calibrated demand-estimation model — a real read on
how a product moves, not just "BSR #14,000."

**It won't torch your Keepa tokens.** Every call reports its token cost,
balance is checked before spending, failed calls are refunded, and
oversized jobs queue instead of erroring. Your metered quota is treated
as the scarce resource it is.

**Remembers across chats.** Fetched products and result sets are cached
on your machine and shared across every chat and every connected app — so
you never pay Keepa twice for the same lookup, and you can reopen a
finder or cross-border run in a fresh conversation.

**Background jobs that drain themselves — locally.** Too big to run right
now? It queues and works the backlog automatically as Keepa tokens
refill, no babysitting. It runs on **your machine, not the cloud — leave
the app open and the machine awake for jobs to progress** — and the queue
is durable, so quitting pauses it and relaunching resumes right where it
left off.

**Cross-marketplace arbitrage in one question.** Match a product across
Amazon marketplaces by UPC/EAN, convert the prices, and surface the gap
— sourcing research that's tedious by hand.

**Natural-language product discovery.** Describe what you want ("car
accessories under $30, rank under 50k") and it builds the Keepa Product
Finder query — then bulk-screens and ranks the hits against your
criteria. No filter-form wrangling.

**Built for the model.** Compact, structured output that fits the context
window (cheaper, faster, sharper answers) — plus price/BSR charts the
model can actually see and read.

**One paste, set up once, batteries included.** Runs in Claude Desktop
and Claude Code today, configure once per machine (shared credentials), with
category + demand calibration data bundled in — no extra downloads.

## Latest release

[**→ Download from /releases/latest**](https://github.com/Agellic-Commerce/agellic-releases/releases/latest)

## Which file do I download?

| Host                             | File               | Recipe                                |
| -------------------------------- | ------------------ | ------------------------------------- |
| Claude Desktop (macOS / Windows) | `agellic-mcp.mcpb` | Drag into CD → Settings → Extensions  |
| Claude Code (macOS / Linux)      | `agellic-mcp.zip`  | `unzip` + `node install.mjs`          |
| Claude Code (Windows)            | `agellic-mcp.zip`  | `Expand-Archive` + `node install.mjs` |

Both files are **byte-identical**; just two filenames so each host's
unzip tool recognises the extension.

## Prerequisites

- **Agellic license token** — from the email
- **Keepa API key** — get one at [keepa.com/#!api](https://keepa.com/#!api)
- **Node.js 22.22.2+ or 24.15.0+** (Claude Code only — Claude Desktop ships
  its own Node runtime)

## Documentation

- [**Install guide**](./INSTALL.md) — requirements, step-by-step for CD
  and CC, plus upgrade and uninstall.
- [**Tool reference**](./TOOLS.md) — the 11 tools, what each one does, and
  what it costs in Keepa tokens.
- [**Usage examples**](./USAGE.md) — example prompts to try first.
- [**Troubleshooting**](./TROUBLESHOOTING.md) — first-install behavior,
  log locations, common errors.
- [**FAQ**](./FAQ.md) — Keepa token economics, release status, billing,
  privacy, compatibility.
- [**Changelog**](./CHANGELOG.md) — release notes.
- [**License**](./LICENSE.md) — proprietary early-access terms (free
  of charge, AS-IS, no warranty, no redistribution).

## Release status

This is the **v1.1.0 early-access release**, distributed to invited
testers on a free, revocable license. Today's artifacts are unsigned
builds; the trust stack — Sigstore / cosign supply-chain attestation,
signed checksums, and an SBOM — is on the roadmap. Until then, install
only from `Agellic-Commerce/agellic-releases` and verify file sizes
against the release page before running.
