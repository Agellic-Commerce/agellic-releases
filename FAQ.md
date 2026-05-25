# FAQ

Answers to questions cohort testers actually ask.

## Keepa tokens

### What's a Keepa token?

A Keepa token is a unit of API quota issued by Keepa, the data vendor
agellic-mcp sits on top of. Every tool call consumes some number of
tokens against the API key you supply at install. Per-tool costs are
listed in [TOOLS.md](./TOOLS.md).

### Why do tools have different costs?

Discovery is cheap; per-ASIN enrichment costs more, scaling with the
depth of data each tool needs:

| Tool | Cost | What it pays for |
|---|---|---|
| `execute_keepa_finder` | cached page reads | discovery against pre-aggregated Keepa pages |
| `screen_products` | 3 tokens / ASIN | lite product record + buy-box snapshot |
| `get_product_details` | ~8 tokens / ASIN (9 reserved, unused graph token refunded) | full product record (offers, history, dimensions, calibrated demand) |
| `resolve_cross_border` | up to 12 tokens / ASIN | 3 for the source ASIN + 9 for up to 3 target-marketplace candidate fetches |
| `get_product_chart` | 1 token / chart | rendered PNG from Keepa |

`check_token_balance`, `check_job_status`, and `get_*_result` are
local — they cost nothing.

### What happens when I run out of tokens?

The server enqueues a background job and returns a `jobId`. Poll it
with `check_job_status`. The token bucket refills at
`AGELLIC_MCP_TOKENS_PER_MINUTE` per minute (default 20, matching Keepa's
minimum paid tier). Once the bucket has enough capacity to cover the
job, it runs. Until then the job stays `pending` — no work lost, no
retry needed on your end.

### Do tokens carry over month-to-month?

That's a Keepa subscription question, not an agellic one. Check your
Keepa account dashboard at [keepa.com/#!api](https://keepa.com/#!api).

### Where do I check my balance?

Ask Claude to run `check_token_balance`. It's a free local read —
no Keepa call, no cost.

## Beta vs GA

### What does "closed beta" mean here?

Unsigned build artifacts distributed via the
`Agellic-Commerce/agellic-releases` GitHub repo to invited cohort
testers. There's no supply-chain attestation yet, no automatic
updates, and no public signup. Issues are triaged manually by the
agellic team via `support@agellic.com`.

### What's planned for GA?

- Sigstore / cosign supply-chain attestation
- Signed checksums published alongside each release
- SBOM (CycloneDX) per artifact
- Linux and Windows smoke coverage in CI

No dates promised — these are the gates we're working through, not
a schedule.

### Will my license keep working as the beta evolves?

Yes. Your beta license stays valid going forward — no expiry, no
reissue, no migration step on our end.

## Privacy

### What does the MCP send to Keepa?

Only the call parameters you'd send if you used Keepa's API directly:
the ASIN list, the marketplace domain code, and the filter parameters
you specified. No conversation transcript, no Claude context, no PII.
Each call goes from your machine directly to `api.keepa.com` over
HTTPS.

### Are my queries logged?

Yes — locally, at `~/.agellic-mcp/logs/agellic-<date>.log` (rotated
daily, 14-day retention by default). License tokens and Keepa API
keys are redacted before they touch the log. Logs stay on your machine;
agellic does not collect telemetry and the server makes no outbound
calls except to Keepa.

### Does agellic see my Claude conversations?

No. The MCP boundary only exposes the tool-call arguments Claude
chooses to send (e.g. an ASIN list, a domain code). The surrounding
conversation, your prompts, and Claude's reasoning are never sent
to the agellic process.

## Compatibility

### Which Claude clients does this work in?

- **Claude Desktop** (macOS, Windows) — install via the `.mcpb`
  extension. Drag the file into Settings → Extensions.
- **Claude Code** (macOS, Linux, Windows) — install via
  `node install.mjs` after unzipping the release archive.

Linux desktop is not supported because Claude Desktop itself doesn't
ship a Linux build. If/when that changes, the existing `.mcpb` will
work there too.

### Cowork limitations?

Cowork (Claude Desktop's agent-mode surface) currently strips
`type: 'image'` content blocks from MCP tool results before the model
sees them. The practical effect: `get_product_chart` falls back to
its text-only summary in Cowork instead of showing the rendered PNG.
Every other tool works normally. Regular Claude Desktop chat renders
the chart inline as expected.

In the meantime, switch to a regular chat thread when you need chart
visuals.

## Exchange rates

### How current are the rates in `resolve_cross_border`?

The tool converts prices between marketplaces using an exchange-rate
table baked into the build; each cross-border result reports that table's
date as `summary.exchangeRateDate`. The rates are approximate — fine for
the ballpark price-gap analysis the tool does, but they drift a few
percent over weeks. Every release ships with refreshed bundled defaults.

### Can I update the exchange rates myself?

**Claude Code** — yes, any time:

```bash
node install.mjs --refresh-rates
```

This fetches current ECB reference rates (via Frankfurter) and writes
`~/.agellic-mcp/exchange-rates.json`, which the server uses in preference
to the bundled table. Restart your Claude session afterward. It doesn't
reinstall anything — just the rate file.

**Claude Desktop only** — the `.mcpb` install has no terminal step, so
CD-only users can't run `--refresh-rates`. You get fresher rates by
upgrading to a newer release (refreshed bundled defaults), or by also
installing the Claude Code build on the same machine — the rate file
lives in the shared `~/.agellic-mcp/`, so one refresh covers both hosts.
