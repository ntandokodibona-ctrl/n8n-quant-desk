# Build Log — n8n Quant Desk

A record of what was decided and what actually went wrong while building
this, kept because the troubleshooting is as much a demonstration of the
underlying skills as the finished workflow is.

## 1. Environment setup

**Decision:** self-host n8n via Docker rather than use n8n's paid cloud
plan. Self-hosting is n8n's own free tier, not a workaround — the paid
plan pays for *them* to host and manage it, not for the software itself.

**Issue:** Docker's HTTPS requests were failing with
`SELF_SIGNED_CERT_IN_CHAIN`. Cause: Kaspersky's "Encrypted connections
scan" feature was intercepting HTTPS traffic and substituting its own
certificate, which the container didn't trust.

**Fix path (took several attempts):**
- First tried excluding `Docker Desktop.lnk` (a shortcut) in Kaspersky's
  trusted applications — didn't work, since a shortcut isn't the actual
  process handling network traffic.
- Found the real executable at
  `C:\Users\<user>\AppData\Local\Programs\DockerDesktop\Docker Desktop.exe`
  and excluded that instead, with "do not scan all traffic" enabled.
- Also added `api.groq.com` under Kaspersky's trusted domains as a second,
  narrower layer of exclusion.
- Both together resolved it. Takeaway: excluding a shortcut file is not
  the same as excluding the program it points to.

## 2. Model availability

**Issue:** the first working chat model, `llama3-8b-8192`, had been
retired by Groq mid-project. n8n's error message said the model "does not
exist," which read like a configuration bug but was actually accurate.

**Fix:** checked Groq's own Playground UI directly rather than trusting
written documentation, since model catalogs on fast-moving providers
change faster than docs get updated. Found `openai/gpt-oss-120b` was
live and working on the account, and used that for the rest of the build.

**Recurring sub-issue:** every new Chat Model node in n8n defaults to the
retired `llama3-8b-8192` model, and the Model field is a "resource
locator" type that needs switching from "Fixed" to "Expression" mode
before it accepts a typed value. This had to be repeated for each of the
three agent nodes built.

**Recurring sub-sub-issue:** several of these model-field edits initially
failed with "the model does not exist" even when the typed name was
correct, due to invisible leftover characters (a stray quote mark once,
a trailing space another time) left over from copy-pasting over the old
value. Fix each time was to select-all and delete before retyping, rather
than editing around the old text.

## 3. Language choice for the metrics node

**Original plan:** compute metrics in Python, to keep the "Python
reinforcement" goal of the sprint embedded in the build.

**Issue:** n8n's Python (Beta) Code node mode requires an actual Python 3
interpreter inside the container. The official n8n Docker image doesn't
ship with one. Running it threw "Python runner unavailable" and pointed
to setting up a separate external Python runner service — out of scope
for a two-week sprint.

**Decision:** switched the metrics logic to JavaScript. Logged explicitly
as a scope trade-off rather than silently treated as if the plan hadn't
changed — the Pico project stayed 100% Python to preserve the original
goal elsewhere in the sprint.

## 4. Data pipeline structure

**Issue:** initial single-ticker test used `outputsize=30` (30 days),
which is too short a window for a meaningful 50/200-day SMA. Bumped to
`outputsize=200` once the pipeline was proven end-to-end on one ticker.

**Scaling to 5 tickers:** rather than duplicating the HTTP + Code node
pair five times, a small Code node generates five ticker items up front,
and both downstream nodes process each item automatically (HTTP Request
does this by default; the metrics Code node needed its Mode switched from
"Run Once for All Items" to "Run Once for Each Item" to do the same).

**Sub-issue:** n8n's per-item Code node mode expects a single returned
object (`return { json: result }`), not an array (`return [{ json:
result }]`) — the array-wrapped form is only correct in "all items" mode.
Mixing the two throws `A 'json' property isn't an object`.

## 5. The AI Agent data-loss bug

**Issue:** n8n's AI Agent node returns only its own `output` field on
each item, silently discarding every other field that was present before
it (ticker symbol, volatility, drawdown, etc.). This wasn't visible until
the Risk agent started receiving blank tickers and "insufficient
volatility data" errors.

**Fix:** the parsing node placed between the Analyst and Risk agents
explicitly re-fetches the original metrics by referencing the metrics
node directly (`$('Compute Metrics').item.json`), rather than assuming
the data survived the trip through the agent. This is now documented in
the README as a deliberate architectural note, since it's the kind of
thing that produces silently wrong numbers if missed.

## 6. Outcome

End-to-end pipeline: ticker generation → real price data → deterministic
metrics → Analyst interpretation → Risk sizing → combined report, tested
successfully across all 5 tickers with real, differentiated output per
ticker (not just boilerplate repeated 5 times).
