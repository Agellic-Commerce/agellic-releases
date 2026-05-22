# agellic-releases

Pre-built release artifacts for **agellic-mcp** — Buy-Box-aware Amazon
product intelligence inside Claude.

Source code lives in the private `Agellic-Commerce/agellic-mcp` repo.
This repo hosts only the published release artifacts.

## Latest release

[**→ Download from /releases/latest**](https://github.com/Agellic-Commerce/agellic-releases/releases/latest)

## Which file do I download?

| Host | File | Recipe |
|---|---|---|
| Claude Desktop (macOS / Windows) | `agellic-mcp.mcpb` | Drag into CD → Settings → Extensions |
| Claude Code (macOS / Linux) | `agellic-mcp.zip` | `unzip` + `node install.mjs` |
| Claude Code (Windows) | `agellic-mcp.zip` | `Expand-Archive` + `node install.mjs` |

Both files are **byte-identical**; just two filenames so each host's
unzip tool recognises the extension.

## Prerequisites

- **Agellic license token** — from your purchase email
- **Keepa API key** — get one at [keepa.com/#!api](https://keepa.com/#!api)
- **Node.js 22.22.2+ or 24.15.0+** (Claude Code only — Claude Desktop ships
  its own Node runtime)

## Documentation

Tool reference, install troubleshooting, and the full credential model
live in the source repo: [Agellic-Commerce/agellic-mcp](https://github.com/Agellic-Commerce/agellic-mcp).

## Beta status

We're in closed beta. Supply-chain signing (Sigstore attestation, signed
checksums, SBOM) is on the roadmap for GA — see the source repo's
[`docs/superpowers/specs/2026-05-22-beta-distribution-design.md`](https://github.com/Agellic-Commerce/agellic-mcp/blob/main/docs/superpowers/specs/2026-05-22-beta-distribution-design.md)
for the full trust-stack design.
