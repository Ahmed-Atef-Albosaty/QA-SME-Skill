# Quick mode - Selenium MCP, single page + subsidiaries

Triggered by `/qa-review quick <page_url> <email> <password>`. This is a **self-contained** procedure - for
a quick-mode run, follow this file in full instead of `SKILL.md`'s Steps 1–6. Nothing here reads from
`references/coverage-areas.md` or `references/report-template.md` (those are SOP-scorecard-specific and
don't apply to this mode).

## This is NOT `/qa-run`

`/qa-run` (a separate command, `.claude/commands/qa-run.md` backed by `tools/qa/run.py` and friends) is a
**script-driven** architecture: raw Playwright-Python library, no turn-by-turn MCP tool calls, results written
to a Google Sheet, a hard 100K-token budget, and it explicitly forbids MCP-driven step-by-step testing.

Quick mode is the opposite of that on every one of those axes: it stays **fully conversational, turn-by-turn
MCP tool calls**, exactly like the rest of `qa-review` - it only swaps which browser MCP server drives the
session (`selenium` instead of `fast-playwright`/`chrome-devtools`) and uses a smaller, auto-derived scope
instead of a full-platform inventory. If you find yourself reaching for a Python script, a Google Sheet, or a
token-budget cutoff while running this mode, stop - that's `/qa-run`'s pattern, not this one.

## One-time setup

Check the server is connected:
```
claude mcp list   # look for "selenium: npx -y @angiejones/mcp-selenium@latest ... - ✔ Connected"
```
If missing, `claude mcp add selenium -- npx -y @angiejones/mcp-selenium@latest`. **A freshly-added server's
tools only appear in a new session** - same rule as fast-playwright/chrome-devtools/google-workspace/any
other MCP server; tell the user to restart if it was just added.

**This server has no `uid`-addressable click-by-reference concept.** Elements are addressed by **locator**
(`by: id|css|xpath|name|tag|class`, `value: <selector string>`) instead of a snapshot-derived `uid`. It does
expose an **`accessibility://current` MCP resource** (confirmed live) - a compact accessibility-tree JSON
with real `id`/`name`/`role` attributes for interactive elements - which is the fastest way to derive a
locator. Every instruction below that talks about "finding a control" means: read `accessibility://current`
first, derive a locator from what it shows, and only fall back to guessing common CSS patterns or a raw page
source dump if that comes up empty.

Load the tool schemas before first use (deferred like any MCP tool):
```
ToolSearch({query: "select:mcp__selenium__start_browser,mcp__selenium__navigate,mcp__selenium__interact,mcp__selenium__send_keys,mcp__selenium__get_element_text,mcp__selenium__get_element_attribute,mcp__selenium__press_key,mcp__selenium__upload_file,mcp__selenium__take_screenshot,mcp__selenium__close_session,mcp__selenium__execute_script,mcp__selenium__window,mcp__selenium__frame,mcp__selenium__alert,mcp__selenium__diagnostics"})
```

## Tool mapping - chrome-devtools → Selenium

If you already know the chrome-devtools-era crawl procedure, use this table to translate. These are **not**
drop-in renames - the interaction model itself changed (locator-based, not `uid`-based), and viewport is now
a per-session launch option rather than a mid-session resize.

| chrome-devtools | Selenium (`@angiejones/mcp-selenium`) | Key difference |
|---|---|---|
| `navigate_page` | `navigate` | same idea, no `type: url\|back\|forward\|reload` variants - use `execute_script` (`history.back()`) for back/forward if needed |
| `take_snapshot` | *(none)* | no accessibility-tree/`uid` concept at all - locate elements by `by`/`value` locator instead, derived from page source or a targeted `execute_script` query |
| `take_screenshot` | `take_screenshot` | `outputPath`, not `filePath`; no documented `fullPage` option |
| `click` | `interact({action: "click", by, value})` | takes a locator, not a `uid` - also covers doubleclick/rightclick/hover via the same tool's `action` param |
| `fill` / `type_text` | `send_keys({by, value, text})` | always locator-based - there's no keyboard-level "type into whatever's focused" variant, always target a specific element |
| `fill_form` | *(none)* | no multi-field-in-one-call helper - issue one `send_keys` per field |
| `resize_page` | `start_browser({options: {arguments: ["--window-size=W,H"]}})` | **confirmed shape is `options.arguments` (plain string array of Chrome CLI flags), not `args`, and viewport is a launch option, not a runtime call** - to change it, close the session and start a new one. Confirmed live: width passes through exactly at 1920, but has a **hard floor around 500px** - a request below that clamps to exactly 500, in both headed and headless mode. Height passes through with a consistent ~143px reduction from the requested value (`--window-size=500,943` → confirmed exact 500×800). This is why Mobile is fixed at **500×800** for this skill (see `references/coverage-areas.md`) rather than a true phone width - see Step 5 below. |
| `emulate` | *(none - no structured capability at all)* | **confirmed: `start_browser`'s `options` only has `headless` and `arguments` - there is no `mobileEmulation`, device-metrics, touch, network-throttling, geolocation, or colorScheme field of any kind.** The only device-signal control available is a bare `--user-agent=<string>` Chrome argument, which spoofs `navigator.userAgent` (confirmed) but not `navigator.maxTouchPoints` or `devicePixelRatio` - a real capability gap versus chrome-devtools' `emulate()`, not just a different call shape. |
| `list_console_messages` / `list_network_requests` | `diagnostics({type: "console"\|"errors"\|"network"})` | **confirmed: three separate calls, one per `type`, not one combined payload.** Confirmed gotcha in the `network` output: every entry is paired with a second `{"type":"error"}` entry regardless of actual success - only an entry that also carries an `errorText` field is a real failure; the bare `"type":"error"` label means nothing on its own. |
| `evaluate_script` | `execute_script({script, args})` | **confirmed: `args` are plain JSON literals (strings/numbers) read inside the script via `arguments[0]`, `arguments[1]`, etc.** - e.g. pass a CSS selector string and re-query with `document.querySelector(arguments[0])` inside the script. There's no element-handle passing between separate tool calls (nothing to pass "an element reference" as), but this is still a real simplification over chrome-devtools' uid-only restriction, which couldn't take even a plain string arg. |
| `list_pages` / `new_page` / `select_page` / `close_page` | `window({action: "list"\|"switch"\|"close"})` | window-handle based, not numeric `pageId` based |
| `wait_for` | *(none)* | no dedicated wait tool at all - do **one** immediate `execute_script` state check first; only add a single brief pause and retry once if that first check comes back empty. See `mcp-crawl-procedure.md`'s "Speed" section - don't default to a multi-cycle poll |
| `handle_dialog` | `alert({action: "accept"\|"dismiss"\|"read"\|"send text"})` | same purpose, different tool name |
| `upload_file` | `upload_file({filePath})` | same idea, needs a locator for the file input via a separate `interact`/derived locator step first in some flows - confirm the exact call shape on first live use |
| - (n/a) | `add_cookie` / `get_cookies` / `delete_cookie` | bonus capability chrome-devtools didn't expose directly |

## Operational gotchas confirmed in live use

**Setup: a stale cached ChromeDriver can block every `start_browser` call.** Selenium Manager caches drivers
at `~/.cache/selenium/chromedriver/` and can fail to resolve the version matching your installed Chrome,
reusing an old cached one instead - producing `session not created: This version of ChromeDriver only
supports Chrome version <old>`. Fix: `rm -rf ~/.cache/selenium/chromedriver` and retry; it re-downloads the
correct version on the next `start_browser` call.

**Device emulation: confirmed real, not just unconfirmed-pending-test.** `start_browser`'s `options` only
exposes `headless` and `arguments` - there is no `mobileEmulation` field. A `--window-size` request below
~500px wide silently clamps to ~500px in both headed and headless mode - which is why Mobile is fixed at
**500×800** for this skill (request `--window-size=500,943`) rather than a true phone width. See the
tool-mapping table above and Step 5 below.

**A native `confirm()`/`alert()` dialog blocks every subsequent call until handled - confirmed recurring
(recipes-website, 2026-08-17, twice in one run).** Clicking a delete/confirm-style control implemented with a
plain JS `confirm()` opens a real dialog that blocks the session - the next call of any kind fails with
`unexpected alert open: {...}`, not a normal result. This is expected browser behavior, not a dead click or a
bug: call `alert({action: "accept"})` (or `"dismiss"`) the moment you see this error, then resume - don't
retry the click or fight it with `execute_script`. Expect this on essentially every Delete control tested.

Two other false-negative patterns showed up repeatedly on real chrome-devtools-era runs. The underlying
discipline still applies here - don't assume Selenium is immune to the same *class* of problem just because
the specific tool changed:

- **A poll-based wait (`execute_script` + pause, repeated) can appear to time out even when the expected
  content is already present**, the same class of issue chrome-devtools' `wait_for` had - if a poll comes back
  empty on the first try, don't assume the content failed to load; retry the direct `execute_script` check
  once or twice with a short pause between before concluding anything is actually wrong.
- **The first visit to a client-side sub-section (e.g. a channel/tab) in a session may open a first-visit
  "About this X" onboarding modal**, which blocks interaction with the underlying content until dismissed. An
  `interact` click on the nav control itself can silently do nothing if a modal is intercepting it - re-check
  the page source/DOM for a visible modal and its dismiss button (often "Got it" or similar) before
  concluding the click didn't work; dismiss it, then proceed - don't write this up as a dead-click finding.
- **A click can report success while doing nothing, on controls a real user click works fine on.** This exact
  failure mode was confirmed for chrome-devtools' synthetic-CDP `click()` tool (Client A's Community-page
  run, 2026-08-15) - **not yet confirmed for Selenium's native `interact` click**, which dispatches real
  WebDriver input events rather than a synthetic CDP click, so the specific mechanism behind that bug may not
  reproduce here. Keep the verification discipline anyway (see the dead-click section in
  `mcp-crawl-procedure.md`, which applies unchanged to this mode): before writing up any dead-click finding,
  always try the JS-dispatched-click fallback via `execute_script` - if it succeeds where `interact` didn't,
  the control is fine and this was a testing-tool artifact, not a bug.

## 1. Gather inputs

- **`page_url`, `email`, `password`** - from the invocation; ask for any that are missing rather than
  guessing.
- **Crawl depth `N`** - ask the user how many hops to follow from `page_url` (suggest **2** as a sensible
  default if they don't have a preference). Regardless of `N`, enforce a **hard safety cap of 60 pages
  visited** - state this cap to the user up front. A well-linked app can otherwise turn an innocent "check
  this page" into a near-full-site crawl.
- **Role label** - optional; default to `"user"` if not given.
- **Slug** - derive from `page_url`'s hostname (e.g. `app.example.com` → `app-example-com`, strip scheme,
  path, and non-alphanumeric characters other than `-`). Tell the user the derived slug and let them override
  before proceeding - it's used for `qa-runs/<slug>/`.
- **Platform spec** - check for `platform-specs/<slug>.md`. If it exists, read it for context (known
  tiers/routes) and check for a `## QA Environment` section with `Scratch/dev confirmed: YES` (see next
  bullet). **Never require creating a spec file for quick mode** - if none exists, proceed without one, and
  only create/append one later if new subsidiary routes are actually found (see Discovery below).
- **Scratch/dev confirmation** - identical rule to classic Step 1: if `platform-specs/<slug>.md` already has
  `Scratch/dev confirmed: YES`, treat it as confirmed and don't re-ask. Otherwise ask the user directly; once
  confirmed, write it back to `platform-specs/<slug>.md` (creating the file only at this point, if it didn't
  already exist) so future runs against this slug don't need to ask again. If not confirmed (or the user
  declines), run **read-only only** - mark every write-path item UNVERIFIED rather than guessing, exactly as
  the classic path does.
- **No parallel-mode question.** Quick mode is always single-agent, single browser - there's one credential
  set, so the classic Step 1b (classic vs. 3-worker parallel) doesn't apply and isn't asked.

## 2. Login

1. `start_browser({browser: "chrome", options: {arguments: ["--window-size=1920,1080"]}})` for the initial
   discovery/desktop pass.
2. `navigate({url: page_url})` - if a separate login URL exists, navigate there first (or let the redirect to
   a login form happen naturally).
3. Read `accessibility://current` first and derive locators for the email/password fields and submit control
   from its `id`/`name`/`role` output (see "Per-role login" in `mcp-crawl-procedure.md` for the exact
   recipe) - don't assume a `<label>` exists, and don't guess a CSS pattern before checking the resource.
4. `send_keys({by, value, text: email})` and `send_keys({by, value, text: password})` - **issue both in one
   message**, one call per field (this server has no multi-field `fill_form`-style helper), but they're
   independent so there's no reason to serialize them.
5. Submit via `interact({action: "click", by, value})` on the submit control, or `press_key({key: "Enter"})`
   while the password field is focused.
6. Confirm login succeeded with **one** immediate `execute_script({script: "return document.body.innerText"})`
   check - if the login fields are already gone, that's confirmation, move on. Only if they're still present,
   add a single brief pause and retry once - don't default to a multi-cycle poll.
7. Same session-sharing gotcha as classic mode applies: logging into a second account in the same browser
   profile replaces the first session everywhere. Quick mode only ever uses one account, so this shouldn't
   come up - but if you ever need to spot-check something as a different login mid-run, expect to log back in
   afterward.

## 3. Discover - "current page + its subsidiaries"

This replaces classic Step 2a's whole-platform inventory with a bounded, same-origin BFS from `page_url`.

**Critical caveat, confirmed in live use (chrome-devtools era, Client A's Community page,
2026-08-15): a pure `<a href>` same-origin scrape can badly mis-scope this step if the given page's own
sub-structure is client-side (buttons/tabs), not real links.** The initial discovery pass on that run scraped
`<a href>` elements and found nothing but the site-wide global sidebar nav - present identically on every
page and not specific to the given page at all - while completely missing the page's actual, meaningful
subsidiaries: a 10-channel switcher and a 5-item Resources/Knowledge-Base panel, both implemented entirely as
`<button>` elements with no `href`. This is a DOM-structure fact independent of which MCP server drives the
session, so it applies exactly the same way here. **Before or alongside the href-based BFS, always inspect
the given page's own page source for a sidebar/tablist/panel-switcher of buttons that represent that page's
own sub-sections** (the same "consolidated admin panel" pattern classic mode's Step 3 warns about) - if
present, treat *those* as the primary subsidiaries for this page, not just whatever a handful of `<a href>`
elements happen to return. Confirm with the user (per Step 4) which interpretation matches what they meant by
"subsidiaries" if it's ambiguous - don't silently default to the shallow link-scrape and call the run
complete.

1. Start a queue with `page_url` at depth 0. Visited-set starts empty.
2. While the queue is non-empty, depth ≤ `N`, and visited count < 60:
   - Pop the next URL. If already visited, skip. `navigate({url})`.
   - Extract same-origin links via `execute_script`, with the origin baked into the script body:
     ```
     execute_script({script: "return [...document.querySelectorAll('a[href]')].map(a => a.href).filter(h => h.startsWith('https://app.example.com'))"})
     ```
   - Record this URL in the inventory (URL, short descriptive name from the page title/heading, via
     `execute_script({script: "return document.title"})` or an `h1` read).
   - Enqueue any newly-seen same-origin links at depth + 1 (only if depth + 1 ≤ `N` and total queued+visited
     is still under the 60-page cap).
3. This builds a flat page inventory - no SOP-area mapping, no ADMIN/TIER/CUSTOM tagging (this mode doesn't
   use the SOP model at all).
4. **Write newly-found routes back to `platform-specs/<slug>.md` only if that file already exists** - append
   any route not already listed, same convention as classic mode (never edit/remove an existing entry). If no
   spec file exists for this slug, skip this silently - don't create one just to record routes from a quick
   scan.

## 4. Confirm scope

Echo the full discovered page list back to the user before the expensive capture pass - depending on `N`,
this could be a handful of pages or several dozen. Let them exclude specific pages before proceeding. There's
no full/partial menu here (scope is auto-derived from the crawl), just a chance to prune before the costly
part starts.

## 5. Capture - per in-scope page

**Viewport is fixed for a browser session's lifetime in this server** - so capture inverts to a
viewport-outer-loop, same restructuring as classic mode (see `mcp-crawl-procedure.md`'s "Per-screen capture"):

**Confirmed live: this server's `start_browser` has no `mobileEmulation` capability at all** - `options` is
only `{headless?, arguments?: string[]}`. `--window-size` has a hard floor on width around 500px (confirmed
on Chrome 151/macOS, headed and headless alike) - a true phone-width viewport below that isn't achievable, so
**Mobile is fixed at 500×800 for this skill** (see `references/coverage-areas.md`) - request
`--window-size=500,943` to get exactly that, confirmed reproducible.

For each of the three viewports, in turn:
1. `close_session()` the previous viewport's session (skip on the very first viewport), then `start_browser`
   with this viewport's launch options (`options.arguments`, a plain string array - confirmed shape):
   - **Desktop (1920×1080):** `start_browser({browser: "chrome", options: {arguments:
     ["--window-size=1920,1080"]}})` - confirmed to produce an exact 1920px-wide viewport.
   - **Tablet (1024×1366, iPad Pro 12.9" dimensions):** `start_browser({browser: "chrome", options:
     {arguments: ["--window-size=1024,1366", "--user-agent=<iPad Pro UA string>"]}})` - the UA string spoofs
     `navigator.userAgent` (confirmed) but not touch support or `devicePixelRatio`.
   - **Mobile (500×800, the fixed target above):** `start_browser({browser: "chrome", options: {arguments:
     ["--window-size=500,943", "--user-agent=<mobile Safari/Chrome-on-iPhone UA string>"]}})` - confirmed to
     produce exactly 500×800.
2. Log in once for this viewport-session (per Step 2 above).
3. For every in-scope page:
   - `navigate({url})`, then go straight to the screenshot/read calls below - no separate settle poll by
     default. Only add one short pause-and-retry if those calls come back visibly empty/loading.
   - `take_screenshot({outputPath: "qa-runs/<slug>/screenshots/<role>_<screen-slug>_<viewport>.png"})`.
   - **One `execute_script` call for both HTML and controls at once**: `execute_script({script: "return
     {html: document.documentElement.outerHTML, controls:
     [...document.querySelectorAll('a,button,input,select,textarea')].map(el =>
     el.outerHTML.slice(0,120))}"})` - write `.html` to `qa-runs/<slug>/html/<role>_<screen-slug>.html`, keep
     `.controls` for the assessment pass.
   - **Skip `diagnostics()` unless this specific page gives a reason to investigate** (a dead-looking click, a
     broken image, a form that didn't seem to save) - it's `{type: "console"|"errors"|"network"}`, three
     separate calls when you do need it, not routine per-page overhead. For `network`, only trust an entry as
     a real failure if it carries an `errorText` field - every request, successful or not, also produces a
     bare `{"type":"error"}` entry with no `errorText`, which is not itself evidence of failure.
4. Move to the next viewport and repeat from step 1.

This means login happens once per viewport-session (three times total across the run), not once total - a
real change from the old single-session chrome-devtools procedure.

**Safe mode still applies exactly as classic mode:** open forms and probe fields, but never click
submit/delete/logout/deactivate/pay/send/purge/confirm controls unless scratch/dev is confirmed and you're
deliberately testing one specific write action.

## 5b. Interact - every functional control on the page MUST be exercised, not just observed

**This is a hard requirement, not best-effort, and it applies to every quick-mode run regardless of scope - partial or total.** A run that only navigates to each in-scope page, takes screenshots, and clicks nav/tab
controls to switch views is **not a complete quick-mode run**, even if every page was visited and
screenshotted. Looking at a screen is not the same as testing it.

**Incident that motivated this rule:** on the first live quick-mode run (Client A's Community page,
2026-08-15, under the chrome-devtools-era version of this mode), the entire pass consisted of clicking
channel tabs and capturing screenshots - the post composer (Post/Photo/Video/GIF/Emoji), reaction buttons,
comment threads, the search box, and every Edit/Delete control on Resource library/Video library/Knowledge
Base were all visually observed and screenshotted but never actually clicked or exercised. The write-up read
as a complete functional QA pass when it was really a UI tour. The gap was only caught because the user asked
directly whether posting, reacting, and commenting had actually been tested. Treat that question as one this
mode must always be able to answer "yes" to, unprompted - regardless of which browser MCP server is driving
the session.

Concretely, before writing up any page's assessment as finished:
1. From that page's page source / a targeted `execute_script` sweep, enumerate every interactive control -
   every button, input, toggle, menu item, and link that isn't pure navigation. This includes (not
   exhaustively): post/comment composers, reaction pickers, comment-thread open/reply, search/filter inputs,
   sort/view toggles, per-item action menus (Edit/Delete/Move/Hide/Pin/etc.), file/image/video upload
   controls, and any "About this X" or settings dialogs reachable from the page.
2. **Exercise every one of them**, not just the ones that look interesting. Once scratch/dev is confirmed
   (Step 1), this includes write actions - reuse classic mode's Step 4b disposable-test-entity discipline:
   create a throwaway post/comment/reaction/upload through the page's own UI, confirm the result (ideally
   with a reload, not just an in-session visual check, to rule out optimistic-UI-only success), then
   delete/revert it afterward. Never exercise a control that would affect another real user's data or send a
   real notification/email without the same safeguards classic mode's Step 4b already documents (disposable
   `+alias` test accounts, etc.).
3. Only a control that is genuinely destructive/irreversible against shared data, or that the platform gives
   no safe way to revert, may be left as "observed, not clicked" - and even then, say so explicitly in that
   finding's notes rather than silently skipping it.
4. If time runs out before every control on every in-scope page has been exercised, **say so explicitly to
   the user** (which pages/controls were only visually observed) rather than presenting the run as complete
   functional coverage. A partial run is fine; an unstated one is not.

## 6. Assess

Per page, evaluate against the general assessment lenses - **not** the SOP criteria-sheet model:

- **`references/istqb-principles.md`** - safe-testing rules, golden-rule mindset. Applies unchanged.
- **`references/review-dimensions.md`** (§3.1–3.10, six-hats dimensions: Functional, Workflow, Validation,
  UI, UX, Accessibility, Design System, Error Handling, Security Observations, Product) - these are general
  review dimensions, not SOP-specific, so they apply directly to any page.
- **Do not read `references/coverage-areas.md`** - no 8-area SOP mapping, no ADMIN/TIER/CUSTOM tags apply
  here; this mode has no fixed-area model to map an arbitrary page into.
- **The mandatory UI nitpick sweep still applies** (fonts, padding, margins & whitespace, spacing/distances,
  alignment & symmetry, visual coordination) - run it as a dedicated final pass over every captured page,
  same discipline as classic Step 4, and file every nitpick found as its own UI finding, however minor.
- **Dead-click / false-positive verification**, adapted to Selenium signatures:
  1. Confirm the target element's position/visibility via `execute_script({script: "return
     document.querySelector(arguments[0]).getBoundingClientRect()", args: ["<css selector>"]})` - `args` are
     plain JSON literals read via `arguments[0]` inside the script (confirmed live), used here to carry the
     selector string rather than baking it into the script body; there's no element-handle passing between
     separate tool calls.
  2. If the rect looks fine but the real `interact({action:"click"})` had no effect, try a JS-dispatched click
     as a second opinion: `execute_script({script: "document.querySelector('...').click()"})`.
  3. Check `diagnostics({type: "network"})` too - only an entry with an `errorText` field is a real failure;
     every request also produces a bare `{"type":"error"}` entry that means nothing on its own (confirmed
     live).
  4. If both the real click and the JS-dispatched click fail, the finding is unconfirmed, not proven - note
     it as such and, if possible, ask for a manual spot-check rather than writing it up as a confirmed
     Critical bug.
  4. Same discipline for a "text/copy missing" claim: search `document.body.innerText` or a container's
     `.textContent`, never a leaf-node-only query - see `references/mcp-crawl-procedure.md`'s "Before marking
     UI text/copy missing" section for the full reasoning (it applies regardless of which browser MCP is
     driving the session).
- Classify every issue found with `references/finding-types.md` (finding type - Bug/Note/Question/Idea - plus
  ISTQB category, severity, priority) - same classification system as classic mode, just with no
  criteria-sheet row to attach it to.
- **No 95%-non-UNVERIFIED hard minimum** - there's no criteria sheet to compute that percentage against. Still
  apply the general rules from `SKILL.md`'s Rules section: don't assume a feature works because it exists,
  test end-to-end not just the first step, redact secrets from screenshots/logs.

## 7. Write outputs

Write to `qa-runs/<slug>/` (same root as classic runs):

- **`screenshots/`**, **`html/`** - identical layout/naming convention to classic mode
  (`<role>_<screen-slug>_<viewport>.png`).
- **`bug-screenshots/`** - same convention as classic mode, and just as mandatory here: the moment a finding
  is classified as a **Bug** in Step 6 (not a Note/Question/Idea), copy (don't move) its evidence screenshot
  from `screenshots/` into `qa-runs/<slug>/bug-screenshots/` right then, not as a batch step at the end - it's much easier to catch a wrong-viewport or wrong-control screenshot immediately than to reconstruct it
  later. Reference it in `findings.json` as a **bare filename** (e.g. `"foucbug_desktop.png"`), never a
  `screenshots/...` path. This curated folder - physically separate from the full unfiltered crawl archive in
  `screenshots/` - is what gets uploaded to Google Drive and embedded in the Google Doc export below; a Bug
  finding with no corresponding file in `bug-screenshots/` will show up as a missing image in the doc.
- **Every `bug-screenshots/` image must have a red circle/box drawn around the exact spot the bug is
  visible** - a reader shouldn't have to hunt across a full-page screenshot to find what's wrong. This is a
  mandatory finishing step for every visual (non-network-level) finding, done once the finding's screenshot
  is otherwise final. The reliable recipe:
  1. With the buggy element still on screen in the live browser, get its precise position via
     `execute_script({script: "return document.querySelector(arguments[0]).getBoundingClientRect()", args:
     ["<css selector>"]})` for a normal element, or - if the visible (CSS `text-transform`'d) text doesn't
     literally exist in the DOM as a match target (e.g. an uppercased heading) - walk text nodes directly
     (`document.createTreeWalker(document.body, NodeFilter.SHOW_TEXT)`) and use
     `document.createRange().selectNodeContents(node).getBoundingClientRect()` on the raw text node itself,
     which is tighter than the parent element's full-width container rect.
  2. Take the screenshot (`take_screenshot`), then check its actual pixel dimensions (e.g. via a quick Python
     `PIL.Image.open(...).size` call) against the CSS-pixel viewport size from step 1 - on a Retina/HiDPI
     session these commonly differ from the raw CSS rect (confirmed live: macOS reports `devicePixelRatio: 2`
     regardless of viewport size or `--user-agent`, since there's no structured device-metrics override to
     change it), so the rect from step 1 must be multiplied by the confirmed scale factor before it lines up
     with actual screenshot pixels. Don't assume 1:1; verify once per session and reuse the confirmed factor.
  3. Draw the annotation with Pillow (`ImageDraw.Draw(im).ellipse(...)` for a small control/icon,
     `.rounded_rectangle(...)` for a wider region like an entire sidebar) with a few pixels of padding around
     the scaled rect, stroked several times at 1px offsets for a visibly thick red outline (`(255,0,0)`) - save over the same `bug-screenshots/` filename.
  4. **Always read the annotated PNG back (the `Read` tool renders images) before moving on** - confirm the
     circle actually lands on the bug and not a same-named sidebar/nav element elsewhere on the page (a plain
     text-match on a common word like "Questions" can hit the nav item instead of the page header; narrow the
     selector with `!el.closest('aside')` or similar until the rect matches the visible target).
  5. For a fleeting/timing-based bug (e.g. a flash-of-unstyled-content that self-corrects in 1-3s), the
     *container's* rect is stable even though its *content* isn't - grab the container's rect from the
     resolved state (any time), then apply that same box to the screenshot that happened to catch the bug
     mid-glitch; the box will still land in the right place since the layout region doesn't move.
- **`findings.json`** - flat array, same shape as classic minus the `area` field (no SOP area to record):
  `{id, screen, role, device, finding_type, category, severity, priority, title, steps, expected, actual,
  impact, recommendation}`.
- **`quick-scan-report.md`** - this mode's lighter report (does not use `references/report-template.md`'s
  SOP-scorecard structure):
  1. **Executive Summary** - `page_url`, crawl depth `N`, number of pages actually visited, role tested,
     scratch/dev status, and **explicit confirmation that every control on every in-scope page was exercised
     per Step 5b** (or, if not, exactly which pages/controls were only visually observed and not clicked - per Step 5b point 4, this must be stated, not left implicit).
  2. **Pages Tested** - flat list of every page visited (URL + short name).
  3. **Findings** - written as **human narration, not a filled-in form.** For each finding, tell the story of
     how you actually found it - what you were doing, what you noticed, what you checked to rule out an
     innocent explanation, and why it matters - in flowing prose under a heading, the way you'd explain it to
     a colleague out loud. This is a hard requirement for both `quick-scan-report.md` and the Google Doc
     export below, not just a style preference: a bulleted `steps:`/`expected:`/`actual:` field dump (that
     belongs in `findings.json`, which stays fully structured for machine use) reads like a bug tracker, not a
     report a human enjoys reading. Still structure the underlying data model exactly as before (one entry per
     finding in `findings.json`, full fields) - this narration requirement only changes how `findings.json`'s
     content gets *written up* in the human-facing report and Doc, not what's captured or how it's classified.
     Reference the matching `bug-screenshots/` image inline in the narration (e.g. "shown below, circled in
     red") rather than as a separate caption line. **Never use an em dash (`—`) anywhere in this report or
     the Google Doc export** - use a plain hyphen with spaces (` - `) instead, or restructure with a comma,
     colon, or period; check with `grep -c "—" quick-scan-report.md` (and a `find_and_replace_doc` pass on
     the Doc) before considering the output done, rather than trusting it was typed correctly throughout.
  4. **UI Nitpick Summary** - a rollup of the nitpick-sweep findings from Step 6.
  5. **Overall Risk Note** - a short paragraph (not a /100 scorecard) characterizing the overall risk level
     given what was found.
  6. **Recommendation** - a three-way call: **Ready** / **Needs Fixes** / **Not Ready** (simpler than
     classic mode's four-way scorecard-derived recommendation, since there's no scorecard to derive it from).
- **Google Docs export - mandatory, automatic, not opt-in.** The moment `quick-scan-report.md` and
  `findings.json` are written, immediately continue in the same turn and produce the Google Doc - don't wait
  for the user to ask for it. Check `claude mcp list` for `google-workspace ... ✔ Connected` without being
  told to. If connected, reuse `SKILL.md`'s existing "Exporting to Google Docs" section verbatim (same tools,
  same setup gotchas, same Drive-upload-then-`insert_doc_image` embedding pattern) - nothing about that
  section is SOP-specific, it applies to any markdown report - using `bug-screenshots/` as the source for
  every inline image. If `google-workspace` isn't connected or fails, say so explicitly in Step 8 rather than
  silently ending the run with only local files.

## 8. Present

Summarize for the user: scope (`page_url`, depth `N`, pages visited, role), finding counts by severity, and
the three-way recommendation (Ready / Needs Fixes / Not Ready). **Include the Google Doc's link** (produced
automatically in Step 7), or state plainly that it was skipped and why (e.g. `google-workspace` not
connected) if so.
