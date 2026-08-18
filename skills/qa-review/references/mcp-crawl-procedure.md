# Driving the crawl via the Selenium MCP server

`qa-review` drives a real, inspectable browser through a **Selenium** MCP server
(`@angiejones/mcp-selenium`) instead of a headless background script - you call the MCP browser tools
directly, turn by turn, so the session is visible and steerable rather than a black-box batch run. This is
the same server quick mode uses (see `references/selenium-quick-mode.md`); classic mode just points it at
the whole platform instead of one page.

**This server has no `uid`-addressable click-by-reference concept the way chrome-devtools did** - there is no
`click(uid)`. Elements are addressed by **locator** instead: `by` (`id | css | xpath | name | tag | class`) +
`value` (the selector string). It does, however, expose an **`accessibility://current` MCP resource**
(confirmed live) - a compact accessibility-tree JSON of the current page's interactive elements and text,
including real `id`/`name`/`role` attributes - which is the fastest, most reliable way to derive a locator.
**Read `accessibility://current` (`ReadMcpResourceTool({server: "selenium", uri: "accessibility://current"})`)
first, before guessing a common CSS pattern or falling back to a raw HTML dump** - it's smaller than full HTML
and gives you the real `id`s directly. Keep this distinction in mind throughout; it's the single biggest
procedural change from the chrome-devtools version of this file.

## Speed - don't default to slow patterns
The classic single-agent path involves hundreds of tool calls across a full run; small per-call overhead
compounds fast. Default to the fast version of every pattern below, not the cautious-but-slow one:
- **One settle-check, not a poll loop.** After `navigate`/`interact`, do **one** immediate
  `execute_script` check for the expected state. Only if that comes back empty/wrong, do **one** short retry
  (a single brief pause, not several). "Poll + pause, repeated a few times" as a *default* wastes several
  seconds per screen for no benefit in the common case - Vercel/SPA hydration on a live deploy is typically
  near-instant. Reserve a genuine multi-retry loop for a screen that's actually shown itself to be slow
  (e.g. a cold serverless function), not as standing practice on every navigation.
- **Batch independent calls into one message.** Two `send_keys` calls to different fields, or a screenshot +
  an `execute_script` read, have no ordering dependency on each other - issue them together rather than
  one-at-a-time across separate turns. (Calls with a real dependency - e.g. `interact` submit after both
  `send_keys` calls - still have to wait for their prerequisite.)
- **`diagnostics()` is on-demand, not routine.** Only call it when actively investigating a specific
  suspected dead click, console error, or failed request - not as a reflexive step after every navigation or
  click. Calling all three `type`s on every screen as a matter of course triples the round-trips for
  information that's rarely needed. Do still call it wherever the skill's dead-click/error-finding discipline
  actually asks for it - that's real investigative use, not routine overhead.
- **Don't re-derive context that's already known.** Scratch/dev confirmation, the platform spec, and the
  credential set are established once per run - don't re-check or re-read them per screen or per
  viewport-session.

## One-time setup
Check the server is connected: `claude mcp list` should show
`selenium: npx -y @angiejones/mcp-selenium@latest ... - ✔ Connected`.
If missing:
```bash
claude mcp add selenium -- npx -y @angiejones/mcp-selenium@latest
```
**There is no server-launch viewport flag** (unlike chrome-devtools' `--viewport=1920x1080`) - viewport is
set per-session, as a `start_browser` launch option (see "Per-screen capture" below). A newly-added MCP
server's tools only appear in a **fresh session** - if you just added it, tell the user to restart/reload
before the `mcp__selenium__*` tools are callable.

**Confirmed live-use gotcha: a stale cached ChromeDriver version can block every `start_browser` call.**
Selenium Manager (bundled in `selenium-webdriver`) caches downloaded drivers at `~/.cache/selenium/chromedriver/`
and can fail to resolve the version matching your currently-installed Chrome, instead reusing an old cached
one - producing `session not created: This version of ChromeDriver only supports Chrome version <old>` even
though a newer, correct driver would resolve fine. If `start_browser` fails with that error, delete the stale
cache and retry (this forces a fresh download matched to the installed Chrome):
```bash
rm -rf ~/.cache/selenium/chromedriver
```
The retry after clearing this cache should succeed immediately - no other setup change is needed.

## Parallel setup (only needed if 3-worker parallel mode was chosen - see `SKILL.md` Step 1b)
`@angiejones/mcp-selenium` is **single-active-session-per-server** by design - there is no multi-tab-within-
one-session model to worry about at all (unlike chrome-devtools, where tabs existed but silently shared
cookies). This actually simplifies isolation: there's nothing to accidentally get wrong by reusing tabs,
because there's no tab-level session sharing to reuse in the first place.

**The only way to run 3 personas at once is 3 separate server processes**, each with its own browser profile
so cookies/localStorage are genuinely isolated:
```bash
claude mcp add selenium1 -- npx -y @angiejones/mcp-selenium@latest
claude mcp add selenium2 -- npx -y @angiejones/mcp-selenium@latest
claude mcp add selenium3 -- npx -y @angiejones/mcp-selenium@latest
```
Each worker's own `start_browser` call must then pass a distinct Chrome `--user-data-dir` through
`options.arguments` (e.g. `arguments: ["--user-data-dir=/Users/<you>/.selenium-profiles/p1"]` - the real
`start_browser` schema is `{browser, options: {headless?: boolean, arguments?: string[]}}`, confirmed live;
there is no `args` key) - this is what actually isolates the profile; adding a separately-named MCP server
without a distinct `--user-data-dir` would still risk sharing the default Chrome profile across workers. Before starting a parallel run, confirm all 3 named
servers show `✔ Connected` in `claude mcp list`. A freshly-added server needs a new Claude Code session
before its tools appear - tell the user to restart if you just added these. Each persona-worker's agent
prompt must scope it to exactly one server's tools (`mcp__selenium1__*` / `mcp__selenium2__*` /
`mcp__selenium3__*`) and must never reference another server's tool prefix - see `references/parallel-mode.md`
for the full worker design.

## Load the tool schemas
The browser tools are deferred. Before the first call:
```
ToolSearch({query: "select:mcp__selenium__start_browser,mcp__selenium__navigate,mcp__selenium__interact,mcp__selenium__send_keys,mcp__selenium__get_element_text,mcp__selenium__get_element_attribute,mcp__selenium__press_key,mcp__selenium__upload_file,mcp__selenium__take_screenshot,mcp__selenium__close_session,mcp__selenium__execute_script,mcp__selenium__window,mcp__selenium__frame,mcp__selenium__alert,mcp__selenium__add_cookie,mcp__selenium__get_cookies,mcp__selenium__delete_cookie,mcp__selenium__diagnostics", max_results: 20})
```
(In parallel mode, substitute the assigned server prefix - `mcp__selenium1__*` etc. - for every name above.)

## Per-role login
1. `start_browser({browser: "chrome", options: {arguments: [...]}})` - see "Per-screen capture" below for the
   viewport-specific launch options; for the lightweight discovery pass (Step 2a), desktop-sized defaults are
   fine.
2. `navigate({url: base_url})`.
3. **Locate the email/password fields and submit control.** Read `accessibility://current` first (see the
   note at the top of this file) - it's confirmed to return real `id`/`name`/`role` attributes for form
   fields directly (e.g. `{"id": "username", "name": "username", "role": "textbox"}`), which is almost always
   enough to build a locator without guessing. If a field genuinely has no useful id/name/class in the
   resource output, fall back to, in order of reliability:
   - Common attribute patterns: `by: "css", value: "input[type='email']"` (or `input[name='email']`,
     `input[id*='email']`), `by: "css", value: "input[type='password']"`.
   - If those come up empty too, pull the raw page source
     (`execute_script({script: "return document.documentElement.outerHTML"})`) and read the actual markup -
     don't guess a third time, look at the real HTML.
   - For the submit control, the accessibility resource typically shows it as `{"role": "button", "children":
     [{"name": "Login", "role": "text"}]}` with no `id` of its own - use its containing form's id if it has
     one (`by: "css", value: "form#login button"`) or an XPath text match (`by: "xpath", value:
     "//button[contains(text(),'Log in')]"`) - don't assume a `<label>` or accessible name exists on the
     button itself.
4. `send_keys({by, value, text: email})` and `send_keys({by, value, text: password})` for the two fields -
   **issue both in one message**, they target independent elements with no ordering dependency.
5. Submit via `interact({action: "click", by, value})` on the submit control, or `press_key({key: "Enter"})`
   while the password field is focused.
6. **SPA client-side redirects are not a real navigation** - a fixed wait can fire before the redirect
   happens. This server has no dedicated `wait_for`/`textGone` tool either. Per the Speed section above: do
   **one** immediate `execute_script({script: "return document.body.innerText"})` check - if the login
   fields are already gone, that's your login-success confirmation, move on immediately; only if they're
   still present, do one short retry, not a multi-cycle poll (this single check replaces a separate
   step-3-style locator re-lookup - no need to check twice).
7. **Logging into a second role/test account in the same browser profile replaces the first account's
   session everywhere.** This is a fact about how auth providers store session state in the browser profile
   (localStorage/cookies), not a chrome-devtools quirk - it applies identically here. If you need to briefly
   check something as a different role mid-run, expect to explicitly log back in as the original role
   afterward.

## Discovery pass (SKILL.md Step 2a) - lightweight, always runs first
**This pass always runs, in full, regardless of whether the run ends up full or partial QA** - it's what
builds the screen inventory that Step 2b's scope choice is presented from. It is deliberately cheap: one
navigate + one lightweight DOM read per screen, one browser session for the whole pass (desktop-sized launch
options are fine here - no per-viewport sessions yet, that's Step 2c).

For each URL discovered (via the endpoint-discovery techniques below, or the platform spec's list):
1. `navigate({url})`, then go straight to step 2 - no separate settle check needed here, since step 2's own
   read already tells you whether the page rendered (empty/garbage text is itself the "not ready yet"
   signal). Only add a single short pause-and-retry if step 2 actually comes back empty - don't pre-emptively
   poll before even trying once.
2. Pull a lightweight read of the page - `execute_script({script: "return document.title + ' | ' +
   document.body.innerText.slice(0, 500)"})` is enough to log a short descriptive name, plus a same-origin
   link scrape (`execute_script({script: "return [...document.querySelectorAll('a[href]')].map(a =>
   a.href)"})`) - **issue both `execute_script` calls in one message**, they're independent reads of the
   same already-loaded page, to discover further routes.
3. Record the screen in the inventory: URL/path, short name, which of the 8 SOP areas it maps to (or
   **Custom** if none), role required to view it.
Do **not** launch tablet/mobile sessions, take screenshot files, or pull `diagnostics` output in this pass -
that's all deferred to Step 2c and scoped to whichever screens end up in scope after Step 2b.

The endpoint-discovery techniques under "Endpoint discovery" below (spec list, sitemap, bundle inspection,
backend schema query, tab/panel enumeration) are exactly what drives this pass - read that section before
starting.

## Per-screen capture (SKILL.md Step 2c) - full capture, only for in-scope screens
**Viewport is fixed for a browser session's lifetime in this server** (there is no chrome-devtools-style
`emulate()` call to resize mid-session) - so the loop structure inverts from the old chrome-devtools
procedure: instead of one session visiting every screen and resizing three times per screen, run **one
browser session per viewport**, and inside that session visit every in-scope screen. Concretely:

**Confirmed live-use limitation: this server's `start_browser` has no `mobileEmulation` capability at
all.** The real schema is only `{browser, options: {headless?: boolean, arguments?: string[]}}` -
`arguments` is a plain array of Chrome command-line flags, nothing richer (no device metrics, no touch
emulation, no `userAgent` structured field - a bare `--user-agent=...` string argument is the only UA control
available). Confirmed on Chrome 151/macOS, both headed and headless: **`--window-size` has a hard floor on
width around 500px that cannot be gone below** - a request narrower than that (e.g. the SOP's original
393px iPhone target) silently clamps to exactly 500px wide. Height passes through with a small, consistent
reduction from the requested value (~143px at every size tested). **The Mobile target for this skill is
therefore fixed at 500×800** (see `references/coverage-areas.md`) - request `--window-size=500,943` to get
exactly that, confirmed reproducible. This isn't a real device preset width, just the narrowest viewport this
tooling can produce - accepted as the standing Mobile definition rather than treated as an open gap.

For each of the three viewports, in turn:
1. `start_browser` with that viewport's launch options (confirmed shape - `options.arguments`, a plain
   string array; no `mobileEmulation` field exists):
   - **Desktop (1920×1080):** `start_browser({browser: "chrome", options: {arguments:
     ["--window-size=1920,1080"]}})` - confirmed to produce an exact 1920px-wide viewport.
   - **Tablet (1024×1366, iPad Pro 12.9" dimensions):** `start_browser({browser: "chrome", options:
     {arguments: ["--window-size=1024,1366", "--user-agent=<iPad Pro UA string>"]}})`. The UA string spoofs
     `navigator.userAgent` correctly (confirmed), but does **not** spoof `navigator.maxTouchPoints` or
     `devicePixelRatio` - a site that feature-detects touch support via JS (rather than just reading the UA
     or a CSS media query) will still see a non-touch desktop-like environment. Note this as a tooling
     limitation if a touch-dependent interaction behaves differently than it would on a real device.
   - **Mobile (500×800, the fixed target above):** `start_browser({browser: "chrome", options: {arguments:
     ["--window-size=500,943", "--user-agent=<mobile Safari/Chrome-on-iPhone UA string>"]}})` - confirmed to
     produce exactly 500×800.
2. Log in once for this viewport-session (per "Per-role login" above).
3. For every in-scope screen:
   - `navigate({url})`. Skip a separate settle-poll by default - go straight to the screenshot/read calls
     below; if the screenshot or HTML capture comes back visibly empty/loading, *then* do one short pause and
     retry, rather than pre-emptively polling every screen on the assumption it might be slow.
   - `take_screenshot({outputPath: "qa-runs/<slug>/screenshots/<role>_<screen-slug>_<viewport>.png"})`.
   - **One `execute_script` call returning everything else needed for this screen at once**, instead of
     separate round-trips: `execute_script({script: "return {html: document.documentElement.outerHTML,
     controls: [...document.querySelectorAll('a,button,input,select,textarea')].map(el =>
     el.outerHTML.slice(0,120))}"})`. Write `.html` out to `qa-runs/<slug>/html/<role>_<screen-slug>.html`;
     keep `.controls` for the Step 3 coverage-gap check and further link discovery.
   - **Skip `diagnostics()` on routine screens.** Only call it (per `type`, still three separate calls when
     you do) when this specific screen gives you a reason to investigate - a click that looked like it did
     nothing, a visibly broken image, a form that didn't seem to save. Calling it on every screen as a matter
     of course triples the round-trips for information you won't use on most screens.
4. `close_session()` once every in-scope screen has been captured at this viewport, then move to the next
   viewport and repeat from step 1.

This also means **login happens once per viewport-session, not once total** - a real change from the old
single-session chrome-devtools procedure, where one login covered all three viewport passes.

## Safe mode (unchanged - still the rule)
- Never click a control whose accessible name matches delete/remove/deactivate/logout/sign out/unsubscribe/
  pay/submit/send/purge/terminate/revoke/confirm, unless the environment is confirmed scratch/dev **and** you
  are deliberately testing that one specific write action per Step 2c of SKILL.md.
- You may type into form fields and then clear them (or press Escape / navigate away) to observe validation
  states - that's fine, it's not a write.
- Direct URL navigation (`navigate`) to a tier-gated deep route can trigger a cold-load false "locked" state
  on some platforms if the session/tier data hasn't hydrated yet (a real bug class - see the MLP run's BUG-02
  for a worked example). When a route looks incorrectly locked, re-check it by navigating to the feature's
  list/index page first and clicking through in-app, and note the discrepancy as a finding if direct
  navigation genuinely fails while in-app navigation succeeds.

## A native `confirm()`/`alert()` dialog blocks every subsequent call until handled - confirmed recurring
**Confirmed twice in the same run (recipes-website, 2026-08-17): clicking any delete/confirm-style control
that's implemented with a native JS `confirm()` (a common, simple pattern - `if (confirm('Delete this?'))
{...}`) opens a real browser dialog that blocks the WebDriver session entirely.** The *next* call of any kind
- `execute_script`, `navigate`, another `interact` - fails with `unexpected alert open: {Alert text: ...}`,
not a normal empty result. This isn't a dead click or a bug in the app; it's expected browser behavior that
this tool surfaces as an error instead of silently waiting. **After clicking any delete/remove/confirm-style
control, if the very next call throws `unexpected alert open`, call `alert({action: "accept"})` (or
`"dismiss"` if the check specifically calls for cancelling) before doing anything else** - don't retry the
click, don't treat the error as a failure, and don't `execute_script` your way around it (a script can't
touch a native dialog). Once accepted/dismissed, resume normally. Recognize this pattern especially before
testing any Delete control - it's common enough (this app uses it for every delete action, recipes and spice
ratios confirmed so far) that it should be an expected step, not a surprise each time.

## Before marking any click a dead click / FAIL - rule out a testing-tool click failure first
The chrome-devtools version of this procedure documented a confirmed synthetic-click-delivery gap in that
server's `click()` tool (Client A's Community-page run, 2026-08-15): it reported success with zero visible
effect and zero network request, on a control a real user click worked fine on. **This exact failure mode is
unconfirmed for Selenium's `interact` tool** - WebDriver dispatches native browser input events rather than a
synthetic CDP click, so the mechanism behind that specific bug may simply not apply here. Don't assume it's
fixed just because the mechanism differs, though - keep the same verification discipline before recording
any FAIL for "click did nothing":
1. Confirm the click target's actual position/visibility - `execute_script({script: "return
   document.querySelector(arguments[0]).getBoundingClientRect()", args: ["<css selector>"]})`. **Confirmed
   live: `args` are plain JSON literals (strings/numbers), read inside the script via `arguments[0]`,
   `arguments[1]`, etc. - there is no element-handle passing between separate tool calls; you re-locate the
   element inside the script itself (e.g. `document.querySelector(...)`), using `args` to carry the selector
   string rather than baking it as a literal into the script body.** This is still a genuine simplification
   over chrome-devtools' uid-only restriction (which couldn't take a plain string arg at all), just not in the
   "pass a resolved element across calls" sense. If the rect is negative/overflowing, the element is scrolled
   out of view or you're in the wrong viewport-session - scroll it into view and retry before concluding
   anything.
2. If the rect looks fine but `interact({action:"click"})` still produced no effect, retry via `execute_script`
   dispatching a direct `.click()` on the same element as a second opinion (locate it inside the script via
   the same selector, e.g. `document.querySelector("...").click()`). **If the JS-dispatched click succeeds
   where the tool's `interact` didn't (a request fires, or the UI updates), this was a testing-tool artifact -
   mark the control as working, not a FAIL.** Only treat it as a real defect if the JS-dispatched click also
   produces no effect.
3. Check `diagnostics({type: "network"})` immediately after the click, not just the visual DOM - a click can
   correctly fire a request that fails silently or succeeds without an obvious UI change. No request at all is
   stronger evidence of a real dead click than "the page didn't visibly change." **Confirmed live gotcha: every
   entry in the `network` output is paired with a second entry of `"type": "error"` regardless of whether the
   request actually failed** - a normal 200 response produces both a `{"type": "response", "status": 200,
   ...}` entry and a `{"type": "error", ...}` entry with no `errorText` field. **Only treat an entry as a real
   failure if it carries an `errorText` field** (e.g. `"errorText": "net::ERR_NAME_NOT_RESOLVED"`) - the bare
   `"type": "error"` label alone is not evidence of failure with this tool, and treating it as such will
   produce false "network request failed" findings on every successful request.
4. This check is especially important for any finding attributed to a shared "systemic dead-click" root
   cause (many independent controls all appearing broken the same way) - that pattern is exactly as
   consistent with a testing-tool click-delivery problem as it is with a single app-level bug, even if this
   specific tool's click mechanism is generally more reliable than a synthetic CDP click.

## Before marking UI text/copy "missing" - rule out a shallow DOM-query artifact first
**An `execute_script` text search that only inspects "leaf" DOM nodes (elements with zero child elements)
will silently return empty for real, visible text whenever that text is split across a text node and a
nested inline element** - e.g. a note like "Inputs changed since this was generated. Hit **Generate script**
to refresh…" where the bolded phrase is its own `<strong>`/`<span>`. No single leaf node then contains the
full search substring, so a query like
`Array.from(document.querySelectorAll('*')).filter(el => el.children.length===0 && /some phrase/i.test(el.textContent))`
returns `[]` regardless of whether the text is actually on screen. This produced a confirmed false-positive
bug report on the Client B's run (2026-07-23): a "note does not appear after editing this field" finding was
retracted after a live re-check showed the note rendering correctly - the DOM query itself was structurally
blind to it, not the app. **A related trap: CSS `text-transform` changes what a user sees without changing
the underlying DOM text** - confirmed on the Client A's Community-page run (2026-08-15), where a heading
rendered as "ONOBOARDING" in `innerText` but the raw text node (found via
`document.createTreeWalker(document.body, NodeFilter.SHOW_TEXT)`) was actually "Onoboarding," uppercased only
by a stylesheet rule. This is a DOM/JavaScript fact independent of which browser MCP server is driving the
session - it applies exactly the same way to Selenium's `execute_script` as it did to chrome-devtools'
`evaluate_script`. Before concluding a typo is a rendering artifact rather than real content (or vice versa),
check the raw text node directly rather than trusting `innerText`/`textContent` alone:
1. Never conclude text/copy is absent from a single leaf-node-only `querySelectorAll` scan. Search the full
   subtree instead - e.g. check `document.body.innerText.includes(...)` or a container element's
   `.textContent` (which concatenates all descendant text regardless of nesting), not a leaf-only filter.
2. Before writing up any "X message/note never appears" finding, confirm with a screenshot taken at the exact
   moment being tested, in addition to (not instead of) a DOM check - a screenshot is ground truth for what a
   real user would see; a DOM query is only as good as its own selector logic.
3. If a DOM check and a screenshot disagree, trust the screenshot and fix the query - don't split the
   difference or report the DOM check's answer anyway.
4. This also applies to timing: a query run synchronously immediately after `send_keys`/`interact` can race a
   debounced re-render and catch the pre-update DOM. If a "missing" result seems surprising given the guide's
   expectation, re-check after a short pause before concluding it's real.

## New-tab detection
Testing a link/button that's supposed to open a new tab (`target="_blank"`, "Apply", "View posting", etc.)
needs a different check than same-page state changes - checking the window list *after* clicking alone isn't
reliable, since a newly-opened tab can be missed if it wasn't being watched for at the moment it opened.
Before concluding such a control is broken:
1. First confirm via `execute_script` whether the element genuinely has a real destination (e.g. an `<a>`
   with a non-empty `href` and `target="_blank"`, or an onclick handler that calls `window.open`). If it
   does, that's strong evidence the control is wired correctly even if a new-tab check comes back empty - a
   real href is not decorative.
2. Call `window({action: "list"})` (or the server's equivalent window-handle listing call) immediately before
   the click to record the current handle count, then click, then call it again - a genuine new-tab open
   should show up in this before/after comparison, which is more reliable than a single post-click check.
3. If both a real href/`window.open` call is confirmed AND a before/after handle-count comparison shows no
   new window, that's real evidence of a defect. If only the handle-count check is inconclusive but the
   href/handler is confirmed real, don't mark it FAIL - mark it PASS (a verified destination is verified
   functionality) or UNVERIFIED with a note about the tooling limitation, not a bug.

## Endpoint discovery
- Prefer a manually-provided endpoint list (`platform-specs/<slug>.md` or a file the user hands you) over
  pure link-following - hidden-nav routes (e.g. an admin or CSM panel reachable only by direct URL, not
  linked from any menu) are easy to miss otherwise.
- Supplement with `execute_script` link discovery on every screen you do visit, to catch anything the manual
  list missed.
- **Query the backend directly, if the spec documents one, before concluding the endpoint list is
  complete.** Nav-following and even bundle inspection only surface routes someone thought to link - the
  backend's own schema enumerates every feature surface that *exists*, whether or not the front end currently
  links to it. Concretely:
  - If the platform spec gives API credentials (e.g. a Supabase project URL + anon key, a REST base URL +
    token), hit the schema/discovery endpoint - for PostgREST/Supabase, `GET {url}/rest/v1/` with the anon
    key returns the OpenAPI-ish table/view list; also check for exposed RPC functions (`/rest/v1/rpc/...`).
    Table and view names (users, tiers, channels, tickets, courses/modules, etc.) tell you which feature
    surfaces exist and often their route/key naming - cross-reference those keys against the front end to
    find deep-links a click-through wouldn't surface.
  - For a Next.js/React SPA, an `execute_script` fetch of `/_next/static/...` chunk source can reveal route
    strings not currently reachable via UI.
  - Check `sitemap.xml` / `robots.txt` at the site root - free, rarely populated for gated apps, but worth a
    quick look.
  - This is read-only reconnaissance (GETs against a schema/discovery endpoint), not a write - safe even
    before scratch/dev is confirmed. Use it to build the endpoint list, then capture/assess each discovered
    screen through the normal per-screen crawl loop below.
- **Write newly-discovered routes back to `platform-specs/<slug>.md` once the discovery pass (`SKILL.md` Step
  2a) is done.** Diff the finished inventory against the spec's existing endpoint table and append any route
  that isn't already there (same row format: path, area, notes, `*hidden nav*` flag) - never edit or remove an
  existing row, only add missing ones. This is what makes the "check the spec first" step at the top of this
  list actually pay off on the *next* run: a hidden-nav admin/CSM route found today via tab-enumeration or a
  schema query doesn't need re-discovering next time, it's just there in the spec.

## Output layout (unchanged)
Same as before: `qa-runs/<slug>/screenshots/`, `qa-runs/<slug>/html/` (save the page source via
`execute_script(() => document.documentElement.outerHTML)` if you want an HTML capture alongside the
screenshot), `capture_manifest.json` (hand-authored/updated as you go, since there's no longer a script
producing it automatically - same shape: `{platform, base_url, roles: [{label, tier, screens: [{path, url,
screenshots: {desktop, tablet, mobile}, html, console_errors, controls}]}]}`).
