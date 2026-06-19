# agellic-releases

Pre-built release artifacts for **agellic-mcp** — Buy-Box-aware Amazon
product intelligence inside Claude.

Source code lives in the private `Agellic-Commerce/agellic-mcp` repo.
This repo hosts only the published release artifacts.

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

This is the **v1.0.0 early-access release**, distributed to invited
testers on a free, revocable license. Today's artifacts are unsigned
builds; the trust stack — Sigstore / cosign supply-chain attestation,
signed checksums, and an SBOM — is on the roadmap. Until then, install
only from `Agellic-Commerce/agellic-releases` and verify file sizes
against the release page before running.
