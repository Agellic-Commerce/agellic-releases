# Troubleshooting

First-install behavior, log locations, and the handful of things most
likely to go sideways on your first run of agellic-mcp v1.1.0.

## 1. First-install behavior: what to expect

A brand-new install on a fresh machine boots the server in
**configuration-pending mode**. This is normal and expected. Here's what
you should see on each host:

### Claude Desktop

After you drag `agellic-mcp.mcpb` into Settings → Extensions:

1. The credential form appears asking for your **license token**, your
   **Keepa API key**, and a **tokens-per-minute** value (default `20`).
2. The extension shows as **connected**, no red banner, no error icon.
3. Exactly **one tool** is visible in the tool list: `_configure_agellic`.
   Ask Claude "what tools do you have?" and it will surface this
   placeholder along with a description of what to enter and where.
4. Once you save the credential form, Claude Desktop auto-restarts the
   extension. After ~2-5 seconds the placeholder is replaced with the
   full **11-tool** set and you're ready to use agellic.

You should **not** see a red `Agellic could not find a valid license`
banner on a brand-new install. If you do, the configuration-pending
detection didn't fire. See [error #1](#1-red-could-not-find-a-valid-license-banner-on-a-brand-new-install)
below.

### Claude Code

After you `unzip agellic-mcp.zip` and run `node install.mjs`:

1. The installer prompts for your **license token**, **Keepa API key**,
   and **tokens-per-minute** interactively.
2. You can skip the prompts by passing flags:
   ```
   node install.mjs --license <token> --keepa-key <key> --tpm 20
   ```
3. The installer probes the server with your credentials before writing
   the MCP config entry, if something is wrong (bad license, bad Keepa
   key) you'll see the error immediately, not on first tool call.
4. There is no placeholder mode on Claude Code: either the install
   completes with all 11 tools available, or it fails with a specific
   error you can act on.

## 2. Log locations

There are two logs worth knowing about. Claude Desktop writes its own
per-server log, and agellic writes its own server-side log too.

### Claude Desktop's per-server log

- **macOS:** `~/Library/Logs/Claude/mcp-server-agellic.log`
- **Windows:** `%APPDATA%\Claude\Logs\mcp-server-agellic.log`

This captures stderr from the server process plus Claude Desktop's own
view of the MCP handshake. Useful for connection-level issues.

### agellic's own log (per-day rotation)

- **macOS / Linux:** `~/.agellic-mcp/logs/agellic-<YYYY-MM-DD>.log`
- **Windows:** `%USERPROFILE%\.agellic-mcp\logs\agellic-<YYYY-MM-DD>.log`

This is the server-side log with full request/response tracing, token
bucket accounting, and credential-resolution diagnostics. The server
prints the active log path to stderr on boot, so the first few lines of
the Claude Desktop log will tell you exactly where to look.

**Both logs redact license tokens and Keepa API keys.** You can paste
log excerpts into a support email without leaking secrets.

### Log volume controls

Two environment variables raise the log ceiling:

- `AGELLIC_LOG_LEVEL`: one of `error | warn | info | debug`
  (default `info`).
- `AGELLIC_LOG_DEBUG`: comma-separated list of module names to
  elevate to debug, e.g. `tool-call,token-bucket`. Useful when you
  want debug detail from just one subsystem without the firehose.

On Claude Desktop, set these in the credential form's "Advanced" pane
(if exposed) or in your system environment before launching CD. On
Claude Code, set them in the MCP server `env` block in your
`~/.claude.json` or the equivalent CC config file.

## 3. Common errors

### 1. Red "could not find a valid license" banner on a brand-new install

**Symptom:** You just installed the `.mcpb` for the first time, opened
the credential form, and Claude Desktop is showing a red error banner
saying `Agellic could not find a valid license` before you've entered
anything.

**Cause:** The configuration-pending detection didn't fire. The server
handles a Claude Desktop quirk where unsubstituted `${user_config.X}`
placeholders are passed as **literal text** (not empty string) when the
credential form is blank. This is defended against in current code.

**Remedy:** Open `~/Library/Logs/Claude/mcp-server-agellic.log` (macOS)
or `%APPDATA%\Claude\Logs\mcp-server-agellic.log` (Windows) and search
for `first-install-detected`. If that line is missing, the defense
didn't engage on your machine. File an issue with the log excerpt
attached.

### 2. License or Keepa key is mangled when pasted into Claude Desktop

**Symptom:** You pasted your license token cleanly from the purchase
email, but the server rejects it with a signature-mismatch error. Or
your Keepa key works on the Keepa dashboard but fails here.

**Cause:** Claude Desktop's credential form has two paste-time
mutations that affect long string values:

- **Fields marked `sensitive: true`** (license token, Keepa key),
  values get soft-wrapped at roughly 76 columns, with `\n` characters
  inserted at the wrap points.
- **Fields marked `sensitive: false`** (rarely used here, but possible
  for future settings), newlines are stripped from pasted multi-line
  content, which destroys things like PEM-encoded keys.

Both are upstream Claude Desktop bugs, not agellic bugs. The server
defends against both: tokens and API keys are whitespace-stripped at
the env-read boundary before any signature validation runs.

**Remedy:** Most paste-time mangling is silently corrected by the
server. If you're still seeing a `signature does not match this build`
error after a clean re-paste, the defense may not have engaged on your
machine. Capture the log and file an issue.

### 3. `AGELLIC_LICENSE signature does not match this build`

**Symptom:** Server fails to boot, log says
`AGELLIC_LICENSE signature does not match this build`.

**Cause:** Your license token was minted for a different build of
agellic-mcp than the one you're trying to run. This usually means
you're on an older license + newer build (or vice versa).

**Remedy:** Email `support@agellic.com` with your purchase email
address. We'll re-mint the license against the build you're running.

### 4. `AGELLIC_LICENSE has expired` or `covers versions released before …`

**Symptom:** Server fails to boot, log says either
`AGELLIC_LICENSE has expired` or
`AGELLIC_LICENSE covers versions released before <date>`.

**Cause:** Either your subscription has lapsed (expired), or your
subscription doesn't include the release window of the build you're
trying to run (version not covered).

**Remedy:** Renew via the link in your original purchase email, or
email `support@agellic.com` if you can't find it.

### 5. `AGELLIC_MCP_TOKENS_PER_MINUTE=… is below the floor of 20`

**Symptom:** Server refuses to boot, log says your configured TPM
value is below the supported minimum.

**Cause:** Your Keepa subscription tier doesn't allocate enough
tokens-per-minute to sustain agellic's tool surface. The floor is `20`
because Keepa's free/inactive-account refill rate is too low for the
server to make useful progress: most multi-ASIN workflows would never
finish on a sub-20 TPM bucket.

**Remedy:** Upgrade your Keepa subscription to a tier that supports at
least 20 TPM, or, if your tier actually supports a higher value but
you've configured TPM too low, set the env var explicitly to a higher
number. The default `20` is the floor, not the cap.

### 6. Token bucket exhausted: `wait <N> minutes`

**Symptom:** A tool call returns a `wait <N> minutes` error or you see
`token-bucket: exhausted` in the agellic log.

**Cause:** You've burned through your per-minute Keepa token allotment.
The bucket refills at `AGELLIC_MCP_TOKENS_PER_MINUTE` per minute
(default `20`).

**Remedy:** Wait the indicated number of minutes for the bucket to
refill, or raise the ceiling by setting `AGELLIC_MCP_TOKENS_PER_MINUTE`
to match your real Keepa subscription tier.

For larger lookups (more than 100 ASINs), agellic queues a background
job rather than blocking on the bucket. Poll with `check_job_status`
and the result will be ready when the bucket has refilled enough to
process it. You don't need to manually re-issue the request.

### 7. Claude Desktop: 11 tools don't appear after install

**Symptom:** You installed the `.mcpb`, the extension shows as
connected, but the tool list shows fewer than 11 tools (or only
`_configure_agellic` persists after you've saved credentials).

**Cause:** Claude Desktop only re-reads its extension's tool list on
cold start. A window close (Cmd+W on macOS, or the X button on Windows)
doesn't count: the app process is still running.

**Remedy:** Fully quit Claude Desktop (**Cmd+Q on macOS**, or right-click
the tray icon → Quit on Windows) and reopen. The full 11-tool set will
appear.

### 8. Cowork shows charts as text-only

**Symptom:** You asked for a product price chart via Cowork (Claude
Desktop's agent-mode surface) and got a text summary plus the Keepa URL
but no rendered image. The same tool call from regular Claude Desktop
chat or Claude Code renders the chart fine.

**Cause:** Cowork runs the agent inside a sandboxed VM that (a) doesn't
paint inline `type: 'image'` content blocks for the user and (b) blocks
reads of files written outside the session's allowed directories, so a
host-saved chart PNG can't be reached either. The chart is generated
successfully and the model still receives the image for analysis; only
the inline display to the user is unavailable on this surface.

**Remedy:** None on the agellic side. This is a Cowork sandbox
constraint, not an agellic bug. In Cowork, use the data readout plus the
Keepa product URL the tool returns (pair the chart request with
`get_product_details` for a fuller picture). When you want the rendered
chart, use regular Claude Desktop chat or Claude Code.

### 9. A background job is stuck at `pending` or `running`

**Symptom:** A tool call returned a `jobId` (a large screen, cross-border
run, or finder), but `check_job_status` keeps reporting `pending` or
`running` and the result never lands.

**Cause:** Two normal, non-error reasons, usually one of:

- **Waiting for tokens.** A big job needs the Keepa bucket to refill
  enough capacity before it can run. On the default 20 TPM plan a
  500-ASIN screen (~1,500 tokens) takes several refill cycles. The job
  stays `pending` until the bucket can cover it.
- **The host isn't running.** The job runner lives inside the MCP server
  process, which only runs while Claude Desktop or Claude Code is open.
  Quit Claude or let the machine sleep/shut down and the queue stops
  making progress: there is no cloud worker.

**Remedy:** Leave a Claude app open and the machine awake; the job drains
on its own as tokens refill. If you did quit, nothing is lost: the job's
lease is reclaimed on the next launch and it resumes where it left off.
Poll with `check_job_status`. There's no `cancel`; to abandon in-flight
jobs, fully restart the MCP host.

