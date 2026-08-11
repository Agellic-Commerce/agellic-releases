# Usage examples

A starter set of prompts to try once `agellic-mcp` is installed and
connected. These are not exhaustive: agellic gives Claude twelve tools
and they compose freely. This is a starter set; the per-tool reference
is in [TOOLS.md](./TOOLS.md).

The notes under each prompt describe roughly which tools Claude will
reach for and what the answer typically looks like. Token costs assume
nothing is cached; cached lookups are free for 24 hours.

---

## Discovery: finding products by criteria

When you don't have a list yet and want Claude to surface candidates
from Amazon's full catalog.

> Show me top-selling products in Toys & Games under $25 with at least
> 4 stars on the US marketplace.

Uses `execute_keepa_finder` with a category hint, price ceiling, and
rating floor. Claude returns a one-line summary like "Matched 1,847
products" and asks where you want to take it: narrow further, screen
the set, deep-dive a top slice. ~11-30 tokens.

> Find Anker products under $40 with rating ≥ 4.2 and at most 5
> third-party sellers on the listing.

`execute_keepa_finder` with a brand filter plus competition gate. The
seller-count filter is the useful one here: it surfaces listings that
aren't already crowded. Typically returns a small enough set (tens to
low hundreds) to screen directly. ~20 tokens.

> Find kitchen products on Amazon US where the price has dropped at
> least $10 vs the 30-day average AND BSR is under 50,000.

`execute_keepa_finder` with `delta30_BUY_BOX_SHIPPING_gte` plus a BSR
ceiling. This is a recent-mover query: products that just got
cheaper and are still selling. ~20 tokens.

> Find pet supply products on the US marketplace that are Subscribe &
> Save eligible with rising review velocity over the last 30 days.

`execute_keepa_finder` with the S&S flag and a review-velocity delta.
Subscribe & Save eligibility is a recurring-revenue signal; combined
with accelerating reviews it points at items climbing the rank.
~15-25 tokens.

## Screening a candidate set

When you have ASINs (your own list, or a finder result) and want
Claude to pull a lightweight signal table so you can rank or filter.

> I have 200 ASINs from my supplier, screen them for FBA viability.
> [paste the list]

`screen_products` returns a 13-column pipe table per ASIN: BSR,
estimated monthly sold, landed Buy Box price (item + shipping), trend,
seller count, whether
Amazon is on the listing, lowest FBA offer, referral fee %, recent
price drops, OOS %, brand, and a truncated title. You then ask Claude to
filter or rank by whichever columns matter. ~3 tokens per uncached ASIN;
500 ASINs per call, staged higher across calls. A 200-ASIN screen on cold
cache is ~600 tokens, so on the default 20 TPM Keepa plan it comes back as
a background work order that runs itself as tokens refill, and Claude polls
it and reads rows as they land.

> From those screening results, drop anything where Amazon is on the
> listing or where the BSR is above 100,000.

No new tool call: Claude filters the table in context. Free. This is
where most of the screening value lives: the table is yours to slice.

> Of the candidates that survived, give me deep offer-level detail on
> the top 10 by sold volume.

`get_product_details` on the 10 shortlisted ASINs. Returns the full
per-product picture: individual seller offers, FBA vs FBM split,
stock depth, Buy Box rotation, calibrated demand range, observed
sell-price bands, seasonality confirmation, review velocity, OOS
history, referral fees, IP risk signals. Quoted at 16 tokens per uncached
ASIN (the worst case it authorises), and it typically settles at ~6-8.

## Deep-dive on a shortlist

When you have a handful of ASINs and need offer-level depth.

> Compare these 5 ASINs for Buy-Box stability over the last 90 days:
> B0CJT5D35W, B08N5WRWNW, B09G9FPHY6, B07FZ8S74R, B0ABC12345.

`get_product_details` returns each ASIN's Buy Box rotation table:
dominant seller, win share, unique winner count over the window, plus
a volatility flag in the insights block. Claude reads off who's
stable, who's getting flipped, and whether Amazon is in the mix. Quoted
at 80 tokens cold, typically settling around 30-40.

> For ASIN B0CJT5D35W, what's the calibrated monthly sales estimate
> and how confident is the model in it?

`get_product_details` on a single ASIN. The `demand` block returns a
range (low / likely / high) with a confidence label
(`high`/`medium`/`low`) and a `mode` field saying where the answer came
from: `read` (the model's estimate), `badge` (Amazon's own "X+ bought
past month" figure, which suppresses the model), or `no-read` (nothing
usable, with a reason). This is a range estimate, not a forecast: the
confidence and mode matter as much as the number. Quoted at 16 tokens,
typically ~6-8.

> Look up the ASIN for UPC 012345678905 on Amazon US.

`get_product_details` in code-lookup mode. Returns up to 3 candidate
ASINs with title + brand for disambiguation. 3 tokens. This is
identification only: no offers / stock / rating data; for that, take
the resolved ASIN and call again with `asins`.

## Cross-marketplace arbitrage

When you want to know if products are priced differently across
Amazon regions.

> Are any of my US bestsellers cheaper to source from the UK
> marketplace? Here are the ASINs: [paste 30 ASINs]

`resolve_cross_border` from US (domain 1) to UK (domain 2). Returns a
table sorted by % gap descending: positive gap means the UK price is
lower, after exchange-rate conversion to USD. Each match carries a
confidence label (high / medium / low) based on product-code overlap
and title similarity. Worst case 12 tokens per source ASIN; cached
products cost 0.

> Compare these 40 ASINs between Amazon US and Amazon DE. I want to
> know which ones have a margin gap of 15% or more either direction.

Same tool, US (1) → DE (3). Claude reads off the matches with
`|gapPercent| >= 15` in the result. The result is also stored under
an `xb_` id for 7 days: you can ask Claude to re-read it later
without re-billing.

## Price and BSR history

When you want to see the picture, not the numbers.

> What does the price and BSR history of B0CJT5D35W look like over the
> last 90 days?

`get_product_chart` returns a PNG with the default reseller view:
BSR, lowest new price, Buy Box price, and lowest FBA offer, all
overlaid on a single chart at 800×400. 1 token. Claude Desktop
(regular chat and Cowork), Claude Code, and the ChatGPT desktop app
(with the opt-in MCP Apps flag, see INSTALL.md) all render the chart
inline.

> Pull the BSR-only chart for the same ASIN at 365 days. I want to
> see if there's seasonality.

Same tool, `rangeDays: 365`, other curves toggled off. The longer
window makes annual patterns (Q4 ramps, summer dips, back-to-school
spikes) visible. 1 token.

## Market sizing and insights

When the question is about the shape of a market, not specific
products.

> I want to size the Anker accessories market on Amazon US: products
> under $40 with BSR under 100,000. What's the average Buy Box price,
> how many sellers per listing, and how often is Amazon on these
> listings?

Two-step. First call: `execute_keepa_finder` filters-only to learn
the match count. Second call: same filters plus `includeStats=true`,
which returns the `searchInsights` summary: average landed Buy Box
price (item + shipping, the same amount Amazon charges its referral fee
on), median seller count, the share of listings where Amazon is the Buy
Box winner, brand fragmentation across the match set, FBA share,
average rating, average review count. The stats call costs `30 + ⌈
totalResults / 1M⌉` tokens; refine first if the count is very large.

> In Pet Supplies on US, what's the typical out-of-stock rate and
> seller count on listings with BSR under 25,000?

Same pattern: finder filters narrow it down, then `includeStats`
reads the shape. Useful before you commit to sourcing in a category
you haven't worked before: high OOS rates and thin seller counts mean
opportunity; low OOS and 20+ sellers per listing usually mean the
category is saturated.

---

## Things to know before you start

- **Tokens are Keepa's currency, not ours.** Your Keepa subscription
  refills tokens on a per-minute schedule (20 TPM on the default
  plan). Run `check_token_balance` any time to see what you have.
- **A short balance is not an error.** Every finder, screen, deep-dive,
  cross-border, or code-resolution call becomes a durable **work order**.
  If the tokens are there, you get the answer inline. If they aren't, the
  order is accepted anyway and runs itself as tokens refill, and Claude
  polls it with `check_job_status` and reads rows as they land. If the work
  would take more than about an hour of refill, you get a quote first and
  nothing is charged until you say yes. See
  [TOOLS.md](./TOOLS.md#every-call-is-a-work-order).
- **Ask for the ETA, and read it as a bound.** A background order reports
  how many orders are ahead of it and a "done within ~X" figure that
  recomputes and counts down every time you check. It assumes no other
  work is running on your Keepa key, and its wall-clock estimate assumes
  the app is open about 8 hours a day, so an always-open session finishes
  roughly 3× sooner.
- **Work orders run on your machine, not the cloud.** A queued
  screen or cross-border run drains as tokens refill, but only while a
  Claude app stays open. Quit Claude (or let the machine sleep) and the
  order pauses; relaunch and it resumes where it left off, nothing lost.
  Kick off the big one, leave Claude running, come back to results. You
  can stop one at any point, and everything it already settled stays
  readable.
- **The 24-hour cache is real.** Re-asking the same question later the
  same day is usually free, and the cache is shared across chats and
  both Claude apps, so a fresh conversation re-reads it for nothing.
- **Claude won't auto-chain expensive steps.** A finder result won't
  silently roll into a 500-ASIN screen. Claude waits for you to ask.
  Compound asks in a single turn ("find X and screen them") work as
  expected.
- **ASINs aren't globally unique.** The same ASIN on US and DE can
  be different products. For marketplace-to-marketplace comparison,
  use the cross-border prompts above: they verify matches via
  product codes and title similarity rather than trusting the ASIN.
