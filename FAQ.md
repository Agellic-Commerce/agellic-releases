# FAQ

Answers to questions testers actually ask.

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
| `execute_keepa_finder` | 10 + 1 per 100 ASINs returned | discovery against Keepa's pre-aggregated finder index |
| `screen_products` | 3 tokens / ASIN | lite product record + buy-box snapshot |
| `get_product_details` | ~8 tokens / ASIN (9 reserved, unused graph token refunded) | full product record (offers, history, dimensions, calibrated demand) |
| `resolve_cross_border` | up to 12 tokens / ASIN | 3 for the source ASIN + 9 for up to 3 target-marketplace candidate fetches |
| `get_product_chart` | 1 token / chart | rendered PNG from Keepa |

`check_token_balance`, `check_job_status`, and `get_*_result` are
local: they cost nothing.

### What happens when I run out of tokens?

The server enqueues a background job and returns a `jobId`. Poll it
with `check_job_status`. The token bucket refills at
`AGELLIC_MCP_TOKENS_PER_MINUTE` per minute (default 20, matching Keepa's
minimum paid tier). Once the bucket has enough capacity to cover the
job, it runs. Until then the job stays `pending`: no work lost, no
retry needed on your end.

### Do tokens carry over month-to-month?

That's a Keepa subscription question, not an agellic one. Check your
Keepa account dashboard at [keepa.com/#!api](https://keepa.com/#!api).

### Where do I check my balance?

Ask Claude to run `check_token_balance`. It's a free local read:
no Keepa call, no cost.

## Caching & background jobs

### If I look up the same product twice, do I pay twice?

No. Every paid fetch is cached on your machine (products for 24 hours,
cross-border analyses for 7 days) and the cache is shared across every
chat and both Claude apps. Re-asking about the same ASIN, even in a
brand-new conversation or the other host, reads from cache at **zero
Keepa tokens**. You can also reopen a stored finder, screen, or
cross-border result by id long after the original chat scrolled away:
see `get_finder_result` / `get_cross_border_result` / `get_codes_result`
in [TOOLS.md](./TOOLS.md).

### Do background jobs keep running if I close Claude?

Only while a Claude app is open. The job runner lives inside the MCP
server process, which Claude Desktop and Claude Code start and stop with
the app: there's no cloud worker. Jobs are durable, though: quitting
Claude (or restarting your machine) **pauses** the queue, and the next
time you launch Claude the job resumes from where it left off: nothing
is lost. So for a big screen or cross-border run, kick it off and leave
Claude running; it drains as your Keepa tokens refill.

## Prices

### Why did Buy Box prices jump after upgrading to v1.8.0?

Because they now include shipping. As of v1.8.0 every Buy Box price this
server reports is the **landed** price (item + shipping), which is what a
buyer actually pays and what Amazon charges its referral fee on. A product
that read $10.99 before may read $15.98 now with no offer change at all:
the item price did not move, the label got honest. Products with free
shipping are unaffected, because their shipping component is zero.

The practical follow-up: re-check any saved price filter, ROI target, or
alert threshold you tuned against the old item-only numbers, since those
are now being compared against a larger number on shipped items.

### Which prices are landed, and which are not?

The Buy Box lane is landed everywhere and always: `pricing.buyBox.*`,
`competition.buyBox.priceCents`, the `BB(c)` column in `screen_products`,
the Buy Box lane of the sell-price read, price position, and the finder's
`avgBuyBox`. Keepa's Buy Box series carries shipping across its entire
history, so there is no cutover date on this lane.

The lowest-new lane is the one with a boundary: Keepa began including
shipping there on 2026-02-23, and it was item-only before that. Any read
that depends on it declares a `shippingBasis` of `landed`, `item-only`, or
`mixed-window`, and a `mixed-window` result means the window straddles that
date, so the prices inside it were measured two different ways.

When shipping is greater than zero, `pricing.buyBox.itemCents` and
`pricing.buyBox.shippingCents` show you exactly how a landed price splits.

## Release status & roadmap

### What does "early-access" mean here?

v1.8.0 is the current stable release, but distribution is still early-access:
unsigned build artifacts delivered via the
`Agellic-Commerce/agellic-releases` GitHub repo to invited testers,
with no supply-chain attestation yet, no automatic updates, and no
public signup. Issues are triaged manually by the agellic team via
`support@agellic.com`.

### What's still on the roadmap?

- Sigstore / cosign supply-chain attestation
- Signed checksums published alongside each release
- SBOM (CycloneDX) per artifact
- Linux and Windows smoke coverage in CI

No dates promised: these are the gates we're working through, not
a schedule.

### Do I need a new license for v1.8.0?

No. If you already have a v1.0.0 license token, it works as-is on v1.8.0:
same signing key, same coverage window. (Only the beta → v1.0.0 graduation
required a one-time reissue, because v1.0.0 rotated the signing key and
stopped accepting beta-era tokens; use the v1.0.0 token from that email.) If
a future build ever falls outside your license's coverage window, you'll get
a clear error at startup and we'll renew it: email `support@agellic.com`.

## Privacy

### What does the MCP send to Keepa?

Only the call parameters you'd send if you used Keepa's API directly:
the ASIN list, the marketplace domain code, and the filter parameters
you specified. No conversation transcript, no Claude context, no PII.
Each call goes from your machine directly to `api.keepa.com` over
HTTPS.

### Are my queries logged?

Yes, locally, at `~/.agellic-mcp/logs/agellic-<date>.log` (rotated
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

### Which clients does this work in?

- **Claude Desktop** (macOS, Windows): install via the `.mcpb`
  extension. Drag the file into Settings → Extensions.
- **Claude Code** (macOS, Linux, Windows): install via
  `node install.mjs` after unzipping the release archive.
- **Codex CLI + ChatGPT desktop** (macOS, Linux, Windows): install via
  `node install.mjs --host codex` after unzipping the same archive.
  One command registers both; they share MCP configuration.

agellic is a standard local MCP server, so other MCP-capable hosts can
run it too; the hosts above are the supported install paths.

Linux desktop is not supported for Claude Desktop because CD itself
doesn't ship a Linux build. If/when that changes, the existing `.mcpb`
will work there too.

### Cowork limitations?

None for charts as of v1.7.1: `get_product_chart` now renders inline in
Cowork, the same MCP Apps view regular chat uses. (Releases before
v1.7.1 could only deliver a text readout there.) After upgrading, quit
Claude Desktop fully (Cmd-Q) and reopen so Cowork picks up the new
server. Every other tool works in Cowork as it always has.

## Exchange rates

### How current are the rates in `resolve_cross_border`?

The tool converts prices between marketplaces using an exchange-rate
table baked into the build; each cross-border result reports that table's
date as `summary.exchangeRateDate`. The rates are approximate, fine for
the ballpark price-gap analysis the tool does, but they drift a few
percent over weeks. Every release ships with refreshed bundled defaults.

### Can I update the exchange rates myself?

**Claude Code**, yes, any time:

```bash
node install.mjs --refresh-rates
```

This fetches current ECB reference rates (via Frankfurter) and writes
`~/.agellic-mcp/exchange-rates.json`, which the server uses in preference
to the bundled table. Restart your Claude session afterward. It doesn't
reinstall anything, just the rate file.

**Claude Desktop only**: the `.mcpb` install has no terminal step, so
CD-only users can't run `--refresh-rates`. You get fresher rates by
upgrading to a newer release (refreshed bundled defaults), or by also
installing the Claude Code build on the same machine: the rate file
lives in the shared `~/.agellic-mcp/`, so one refresh covers both hosts.
