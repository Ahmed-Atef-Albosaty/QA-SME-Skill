# Driving the crawl via the official Playwright MCP server

`qa-review` drives a real, inspectable browser through Microsoft's official **Playwright MCP** server
(`@playwright/mcp`) - you call the MCP browser tools directly, turn by turn, so the session is visible and
steerable rather than a black-box batch run. This is the same server quick mode uses (see
`references/playwright-quick-mode.md`); classic mode just points it at the whole platform instead of one page.

**Core interaction model: accessibility-snapshot + stable `ref`, not raw CSS/XPath locators.**
`browser_snapshot()` returns the page's accessibility tree as a YAML-like block - roles, accessible names,
and a short stable id per interactive element, e.g. `textbox "Email address" [ref=e20]`,
`button "Sign in" [ref=e29]`. Every action tool (`browser_click`, `browser_type`, `browser_fill_form`,
`browser_hover`, `browser_select_option`, `browser_drag`) takes that id via a param literally called
**`target`** (e.g. `browser_click({target: "e29", element: "Sign in submit button"})`) - `element` is an
optional human-readable label for the action, `target` is the actual ref (or, per the tool's own
description, "a unique element selector" if you don't have a fresh ref). There is no manual CSS/XPath
selector-guessing step the way Selenium's `{by, value}` locator model required. **Call `browser_snapshot()`
first on every new screen** to get current refs; refs regenerate after DOM changes, so re-snapshot after a
navigation or a click that changes the page before targeting anything new. **Confirmed live: every action
tool's own response already includes a fresh Page summary (URL, title, console error/warning count) and
either an inline snapshot or a snapshot-file reference** - often enough to confirm the action landed without
a separate follow-up call at all.

**No `start_browser`-equivalent tool exists.** The browser session is implicit - it launches on first
`browser_navigate` call, using whatever device/viewport/timeout flags the server process itself was started
with (see "One-time setup" and "Parallel setup" below). There's nothing to call to "start" a session beyond
just navigating.

**Two JS-eval tools exist - use the plainer one by default.** `browser_evaluate({function: "() => {...}" or
"(element) => {...}", target?, element?})` runs a JS function in-page (optionally scoped to one element via
`target`) and is NOT flagged RCE-equivalent - use this for the vast majority of DOM reads (a link scrape, a
TreeWalker text-node walk, a `getBoundingClientRect()` read). `browser_run_code_unsafe({code: "async (page)
=> { ... }"})` is a different, more powerful tool that runs against Playwright's own `page` object (real
Playwright API calls, not just a DOM script) and IS flagged RCE-equivalent (`--caps=devtools`) - reserve it
for the rare case you need Playwright-level APIs `browser_evaluate` genuinely can't reach, not as the default
JS-eval tool.

**File writes (screenshots, HTML dumps, storage-state files) are sandboxed to the project's working
directory - confirmed live.** A `browser_take_screenshot`/`browser_storage_state` call with a path outside the
project root (e.g. `/tmp/...` or a bare `~/...` path) fails with "File access denied: ... is outside allowed
roots." Always use a path relative to (or physically inside) the project's cwd - e.g.
`qa-runs/<slug>/screenshots/...`, not `/Users/<you>/.pw-state/...`. This changes the storage-state path
convention below - see "Parallel setup."

## Speed - trust auto-waiting, don't hand-roll waits
Every Playwright MCP action tool inherits Playwright's own actionability auto-waiting: before acting, it
waits (up to `--timeout-action`) for the target to be visible, stable (unchanged position/size for 2+ frames),
enabled, and able to receive pointer events. This eliminates almost all of the manual "settle-check via script,
then one retry" discipline older locator-based tools needed. The decision tree for the rest of this run:

**Confirmed live gotcha: `--timeout-action` also covers waiting for a click-triggered navigation to settle,
not just pre-click actionability - and a timeout here does NOT mean the click failed.** A login-submit
`browser_click` on a cold-loading page threw `TimeoutError: ... waiting for scheduled navigations to finish`
at the configured `--timeout-action=3500` budget - but the tool's own log showed `click action done` right
before that wait, and a follow-up `browser_snapshot()` confirmed the login had genuinely succeeded server-side
(the post-login nav rendered correctly). **Never blindly retry a click after this specific timeout shape**
(actionability/click steps completed, only the post-click navigation-settle wait exceeded budget) - retrying
a non-idempotent action (a form submit, an Add/Create button) risks a real duplicate submission. Instead:
check current state first (`browser_snapshot()` or the URL) to see whether it already went through, and only
retry the actual click if it clearly didn't. This is more likely on submit-style buttons on a slow/cold-loading
target than on plain same-page clicks - if it recurs often against one platform, raise that path's
`action_ms` via the platform spec's `slow_endpoints` config rather than tolerating the timeout every time.

1. **Default: call the action tool directly, nothing else.** `browser_click({target})`,
   `browser_type({target, text})`, `browser_fill_form({fields: [{target, name, type, value}, ...]})`,
   `browser_select_option({target, values})` all auto-wait internally. Don't add a manual check before or
   after most of the time - this is the single biggest simplification versus the old locator-based
   discipline, and defaulting to it (not a habitual "just in case" check) is what keeps a run fast. Confirmed
   live: a login-submit click that also triggers a navigation returned with the post-navigation URL already
   in its own response - no separate wait was needed at all.
2. **After a navigation, or a click that triggers a route/content change you're about to assert on:**
   `browser_wait_for({text: "<distinctive marker text>"})`. Use a marker string that's genuinely distinctive
   to the destination state (a heading, a label unique to that screen) - check the platform spec's
   `## Playwright QA Config` `ready_markers` map (see SKILL.md Step 1) for a pre-known marker before guessing
   one live.
3. **When something should have disappeared** (a modal closed, a spinner gone, a deleted row no longer
   listed): `browser_wait_for({textGone: "<marker>"})`. This is a real capability gap the old locator-based
   tooling never had at all - use it rather than a poll-and-recheck loop.
4. **For any PASS/FAIL verdict on presence** (a criteria-sheet check item): prefer
   `browser_verify_text_visible({text})` or `browser_verify_element_visible({role, accessibleName})`
   (`--caps=testing`) over a raw snapshot read - both auto-retry and give a clean result instead of a DOM
   dump you have to interpret yourself. Confirmed live: `browser_verify_element_visible` takes `role` +
   `accessibleName` (read straight off a snapshot line like `button "Sign in" [ref=e29]` -> role `button`,
   accessibleName `Sign in`), not a `ref` - it re-derives the element by role/name itself rather than reusing
   a snapshot ref.
5. **Fixed `browser_wait_for({time: N})` is a last resort - only these two cases:**
   - A page whose content streams/paints progressively with no single stable "ready" marker (check the
     platform spec's `slow_endpoints` list first - a documented slow path should already carry its own longer
     `--timeout-action` override rather than needing an ad hoc sleep).
   - The CDN/embed bot-detection re-check (`SKILL.md` Step 4b) which explicitly needs a real 2-5s wait on a
     second, different item before concluding a block is persistent rather than transient.
   Reaching for a fixed-time wait anywhere else is exactly the naive-sleep pattern this migration removes -
   don't default to it out of caution.
6. **Batch independent calls into one message** where there's no ordering dependency (e.g. a `browser_snapshot`
   alongside a `browser_console_messages` read of the same already-loaded page) - unchanged good practice from
   before, still worth doing.
7. **`browser_console_messages`/`browser_network_requests` are on-demand, not routine.** Both default to
   "since last navigation" (a real improvement - Playwright resets this scope automatically, unlike the old
   tool's fully-accumulating log), and `all: true` pulls full history if you need it. Only call either when
   actively investigating a specific suspected dead click, console error, or failed request - not as a
   reflexive step after every screen. `browser_console_messages` takes a `level` param (error/warning/info/
   debug, each including more-severe levels) rather than a filter list. `browser_network_requests` takes a
   `filter` param (a regexp string matched against the URL, e.g. `"/api/.*user"`) and a `static` boolean
   (default `false`, excludes fonts/images/scripts unless set `true`) - use `filter` to narrow to the request
   you actually care about instead of scanning a full unfiltered list; get one request's full headers/body via
   the separate `browser_network_request({index})` tool using the number `browser_network_requests` printed.
8. **`browser_evaluate` is the default JS-eval tool; `browser_run_code_unsafe` is a different, more powerful,
   scoped-use tool, not a general-purpose one.** `browser_evaluate({function: "() => {...}", target?,
   element?})` runs a plain JS function in-page (optionally scoped to one element via `target`) and is NOT
   RCE-flagged - use it for the vast majority of DOM reads. `browser_run_code_unsafe({code: "async (page) =>
   {...}"})` runs against Playwright's own `page` object (real Playwright API calls) and IS flagged
   RCE-equivalent (`--caps=devtools`) by its own naming - reach for it only for the handful of things
   snapshot+ref tools and `browser_evaluate` genuinely can't do: a raw DOM text-node walk for the CSS
   `text-transform` trap (see
   below), a `getBoundingClientRect()` read for the fleeting-bug annotation technique, or a stubborn
   React-controlled input that `browser_fill_form`/`browser_type` somehow doesn't register with (rare - try
   the normal fill tools first every time).

## One-time setup
Check the server is connected: `claude mcp list` should show
`playwright: npx @playwright/mcp@latest ... - ✔ Connected`.
If missing, add it with the full capability set this skill needs:
```bash
claude mcp add playwright -- npx @playwright/mcp@latest --caps=storage,testing,devtools
```
- `storage` exposes cookie/localStorage/sessionStorage tools plus `storage_state` save/restore - keep this cap
  on for every launch, it's how Phase 0's pre-authenticated bootstrap (see "Parallel setup" below) works.
- `testing` exposes the auto-retrying `browser_verify_text_visible`/`browser_verify_element_visible` assertions
  used throughout the Speed section above.
- `devtools` exposes `browser_run_code_unsafe`, `browser_highlight`/`browser_hide_highlight` (bug-evidence
  annotation - see `SKILL.md` Step 5), and tracing/video tools if a run wants richer evidence than screenshots
  for a hard-to-explain bug.
A freshly-added MCP server's tools only appear in a **fresh session** - if you just added it, tell the user to
restart/reload before the `mcp__playwright__*` tools are callable.

## Load the tool schemas
The browser tools are deferred. Before the first call:
```
ToolSearch({query: "select:mcp__playwright__browser_navigate,mcp__playwright__browser_click,mcp__playwright__browser_type,mcp__playwright__browser_fill_form,mcp__playwright__browser_select_option,mcp__playwright__browser_hover,mcp__playwright__browser_drag,mcp__playwright__browser_press_key,mcp__playwright__browser_file_upload,mcp__playwright__browser_wait_for,mcp__playwright__browser_handle_dialog,mcp__playwright__browser_tabs,mcp__playwright__browser_snapshot,mcp__playwright__browser_find,mcp__playwright__browser_take_screenshot,mcp__playwright__browser_console_messages,mcp__playwright__browser_network_requests,mcp__playwright__browser_verify_text_visible,mcp__playwright__browser_verify_element_visible,mcp__playwright__browser_highlight,mcp__playwright__browser_hide_highlight,mcp__playwright__browser_run_code_unsafe", max_results: 25})
```
(In parallel mode, substitute the assigned server prefix - `mcp__playwright1__*` etc. - for every name above.)

## Parallel setup (only needed if 3-worker parallel mode was chosen - see `SKILL.md` Step 1b)
Playwright MCP is still **single-active-session-per-server** in the current stable release (a `sessionId`
multi-session proposal exists upstream but isn't merged) - so, same as before, **3 personas need 3 separate
server processes**, each with its own `--device` preset:
```bash
claude mcp add playwright1 -- npx @playwright/mcp@latest \
  --caps=storage,testing,devtools \
  --device="Desktop Chrome" --viewport-size=1920,1080 \
  --timeout-action=3500 --timeout-navigation=45000

claude mcp add playwright2 -- npx @playwright/mcp@latest \
  --caps=storage,testing,devtools \
  --device="iPad Pro 11" \
  --timeout-action=3500 --timeout-navigation=45000

claude mcp add playwright3 -- npx @playwright/mcp@latest \
  --caps=storage,testing,devtools \
  --device="iPhone 15" \
  --timeout-action=3500 --timeout-navigation=45000
```
`--device` is the real improvement over the old manual `--window-size`+UA-string hack: it sets viewport, device
pixel ratio, touch capability, and user agent all atomically from Playwright's own device descriptor table -
no floor, no partial spoofing. Recommended trio: **Desktop = `"Desktop Chrome"` @1920x1080, Tablet =
`"iPad Pro 11"`, Mobile = `"iPhone 15"` (393x852 - a real device, not the old fake 500x800 floor)**. Confirm
the exact device name strings against the running server's own device list on first use each session (they're
Playwright's own presets and occasionally get renamed/added upstream) rather than assuming these three names
are permanently exact - see the "confirmed live" note at the end of this file for how to do that once and
reuse the result. **Confirmed live: with no `--device`/`--viewport-size` flag at all, a session defaults to a
Retina-scaled desktop-ish window (~1900x1012, DPR 2)** - not a clean 1920x1080 - so always pass an explicit
device/viewport flag rather than relying on the default.

**Prefer pre-authenticated state over a live login every run - confirmed-live method, not a launch flag.**
Official docs describe an `--isolated`/`--storage-state=<file>` pair of CLI launch flags for this; this
session only independently confirmed the equivalent **runtime tool calls** `browser_storage_state({filename})`
(save) and `browser_set_storage_state({filename})` (restore, clears existing cookies/localStorage first) -
use those as the confirmed method unless you've separately verified the CLI flags work on the server version
in use. **Critical: file paths for both tools are sandboxed to the project's working directory - confirmed
live** (a path like `/Users/<you>/.pw-state/...` or `/tmp/...` fails with "File access denied ... outside
allowed roots"). Use a project-relative path instead, e.g. `qa-runs/<slug>/.pw-state/<persona>.json`.

Capture each persona's authenticated state once per run session, then boot all 3 workers pre-authenticated:
1. Do one plain login first, per persona, exactly like "Per-role login" below.
2. Save that session's state: `browser_storage_state({filename: "qa-runs/<slug>/.pw-state/<persona>.json"})`.
3. Relaunch that persona's worker server with `--isolated --storage-state=<that file>` if confirmed working
   on your server version, otherwise have the worker call
   `browser_set_storage_state({filename: "<that file>"})` as its very first action after `browser_navigate`.
This sidesteps two real risks a live login carries on every run: a flaky/500ing auth endpoint blocking a
whole worker's run, and the "logging into account B replaces account A's session in the same profile" gotcha
still true of shared-cookie auth providers regardless of which browser driver is in front of them.
**Storage-state expires** (auth tokens have a lifetime) - if a worker's first `browser_snapshot()` after
booting from a state file still shows the login form, the state expired; fall back to a live login for that
one worker and re-mint its file, rather than treating it as a hard dependency.

Each persona-worker's agent prompt must scope it to exactly one server's tools (`mcp__playwright1__*` /
`mcp__playwright2__*` / `mcp__playwright3__*`) and must never reference another server's tool prefix - see
`references/parallel-mode.md` for the full worker design.

## Per-role login
1. If a storage-state bootstrap (above) already covers this persona, skip straight to step 5 - the session
   is already authenticated once restored.
2. Otherwise: `browser_navigate({url: base_url + "/auth"})` (or whatever the platform's login path is).
3. `browser_snapshot()` to find the email/password fields and submit control by their accessible name/role -
   this is almost always enough (e.g. a snapshot line like `textbox "Email address" [ref=e20]`,
   `textbox "Password" [ref=e23]`, `button "Sign in" [ref=e29]`). If a field genuinely has no useful
   accessible name, fall back to `browser_evaluate` to read the raw HTML and locate it manually - don't guess
   a third time.
4. `browser_fill_form({fields: [{target: "e20", name: "Email address", type: "textbox", value: email},
   {target: "e23", name: "Password", type: "textbox", value: password}]})` - one call for both fields, no
   ordering dependency between them.
5. `browser_click({target: "e29", element: "Sign in submit button"})` - auto-waits for it to be actionable,
   then submits. Confirmed live: a submit click that also redirects returns with the post-navigation URL
   already reflected in its own response - no separate settle-check needed.
6. `browser_wait_for({text: "<a marker unique to the logged-in state, e.g. the app's own dashboard heading>"})`
   confirms login succeeded without a manual settle-check/retry dance. If the platform spec's
   `## Playwright QA Config` doesn't yet have a `ready_markers._default` entry, capture one now and write it
   back per the endpoint-discovery write-back convention below.
7. **Logging into a second role/test account in the same browser profile replaces the first account's session
   everywhere** - this is a fact about how the auth provider stores session state (localStorage/cookies), not
   a tooling quirk, and it applies exactly the same regardless of which browser MCP is driving the session. If
   you need to briefly check something as a different role mid-run in a shared (non-isolated) profile, expect
   to explicitly log back in as the original role afterward - or better, avoid the whole problem by using
   `--isolated` sessions per persona in the first place (see "Parallel setup").

## Discovery pass (SKILL.md Step 2a) - lightweight, always runs first
**This pass always runs, in full, regardless of whether the run ends up full or partial QA** - it's what
builds the screen inventory Step 2b's scope choice is presented from. Deliberately cheap: one navigation + one
lightweight read per screen, one browser session for the whole pass (Desktop device preset is fine here - no
per-viewport sessions yet, that's Step 2c).

For each URL discovered (via the endpoint-discovery techniques below, or the platform spec's list):
1. `browser_navigate({url})`.
2. `browser_snapshot()` gives you the page's structure directly (headings, nav, interactive elements) - this
   alone is usually enough to log a short descriptive name and map the screen to an SOP area or Custom. For a
   same-origin link scrape to catch further routes, `browser_evaluate({function: "() =>
   [...document.querySelectorAll('a[href]')].map(a => a.href)"})` - **issue both calls in one message**, they're
   independent reads of the same already-loaded page.
3. Record the screen in the inventory: URL/path, short name, which of the 8 SOP areas it maps to (or
   **Custom** if none), role required to view it.
Do **not** launch tablet/mobile sessions, take screenshot files, or pull console/network diagnostics in this
pass - that's all deferred to Step 2c and scoped to whichever screens end up in scope after Step 2b.

The endpoint-discovery techniques under "Endpoint discovery" below (spec list, sitemap, bundle inspection,
backend schema query, tab/panel enumeration) are exactly what drives this pass - read that section before
starting.

## Per-screen capture (SKILL.md Step 2c) - full capture, only for in-scope screens
Each of the 3 device-pinned sessions (classic mode: one session, cycling through Desktop/Tablet/Mobile device
presets in turn; parallel mode: one session per worker, per "Parallel setup" above) visits every in-scope
screen:
1. If cycling viewports within one session (classic mode), close the previous session and start a new one
   with the next device preset's launch flags - Playwright MCP still has no mid-session device-resize call,
   so viewport is fixed for a session's lifetime exactly as before, just via a real `--device` preset instead
   of a manual `--window-size` hack.
2. Log in once per session (per "Per-role login" above, using storage-state bootstrap where available).
3. For every in-scope screen:
   - `browser_navigate({url})`. Skip a separate settle-check by default (per the Speed section) - go straight
     to the capture calls below; only add a `browser_wait_for({text:...})` first if you have a known
     ready-marker for this path or the capture below comes back visibly empty/loading.
   - `browser_take_screenshot({filename: "qa-runs/<slug>/screenshots/<role>_<screen-slug>_<viewport>.png"})`.
   - `browser_snapshot()` for the structured interactive-element list (this replaces the old
     manual controls-scrape - the snapshot already lists every interactive element with its role/name), plus,
     only if you want a raw HTML capture too,
     `browser_evaluate({function: "() => document.documentElement.outerHTML"})` written to
     `qa-runs/<slug>/html/<role>_<screen-slug>.html`.
   - **Skip console/network reads on routine screens** (per the Speed section) - only pull them when this
     specific screen gives you a reason to investigate.
4. Close the session once every in-scope screen has been captured at this viewport (classic mode), then move
   to the next device preset and repeat from step 1. In parallel mode each worker just finishes its own run.

## Safe mode (unchanged - still the rule)
- Never click a control whose accessible name matches delete/remove/deactivate/logout/sign out/unsubscribe/
  pay/submit/send/purge/terminate/revoke/confirm, unless the environment is confirmed scratch/dev **and** you
  are deliberately testing that one specific write action per Step 2c of `SKILL.md`.
- You may type into form fields and then clear them (or press Escape/navigate away) to observe validation
  states - that's fine, it's not a write.
- Direct URL navigation to a tier-gated deep route can trigger a cold-load false "locked" state on some
  platforms if session/tier data hasn't hydrated yet. When a route looks incorrectly locked, re-check it by
  navigating to the feature's list/index page first and clicking through in-app (using a real
  `browser_wait_for({text: "<ready marker>"})` rather than assuming the first read is final), and note the
  discrepancy as a finding if direct navigation genuinely fails while in-app navigation succeeds.

## Dialog handling - a real regression to manage, not a code fix
Playwright MCP auto-dismisses a native `alert()`/`confirm()`/`prompt()` dialog by default if nothing else
handles it, and `browser_handle_dialog({accept, promptText?})` lets you explicitly accept/dismiss (with text
for a `prompt()`) when you know one is coming. **Known open issue: an action that triggers a dialog can hang
the tool call with no notification that a dialog even appeared**, unlike the old locator-based tool's at least
*explicit* blocking behavior (which threw a clear "unexpected alert open" error you could react to).
Mitigation, since this is an upstream gap, not something a documentation fix can eliminate: **use the platform
spec's `## Playwright QA Config` `dialog_controls` list (see `SKILL.md` Step 1) to know, in advance, exactly
which controls on this platform trigger a genuinely *native* browser dialog** (most "Delete"/"Confirm" UIs on
modern web apps are just in-page modals with their own Confirm/Cancel buttons, NOT native dialogs at all - only
pre-arm `browser_handle_dialog` for ones confirmed native, since pre-arming it before a non-native inline
confirm just leaves it waiting for a dialog that will never fire and does nothing useful). If a click hangs
unexpectedly and you didn't know a dialog was involved, that's itself worth recording as a platform-spec
`dialog_controls` addition once you figure out what happened, so the next run doesn't hit the same surprise.

## Before finalizing any dead-click FAIL
Playwright's `browser_click` targets a `ref` from a live snapshot and auto-waits for real actionability before
acting - this removes most of the old locator-based tooling's synthetic-click-artifact risk (a click reported
as "successful" with zero real effect and zero network request). The discipline shrinks to one check, not a
multi-step dual-click dance:
1. After a click that should have visibly changed something but didn't, do one `browser_network_requests`
   check (filtered to the endpoint you'd expect, via the `filter` regexp param) to see whether a request fired
   at all. No request at all is real evidence of a dead click; a request that fired but the UI didn't
   obviously update might just mean the app doesn't render visible feedback for that action - don't conflate
   the two.
2. Only if you have specific reason to suspect a stale `ref` (you clicked something from a snapshot taken
   before an intervening DOM change), re-`browser_snapshot()` and retry once against the fresh ref before
   concluding anything.
3. Reproduce once more with a fresh navigation before filing a FAIL, exactly as the multi-step-verification
   rule in `SKILL.md` Step 2c already requires for any bug - this part is unchanged.

## New-tab / new-window detection
Testing a link/button that's supposed to open a new tab (`target="_blank"`, "Apply", "View posting", etc.):
1. Confirm via a quick DOM check (`browser_evaluate` or reading the element's attributes off the
   snapshot/HTML) whether it genuinely has a real destination (a non-empty `href` + `target="_blank"`, or an
   onclick handler calling `window.open`) - a real href is strong evidence the control is wired correctly even
   if a tab-open check below comes back empty.
2. `browser_tabs({action: "list"})` immediately before the click to record the current tab count, click, then
   list again - a genuine new tab should show up in this before/after comparison. (Confirmed live: actions are
   `list`/`new`/`close`/`select`, addressed by a numeric `index`, not a window-handle string.)
3. If both a real href/handler is confirmed AND the before/after tab-count comparison shows no new tab, that's
   real evidence of a defect. If only the tab-count check is inconclusive but the href/handler is confirmed
   real, don't mark it FAIL - mark it PASS or UNVERIFIED with a note about the tooling limitation, not a bug.

## Before marking UI text/copy "missing" - rule out a shallow DOM-query artifact first
This trap is a DOM/JavaScript fact independent of which browser MCP is driving the session, so it applies here
exactly as it did before:
- A DOM query that only inspects "leaf" nodes will silently return empty for real, visible text whenever that
  text is split across a text node and a nested inline element (e.g. a bolded phrase mid-sentence). Search the
  full subtree instead (`document.body.innerText.includes(...)` or a container's `.textContent`, via
  `browser_evaluate`), not a leaf-only filter.
- CSS `text-transform` changes what a user sees without changing the underlying DOM text (a heading rendered
  as "ONOBOARDING" via `innerText` when the raw text node actually reads "Onoboarding," uppercased only by a
  stylesheet rule - confirmed still present on this platform, 2026-08-20, via exactly this technique). Before
  concluding a typo is a rendering artifact (or vice versa), walk the raw text nodes directly via
  `browser_evaluate({function: "() => { const walker =
  document.createTreeWalker(document.body, NodeFilter.SHOW_TEXT); let node, matches = []; while (node =
  walker.nextNode()) if (/onoboarding|onboarding/i.test(node.textContent)) matches.push(node.textContent);
  return matches; }"})` - this is one of the few cases that still genuinely needs a JS-eval tool rather than a
  snapshot/ref-based one, since the visible-but-transformed text has no clean accessibility-tree node to
  target.
- Before writing up any "X message/note never appears" finding, confirm with a screenshot taken at the exact
  moment being tested, in addition to (not instead of) a DOM check - trust the screenshot if the two disagree,
  and fix the query rather than reporting the DOM check's answer anyway.

## Endpoint discovery
- Prefer a manually-provided endpoint list (`platform-specs/<slug>.md` or a file the user hands you) over pure
  link-following - hidden-nav routes (an admin or coach panel reachable only by direct URL) are easy to miss
  otherwise.
- Supplement with a link scrape (`browser_evaluate`) on every screen you do visit, to catch anything the
  manual list missed.
- **Query the backend directly, if the spec documents one, before concluding the endpoint list is complete.**
  If the platform spec gives API credentials (a Supabase project URL + anon key, a REST base URL + token), hit
  the schema/discovery endpoint - for PostgREST/Supabase, `GET {url}/rest/v1/` with the anon key returns the
  table/view list; also check for exposed RPC functions. This is read-only reconnaissance, safe even before
  scratch/dev is confirmed.
- For a Next.js/React SPA, a link scrape of `/_next/static/...` chunk source can reveal route strings not
  currently reachable via UI.
- Check `sitemap.xml`/`robots.txt` at the site root - free, rarely populated for gated apps, but worth a quick
  look.
- **Systematically enumerate every tab inside every panel you land on** - a single "Manage"/"Admin"/"Settings"
  entry point often hides many distinct sub-panels behind a tablist or sidebar that a generic crawler won't
  find, because they only exist as client-side tab state, not separate URLs. Click every tab, not just the one
  you happened to land on.
- **Write newly-discovered routes back to `platform-specs/<slug>.md` once the discovery pass is done.** Diff
  the finished inventory against the spec's existing endpoint table and append any route that isn't already
  there (same row format: path, area, notes, `*hidden nav*` flag) - never edit or remove an existing row, only
  add missing ones. Do the same for any `ready_markers`/`slow_endpoints` facts learned this pass, per the
  `## Playwright QA Config` schema in `SKILL.md` Step 1 - this is what makes "check the spec first" actually
  pay off on the *next* run.

## Confirming device-preset names and timeout comfort on first use each session
Playwright's device-preset table (`"Desktop Chrome"`, `"iPad Pro 11"`, `"iPhone 15"`, etc.) is Microsoft's own
and can gain/rename entries between Playwright releases - don't assume the three names recommended in "Parallel
setup" are permanently exact. On first use against a new server version, confirm the preset resolved to a
sane viewport/UA (a quick `browser_evaluate({function: "() => ({width: window.innerWidth, height:
window.innerHeight, ua: navigator.userAgent, touch: navigator.maxTouchPoints, dpr: window.devicePixelRatio})"})`
read after the first navigation is enough - confirmed live without any `--device` flag at all, a session
defaults to `~1900x1012, dpr:2`, not a clean number, which is exactly why an explicit device/viewport flag
matters) and correct the recommended name in this file if it drifted. Likewise, treat `--timeout-action=3500` as a
starting point, not a permanent constant - if a specific platform's real controls routinely need longer (check
`slow_endpoints` in its `## Playwright QA Config` first), override per-path rather than loosening the global
default, and note here if the global default itself turns out uncomfortable across multiple platforms.

## Output layout (unchanged)
Same as before: `qa-runs/<slug>/screenshots/`, `qa-runs/<slug>/html/` (only if you chose to capture raw HTML
alongside the snapshot), `capture_manifest.json` (hand-authored/updated as you go, same shape:
`{platform, base_url, roles: [{label, tier, screens: [{path, url, screenshots: {desktop, tablet, mobile}, html,
console_errors, controls}]}]}`).
