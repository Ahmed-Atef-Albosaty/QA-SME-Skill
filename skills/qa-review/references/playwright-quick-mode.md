# Quick mode - Playwright MCP, single page + subsidiaries

Triggered by `/qa-review quick <page_url> <email> <password>`. This is a **self-contained** procedure - for
a quick-mode run, follow this file in full instead of `SKILL.md`'s Steps 1–6. Nothing here reads from
`references/coverage-areas.md` or `references/report-template.md` (those are SOP-scorecard-specific and
don't apply to this mode).

## This is NOT `/qa-run`

`/qa-run` (a separate command, `.claude/commands/qa-run.md` backed by `tools/qa/run.py` and friends) is a
**script-driven** architecture: raw Playwright-Python library, no turn-by-turn MCP tool calls, results written
to a Google Sheet, a hard 100K-token budget, and it explicitly forbids MCP-driven step-by-step testing.

Quick mode is the opposite of that on every one of those axes: it stays **fully conversational, turn-by-turn
MCP tool calls**, exactly like the rest of `qa-review` - it uses the official `@playwright/mcp` server (the
same one classic mode uses) and a smaller, auto-derived scope instead of a full-platform inventory. If you
find yourself reaching for a Python script, a Google Sheet, or a token-budget cutoff while running this mode,
stop - that's `/qa-run`'s pattern, not this one.

## One-time setup

Check the server is connected:
```
claude mcp list   # look for "playwright: npx @playwright/mcp@latest ... - ✔ Connected"
```
If missing, `claude mcp add playwright -- npx @playwright/mcp@latest --caps=storage,testing,devtools`. **A
freshly-added server's tools only appear in a new session** - same rule as any other MCP server; tell the
user to restart if it was just added.

**This server's core interaction model is accessibility-snapshot + stable `ref`.** `browser_snapshot()`
returns the page's accessibility tree as structured text with a short stable id (`ref: "e12"`, etc.) per
interactive element - every action tool takes that `ref` directly, no manual CSS/XPath locator-guessing
required. Every instruction below that talks about "finding a control" means: call `browser_snapshot()`
first, read the `ref` off what it shows (pass it via each action tool's `target` param), and only fall back
to a raw HTML read (`browser_evaluate`) if a control genuinely has no useful accessible name in the tree.

Load the tool schemas before first use (deferred like any MCP tool):
```
ToolSearch({query: "select:mcp__playwright__browser_navigate,mcp__playwright__browser_click,mcp__playwright__browser_type,mcp__playwright__browser_fill_form,mcp__playwright__browser_select_option,mcp__playwright__browser_hover,mcp__playwright__browser_press_key,mcp__playwright__browser_file_upload,mcp__playwright__browser_wait_for,mcp__playwright__browser_handle_dialog,mcp__playwright__browser_tabs,mcp__playwright__browser_snapshot,mcp__playwright__browser_find,mcp__playwright__browser_take_screenshot,mcp__playwright__browser_console_messages,mcp__playwright__browser_network_requests,mcp__playwright__browser_verify_text_visible,mcp__playwright__browser_verify_element_visible,mcp__playwright__browser_highlight,mcp__playwright__browser_hide_highlight,mcp__playwright__browser_run_code_unsafe"})
```

## Core tool set used in this mode

- `browser_navigate(url)` / `browser_snapshot()` - navigate and read the page's current interactive structure.
- `browser_click({target, element?, button?, modifiers?})`, `browser_type({target, text, submit?})`,
  `browser_fill_form({fields: [{target, name, type, value}, ...]})`, `browser_select_option({target, values})`,
  `browser_hover({target})`, `browser_press_key({key})`, `browser_file_upload({paths})` - all take a `target`
  (the snapshot's ref, or a unique selector) and auto-wait for real actionability before acting; no manual
  settle-check needed for the common case.
- `browser_wait_for({time?|text?|textGone?})` - condition-based waiting; prefer `text`/`textGone` over a fixed
  `time` wherever a distinctive marker string exists.
- `browser_verify_text_visible({text})` / `browser_verify_element_visible({role, accessibleName})` - auto-
  retrying assertions for a clean PASS/FAIL read instead of a raw DOM dump (read `role`/`accessibleName`
  straight off a snapshot line, e.g. `button "Sign in" [ref=e29]` -> role `button`, accessibleName `Sign in`).
- `browser_take_screenshot({filename?})`, `browser_tabs({action: "list"|"new"|"close"|"select", index?})`,
  `browser_console_messages({level})`, `browser_network_requests({filter?, static?})`.
- `browser_highlight({target, element?, style?})` / `browser_hide_highlight({target, element?})` - draws a
  visual overlay directly on a live element (pass `style: "outline: 4px solid red; outline-offset: 2px;"` for
  a clear red box, confirmed live to render exactly as given) - the default bug-evidence-annotation recipe
  (see Step 7 below).
- `browser_evaluate({function, target?, element?})` - runs a JS function in-page (optionally scoped to one
  element via `target`), NOT RCE-flagged - the default tool for DOM reads: a link scrape, a raw DOM
  text-node walk for a CSS `text-transform` trap, a `getBoundingClientRect()` read for the fleeting-bug
  annotation technique.
- `browser_run_code_unsafe({code: "async (page) => {...}"})` - a different, more powerful tool that runs
  against Playwright's own `page` object (`--caps=devtools`, flagged RCE-equivalent) - reserve for the rare
  case you need real Playwright APIs, not as the default JS-eval tool.
- `browser_handle_dialog(accept, promptText?)` - explicit native-dialog control; auto-dismissed by default if
  unhandled. **Known open gap: an action that triggers a dialog can hang with no notification** - if a click
  unexpectedly hangs, that's the likely cause; there's no fix beyond knowing in advance which controls are
  genuinely native dialogs (most in-page "Delete?" UIs are just modals with their own Confirm/Cancel buttons,
  not native dialogs at all, so don't pre-arm `browser_handle_dialog` for those).

## Operational gotchas confirmed in live use

**Auto-waiting removes most of the old settle-check-and-retry discipline.** Every action tool already waits
for visible+stable+enabled+receiving-events before acting, within `--timeout-action` (recommend `3500`ms for
fast-but-comfortable clicks) - don't add a manual check before or after most actions. Reach for
`browser_wait_for({text:...})` after a navigation/content change instead of guessing whether something loaded.

**A native `confirm()`/`alert()` dialog auto-dismisses by default** unless you've called
`browser_handle_dialog` in advance - so a delete/confirm-style control implemented with a plain JS `confirm()`
won't block the session the way it did with a different, more primitive browser MCP tool; it'll just silently
proceed as if dismissed. If a "Delete" action is supposed to prompt for confirmation and nothing seems to
happen, check whether it's a native `confirm()` (auto-dismissed, so the delete may have been silently
cancelled) vs. an in-page modal (should still render and need a real click on its own Confirm button).

Two other false-negative patterns showed up repeatedly on real runs with earlier browser MCP tooling. The
underlying discipline still applies here - don't assume Playwright MCP is immune to the same *class* of
problem just because the specific tool changed:

- **The first visit to a client-side sub-section (e.g. a channel/tab) in a session may open a first-visit
  "About this X" onboarding modal**, which blocks interaction with the underlying content until dismissed. A
  click on the nav control itself can appear to do nothing if a modal is intercepting it - re-check
  `browser_snapshot()` for a visible modal and its dismiss button (often "Got it" or similar) before
  concluding the click didn't work; dismiss it, then proceed - don't write this up as a dead-click finding.
- **Playwright's `browser_click` targets a `ref` from a live snapshot and auto-waits for real actionability**,
  which removes most of the old synthetic-click-artifact risk (a click reported as "successful" with zero
  real effect and zero network request - a failure mode confirmed for a different, more primitive MCP
  browser tool's synthetic click delivery). Keep a lighter version of the verification discipline anyway: if
  a click should have visibly changed something but didn't, check `browser_network_requests` (filtered to the
  expected endpoint) before concluding it's a dead click - and if you have specific reason to suspect a stale
  `ref` (clicked from a snapshot taken before an intervening DOM change), re-`browser_snapshot()` and retry
  once against the fresh ref first.

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
  tiers/routes, and any `## Playwright QA Config` block - device presets, storage-state paths, ready markers)
  and check for a `## QA Environment` section with `Scratch/dev confirmed: YES` (see next bullet). **Never
  require creating a spec file for quick mode** - if none exists, proceed without one, and only create/append
  one later if new subsidiary routes are actually found (see Discovery below).
- **Scratch/dev confirmation** - identical rule to classic Step 1: if `platform-specs/<slug>.md` already has
  `Scratch/dev confirmed: YES`, treat it as confirmed and don't re-ask. Otherwise ask the user directly; once
  confirmed, write it back to `platform-specs/<slug>.md` (creating the file only at this point, if it didn't
  already exist) so future runs against this slug don't need to ask again. If not confirmed (or the user
  declines), run **read-only only** - mark every write-path item UNVERIFIED rather than guessing, exactly as
  the classic path does.
- **No parallel-mode question.** Quick mode is always single-agent, single browser - there's one credential
  set, so the classic Step 1b (classic vs. 3-worker parallel) doesn't apply and isn't asked.

## 2. Login

1. `browser_navigate({url: page_url})` (or a launch with `--device="Desktop Chrome" --viewport-size=1920,1080`
   already applied via the server's launch flags, for the initial discovery/desktop pass) - if a separate
   login URL exists, navigate there first (or let the redirect to a login form happen naturally).
2. `browser_snapshot()` to find the email/password fields and submit control by accessible name/role (see
   "Per-role login" in `playwright-crawl-procedure.md` for the exact recipe) - don't guess a CSS pattern
   before checking the snapshot.
3. `browser_fill_form({fields: [{target: "<email ref>", name: "Email address", type: "textbox", value: email},
   {target: "<password ref>", name: "Password", type: "textbox", value: password}]})` - one call for both
   fields, no ordering dependency between them.
4. `browser_click({target: "<submit ref>", element: "submit button"})`, or `browser_press_key({key: "Enter"})`
   while the password field is focused.
5. Confirm login succeeded with `browser_wait_for({text: "<a marker unique to the logged-in state>"})` rather
   than a manual settle-check - if you don't yet know a good marker for this app, one immediate
   `browser_snapshot()` read after the click is enough to confirm the login form is gone.
6. Same session-sharing gotcha as classic mode applies: logging into a second account in the same browser
   profile replaces the first session everywhere. Quick mode only ever uses one account, so this shouldn't
   come up - but if you ever need to spot-check something as a different login mid-run, expect to log back in
   afterward.

## 3. Discover - "current page + its subsidiaries"

This replaces classic Step 2a's whole-platform inventory with a bounded, same-origin BFS from `page_url`.

**Critical caveat, confirmed in live use: a pure `<a href>` same-origin scrape can badly mis-scope this step
if the given page's own sub-structure is client-side (buttons/tabs), not real links.** An early discovery
pass on a community-style app scraped `<a href>` elements and found nothing but the site-wide global sidebar
nav - present identically on every page and not specific to the given page at all - while completely missing
the page's actual, meaningful subsidiaries: a multi-channel switcher and a Resources/Knowledge-Base panel,
both implemented entirely as `<button>` elements with no `href`. This is a DOM-structure fact independent of
which MCP server drives the session, so it applies exactly the same way here. **Before or alongside the
href-based BFS, always inspect the given page's own accessibility snapshot for a sidebar/tablist/panel-switcher
of buttons that represent that page's own sub-sections** (the same "consolidated admin panel" pattern classic
mode's Step 3 warns about) - if present, treat *those* as the primary subsidiaries for this page, not just
whatever a handful of `<a href>` elements happen to return. Confirm with the user (per Step 4) which
interpretation matches what they meant by "subsidiaries" if it's ambiguous - don't silently default to the
shallow link-scrape and call the run complete.

1. Start a queue with `page_url` at depth 0. Visited-set starts empty.
2. While the queue is non-empty, depth ≤ `N`, and visited count < 60:
   - Pop the next URL. If already visited, skip. `browser_navigate({url})`.
   - Extract same-origin links via `browser_evaluate`, with the origin baked into the function body:
     ```
     browser_evaluate({function: "() => [...document.querySelectorAll('a[href]')].map(a => a.href).filter(h => h.startsWith('https://app.example.com'))"})
     ```
     and read the page's own `browser_snapshot()` for client-side tablist/button subsidiaries per the caveat
     above.
   - Record this URL in the inventory (URL, short descriptive name from the page title/heading - a
     `browser_snapshot()` read is usually enough).
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
viewport-outer-loop, same restructuring as classic mode (see `playwright-crawl-procedure.md`'s "Per-screen
capture"). Playwright's `--device` presets set viewport+DPR+touch+UA all atomically, no manual hacking
needed:

For each of the three viewports, in turn:
1. Close the previous viewport's session (skip on the very first viewport), then start a new one with this
   viewport's device preset:
   - **Desktop:** `--device="Desktop Chrome" --viewport-size=1920,1080`.
   - **Tablet:** `--device="iPad Pro 11"` - real device dimensions, DPR, and touch emulation, not a manual
     window-size+UA hack.
   - **Mobile:** `--device="iPhone 15"` (393×852) - a real device preset, not a workaround viewport.
2. Log in once for this viewport-session (per Step 2 above).
3. For every in-scope page:
   - `browser_navigate({url})`, then go straight to the screenshot/read calls below - auto-waiting handles the
     common settle case. Only add a `browser_wait_for({text:...})` first if you have a known ready-marker or
     the capture below comes back visibly empty/loading.
   - `browser_take_screenshot({filename: "qa-runs/<slug>/screenshots/<role>_<screen-slug>_<viewport>.png"})`.
   - `browser_snapshot()` for the interactive-element list (replaces the old manual controls-scrape); if you
     also want a raw HTML capture, `browser_evaluate({function: "() =>
     document.documentElement.outerHTML"})` written to `qa-runs/<slug>/html/<role>_<screen-slug>.html`.
   - **Skip `browser_console_messages`/`browser_network_requests` unless this specific page gives a reason to
     investigate** (a dead-looking click, a broken image, a form that didn't seem to save) - not routine
     per-page overhead. Both default to "since last navigation," so you get a clean scope without an explicit
     clear step.
4. Move to the next viewport and repeat from step 1.

This means login happens once per viewport-session (three times total across the run), not once total.

**Safe mode still applies exactly as classic mode:** open forms and probe fields, but never click
submit/delete/logout/deactivate/pay/send/purge/confirm controls unless scratch/dev is confirmed and you're
deliberately testing one specific write action.

## 5b. Interact - every functional control on the page MUST be exercised, not just observed

**This is a hard requirement, not best-effort, and it applies to every quick-mode run regardless of scope - partial or total.** A run that only navigates to each in-scope page, takes screenshots, and clicks nav/tab
controls to switch views is **not a complete quick-mode run**, even if every page was visited and
screenshotted. Looking at a screen is not the same as testing it.

**Incident that motivated this rule:** on an early quick-mode run against a community-style app page, the
entire pass consisted of clicking channel tabs and capturing screenshots - the post composer
(Post/Photo/Video/GIF/Emoji), reaction buttons, comment threads, the search box, and every Edit/Delete control
on Resource library/Video library/Knowledge Base were all visually observed and screenshotted but never
actually clicked or exercised. The write-up read as a complete functional QA pass when it was really a UI
tour. The gap was only caught because the user asked directly whether posting, reacting, and commenting had
actually been tested. Treat that question as one this mode must always be able to answer "yes" to,
unprompted - regardless of which browser MCP server is driving the session.

Concretely, before writing up any page's assessment as finished:
1. From that page's `browser_snapshot()` (or a targeted `browser_evaluate` sweep), enumerate every
   interactive control - every button, input, toggle, menu item, and link that isn't pure navigation. This
   includes (not exhaustively): post/comment composers, reaction pickers, comment-thread open/reply,
   search/filter inputs, sort/view toggles, per-item action menus (Edit/Delete/Move/Hide/Pin/etc.), file/
   image/video upload controls, and any "About this X" or settings dialogs reachable from the page.
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
- **Dead-click / false-positive verification**, adapted to Playwright's ref-based model:
  1. If a click should have visibly changed something but didn't, check `browser_network_requests` (filtered
     to the expected endpoint) - only trust it as a real failure if a request genuinely never fired.
  2. If you suspect a stale `ref` (clicked from a snapshot taken before an intervening DOM change),
     re-`browser_snapshot()` and retry once against the fresh ref before concluding anything.
  3. If it still fails after that, the finding is unconfirmed, not proven - note it as such and, if possible,
     ask for a manual spot-check rather than writing it up as a confirmed Critical bug.
  4. Same discipline for a "text/copy missing" claim: search `document.body.innerText` or a container's
     `.textContent` (via `browser_evaluate`), never a leaf-node-only query - see
     `references/playwright-crawl-procedure.md`'s "Before marking UI text/copy missing" section for the full
     reasoning (it applies regardless of which browser MCP is driving the session).
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
  is otherwise final. Default recipe (`--caps=devtools`):
  1. With the buggy element still on screen, `browser_snapshot()` to get its ref (e.g. `e71`).
  2. `browser_highlight({target: "e71", element: "<short description>", style: "outline: 4px solid red;
     outline-offset: 2px;"})` - draws the overlay directly on the live element at real device pixels
     (confirmed live: the custom `style` param renders exactly as given). No manual
     `getBoundingClientRect()`/`devicePixelRatio` scaling needed at all.
  3. `browser_take_screenshot({filename: "..."})`, copy into `bug-screenshots/` per the folder discipline
     above.
  4. `browser_hide_highlight({target: "e71", element: "<same description>"})` before the next capture.
  **Two cases still need a manual fallback via `browser_evaluate` instead of `browser_highlight`:**
  - **CSS `text-transform`'d text with no matching accessibility node** (e.g. an uppercased heading whose real
    DOM text is mixed-case) - walk text nodes directly
    (`document.createTreeWalker(document.body, NodeFilter.SHOW_TEXT)`) and use
    `document.createRange().selectNodeContents(node).getBoundingClientRect()` on the raw text node, then draw
    the box at that (HiDPI-scaled) rect with a minimal image library.
  - **A fleeting/timing-based bug** (e.g. a flash-of-unstyled-content that self-corrects in 1-3s) - grab the
    *container's* stable rect from the resolved state (any time), then apply that same box to the screenshot
    that happened to catch the bug mid-glitch; the box still lands in the right place since the layout region
    doesn't move.
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
