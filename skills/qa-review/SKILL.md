---
name: qa-review
description: "Run a complete ISTQB-based release-readiness + product/UX review of a running LMS/web app, enforcing the ScalingEasy QA SOP coverage-area criteria. Drives a real browser via Microsoft's official Playwright MCP server to first discover every screen/custom page on the platform, then lets the user choose a full QA (every screen) or a partial QA scoped to specific pages/areas/custom features, logging in per role and visiting the in-scope screens across desktop/tablet/mobile, capturing screenshots/HTML, then assessing the six hats + the SOP per-area checklists (ADMIN/TIER/CUSTOM) and writing a Release-Readiness Report plus a filled criteria sheet. Also supports a lighter 'quick mode' (`/qa-review quick <page_url> <email> <password>`) that uses the same Playwright MCP server but scans just a given page and its linked subsidiary pages, writing a findings-only quick-scan report - no SOP scorecard, no platform-wide discovery. Use before promoting a finished phase (or a single page/feature fix) to production, or when asked to QA / review / assess an application or a specific page/feature for release, or to quickly sanity-check one page and what it links to."
---

# /qa-review

A single, standardised assessment that merges two reviews into one: the **Product/UX Application Review**
and the **ISTQB QA Release-Readiness Review** (six hats - PM, UX, UI, QA, Accessibility, End User), enforced
against the **ScalingEasy QA SOP coverage-area criteria** (per-area check items, ADMIN/TIER/CUSTOM tag
rules, desktop/tablet/mobile verdicts). Run it after a phase is finalised, before migrating to production.

## Usage
```
/qa-review
/qa-review <slug> <base_url>
/qa-review quick <page_url> <email> <password>
```

## Reference files
All paths below are relative to this skill's directory (`references/`). Read the relevant file(s) at the
point the step below tells you to - don't rely on memory of what they contain, and don't skip opening one
because its content seems inferable from the section title.
- **`references/parallel-mode.md`** - required reading the moment 3-worker parallel mode is chosen in Step
  1b. Defines the persona-worker design (admin + 2 tier/role workers), the 3-server Playwright MCP
  prerequisite, and the merge/audit/write flow back into Steps 4c/5/6. If you're about to run parallel mode
  and haven't opened this file yet in this conversation, stop and read it first - do not improvise the
  persona-worker split from the summary in Step 1b/2/4b alone.
- `references/playwright-crawl-procedure.md` - full Playwright MCP tool-by-tool crawl procedure, covering both
  the lightweight discovery pass (Step 2a) and the full per-viewport capture pass (Step 2c); read before Step 2a.
- `references/coverage-areas.md` - the 8 SOP coverage areas + per-area check items; read before Step 2a and Step 3.
- `references/review-dimensions.md` - ISTQB six-hats review dimensions §3.1–3.10; read before Step 4.
- `references/finding-types.md` - finding classification (type/category/severity/priority); read before Step 4.
- `references/istqb-principles.md` - safe-testing rules and golden-rule mindset; read before Step 2a and Step 4.
- `references/report-template.md` - exact structure for the three Step 5 output files.
- **`references/playwright-quick-mode.md`** - the entry point for **quick mode**
  (`/qa-review quick <page_url> <email> <password>`). Self-contained: for a quick-mode run, follow this file
  in full **instead of** Steps 1–6 below, which describe the classic full-platform review only.

## Steps

### 0. Mode dispatch
If invoked as `/qa-review quick <page_url> <email> <password>`, this is **quick mode**: follow
`references/playwright-quick-mode.md` in full instead of Steps 1-6 below. Quick mode is a self-contained
procedure with its own lighter input-gathering, discovery, capture, assessment, and output steps - it uses
the same `playwright` MCP server as classic mode below, just scoped to the given page plus its linked
subsidiary pages (not a full-platform inventory), and produces a findings-only quick-scan report (not a SOP
criteria-sheet/scorecard). It stays fully conversational/MCP-driven throughout - never delegate any part of it
to a script or to `/qa-run`'s architecture (a different, script-driven command; see the quick-mode file's
"This is NOT `/qa-run`" section for why the two must stay distinct).

Otherwise (invoked as `/qa-review` or `/qa-review <slug> <base_url>`), this is the classic full-platform
review: continue with Step 1 below.

### 1. Gather inputs
Ask the user for:
- **Platform name + slug** (used for `qa-runs/<slug>/`) and **base URL**.
- **Credentials per role** - at minimum an **admin** and a **standard member** account; add one login per
  **tier** if TIER rows need to be exercised (`[{"label":"admin","tier":null,"email":...,"password":...}, {"label":"member","tier":"tier_lite","email":...,"password":...}]`).
- **The platform spec** - check for `platform-specs/<slug>.md` in the current repo first; it tells you
  which tiers exist and which features are **CUSTOM** for this client. Ask the user if no spec exists. If the
  spec has a `## Playwright QA Config` block (device presets, storage-state paths, ready-markers, slow
  endpoints, dialog controls, complex-feature sequences - see `references/playwright-crawl-procedure.md`),
  read it now; it drives Step 2a's launch options and saves rediscovering the same app-specific quirks every
  run. If no such block exists yet, one gets built up as this run learns the platform (see the write-back
  convention in Step 2a).
- Any deep-link start paths worth seeding the crawl with (e.g. `/community`, `/settings`).
- **Don't ask which pages/features to test yet.** Full vs. partial scope is decided in Step 2b, after the
  discovery crawl has actually enumerated what's on the platform - asking here would just be guessing blind.
- **Confirm this is a scratch/dev environment**, not shared/production data - but check
  `platform-specs/<slug>.md` for a `## QA Environment` section with `Scratch/dev confirmed: YES` first;
  if present, treat it as already confirmed for this platform and skip re-asking. If no spec exists or it
  doesn't have that section, ask the user, and once they confirm, **write it back** to
  `platform-specs/<slug>.md` (create the spec file if it doesn't exist yet) so future runs against this same
  platform don't need to ask again. If confirmed NOT scratch (or the user declines to confirm): run
  read-only only, and mark every write-path item (Add/Edit/Delete rows in the coverage checklists)
  **UNVERIFIED** rather than guessing a verdict. See `references/istqb-principles.md` for the full
  safe-testing rules.

### 1b. Run mode: classic single-agent vs. 3-worker parallel
Ask this once, right after Step 1's inputs are gathered and before starting Step 2a:

> "Run classic single-agent (one browser, fully sequential), or 3-worker parallel (one persona per
> worker - admin + 2 tiers/roles - each with its own isolated browser, crawling and write-testing
> simultaneously)? Parallel is roughly 2-3x faster wall-clock but costs an estimated 20-35% more tokens
> (duplicated per-worker setup, plus a merge pass that doesn't exist in sequential mode), and needs a
> one-time 3-server Playwright MCP setup plus per-run opt-in for the `Workflow` tool."

- **Default to classic single-agent** if the user doesn't have a preference or this is a quick/small run.
- **If parallel is chosen:** this requires the `Workflow` tool, which needs the user's opt-in each time
  (an "ultracode" keyword, or an explicit ask like "use a workflow") - it cannot self-trigger. If that
  opt-in isn't already active this turn, tell the user the exact phrase that unlocks it and wait for them
  to re-ask with it - don't silently invoke `Workflow` anyway. Once opt-in is confirmed, follow
  `references/parallel-mode.md` for the full persona-worker design, the 3-server prerequisite, and the
  merge/audit/write flow. Steps 2a-4b below describe the classic single-agent path; parallel mode replaces
  Step 2c and Step 4b with the persona-worker phases in `references/parallel-mode.md` (Steps 2a and 2b - discovery and scope decision - always run once, in the main/orchestrator turn, before any persona-worker
  is spawned, in both modes), but Steps 4c, 5, and 6 (final audit, write outputs, present) are unchanged and
  always run once, in the main/orchestrator turn, regardless of which mode was used to gather the underlying
  data.

### 2a. Discover - lightweight inventory crawl
**This step always runs, in full, before any scope decision - this is the "explore the whole website first"
pass.** It runs unconditionally regardless of whether the run ends up full or partial, and (in 3-worker
parallel mode) runs once in the main/orchestrator turn before any persona-worker is spawned - see
`references/parallel-mode.md`.

This skill drives a real browser turn-by-turn through Microsoft's official **Playwright MCP server**
(`@playwright/mcp`), not a background script. Full tool-by-tool procedure (setup check, login pattern, the
discovery-pass loop, endpoint discovery techniques) is in `references/playwright-crawl-procedure.md`'s
"Discovery pass" section - read it before starting.

Quick summary: confirm `playwright` shows Connected in `claude mcp list` (add it with
`claude mcp add playwright -- npx @playwright/mcp@latest --caps=storage,testing,devtools` if missing; note a
freshly-added server needs a new session before its tools appear); `ToolSearch` for the `mcp__playwright__*`
tools; log in per role (this server addresses elements by a stable `ref` returned from `browser_snapshot()`,
not a manual CSS/XPath locator - see `references/playwright-crawl-procedure.md` for the full login recipe);
then for every endpoint discovered (prefer a manually-provided or spec-listed endpoint list over pure
link-following - hidden-nav admin/staff panels are easy to miss otherwise), `browser_navigate` + a lightweight
`browser_snapshot()` read at the Desktop device preset is enough for this pass - **no tablet/mobile session,
no screenshot files, no console/network capture yet, that's Step 2c and only for whatever ends up in scope.**
As you go, build the **screen inventory**: for every screen, its URL/path, a short descriptive name, which of
the 8 SOP areas it maps to (or **Custom** if none), and which role is required to view it - this is the same
screen→area mapping Step 3 does today, just done here, up front, and lightly.

**URL/route discovery - don't rely on visible nav links alone.** A normal click-through of the top nav will
miss routes that only render behind a click (query-param tabs, modals, deep-linked settings pages). Before
concluding the endpoint list is complete, actively try these, roughly in order of effort:
1. Ask the user for an endpoint list or check `platform-specs/<slug>.md` first - cheapest and most reliable.
2. Look for a `sitemap.xml` at the site root - rarely populated for gated apps but free to check.
3. For a Next.js/React app, a `browser_network_requests` read or a `browser_evaluate` fetch of
   `/_next/static/...` chunk source can reveal route strings not currently reachable via UI - use if you
   suspect hidden routes and the above two options come up empty.
3b. **If the platform spec documents a backend API (e.g. Supabase project URL + anon key), query it
   directly** - a read-only `GET {url}/rest/v1/` (PostgREST) or equivalent schema/discovery call enumerates
   every exposed table/view/RPC, which reveals feature surfaces (users, tiers, channels, tickets, courses,
   etc.) and their route/key naming even when nothing in the front end currently links to them. This is
   safe recon (GETs only) and often finds routes nav-following and bundle inspection both miss - see
   `references/playwright-crawl-procedure.md` "Endpoint discovery" for the full technique.
4. **Systematically enumerate every tab inside every panel you land on** - this is usually the highest-value
   step. A single "Manage"/"Admin"/"Settings" entry point often hides many distinct sub-panels behind a
   tablist or sidebar (see the consolidated-admin-panel note in Step 3) that a generic crawler or browser
   extension (e.g. a "grab all links" tool) won't find, because they only exist as client-side tab state, not
   separate URLs with their own links. Click every tab, not just the one you happened to land on.
A generic link-scraping browser extension only captures what's in the current page's rendered DOM - it won't
discover role-gated or click-to-reveal panels, so don't rely on one as your primary discovery method for an
authenticated SPA like this; use it (if at all) as a supplement to the steps above, not a replacement.

Every custom/client-specific page found gets equal billing in this inventory, listed by name alongside the
8 SOP areas - this is what Step 2b offers the user as a selectable scope item, and it's exactly the kind of
surface a generic checklist or nav click-through would otherwise miss.

**Write newly-found endpoints back to the platform spec before moving on.** Once the inventory is built,
diff it against `platform-specs/<slug>.md`'s existing endpoint list (create the section if the spec has none
yet): any route discovered this pass that isn't already listed gets appended, in the same format as the
existing entries (path + short description, noting `*hidden nav*` for anything not linked from visible nav
per the spec's own convention). Never remove or rewrite an existing entry - only add what's missing, so a
manually-curated note on a route survives future runs. This is what lets step 1 of the *next* run against
this platform read the spec's list first (per the "URL/route discovery" priority order above) instead of
re-discovering the same hidden-nav routes from scratch every time. Skip this write-back only if no spec file
exists and the user hasn't asked for one to be created.

**Also append any newly-learned Playwright config facts to the same spec's `## Playwright QA Config` block**
(create the block if it doesn't exist yet) - a `ready_markers` entry for a screen's distinctive "loaded" text
(used by `browser_wait_for` instead of guessing), a `slow_endpoints` entry if a path needed a longer
per-action timeout than the global default, or a `dialog_controls` entry if a control turned out to trigger a
genuinely native browser dialog. Same append-only discipline as the endpoint list. See
`references/playwright-crawl-procedure.md` for the full schema.

### 2b. Choose scope - full QA or partial QA
Present the discovered inventory to the user, grouped by SOP area with each custom page listed separately,
and ask once:

> "Full QA (every area + custom page above), or partial QA on specific pages/areas/features? You can pick
> whole SOP areas (e.g. all of Community/Channels), individual pages (e.g. just `/community`), and/or
> specific custom pages (e.g. the AI Coach page) - any mix."

- **Full QA:** scope = everything discovered in 2a. Proceed exactly as before this change.
- **Partial QA:** scope = only the selected areas/pages/custom features. Record the scope explicitly and
  echo it back for confirmation (e.g. "Running partial QA on: Community/Channels, Custom: AI Coach - proceeding") before continuing to 2c. Every later step (3, 4, 4b, 4c, 5, 6) operates only on this scope - out-of-scope areas are not visited again, not assessed, and not written into any output file for this run.
- This decision is made once, in the main/orchestrator turn, **before** any persona-worker is spawned if
  3-worker parallel mode was chosen - see `references/parallel-mode.md`. Each worker then receives only the
  in-scope screen list, never the full inventory.

### 2c. Full capture - per in-scope screen, via Playwright MCP
**If 3-worker parallel mode was chosen in Step 1b, skip this step as written and follow
`references/parallel-mode.md` instead** - each persona-worker does its own version of this step, scoped to
the same in-scope screen list from Step 2b, inside its own isolated browser. What follows is the classic
single-agent path.

Iterate only over the screens confirmed in scope in Step 2b (all of them, for a full run). Playwright MCP
still has no mid-session device-resize call, so viewport stays fixed for a browser session's lifetime - the
loop is **viewport-outer, screen-inner**: launch a session with the `"Desktop Chrome"` device preset
(1920×1080), log in, capture every in-scope screen, close the session; then `"iPad Pro 11"`; then
`"iPhone 15"` (393×852 - a real device preset, not a workaround; see `references/coverage-areas.md`). Each
screen's screenshot + a `browser_snapshot()` read (for interactive elements, replacing the old manual
controls-scrape) - writing screenshots to `qa-runs/<slug>/screenshots/` with the
`<role>_<screen-slug>_<viewport>.png` naming convention and HTML to `qa-runs/<slug>/html/` if captured. Full
tool-by-tool procedure, including the exact launch options per viewport, is in
`references/playwright-crawl-procedure.md`'s "Per-screen capture" section.

**Safe mode still applies:** open forms and probe fields, but never click submit/delete/logout/deactivate/
pay/send/purge/confirm controls without the scratch/dev confirmation below.

**Every write-capable control on an in-scope screen MUST actually be exercised once scratch/dev is
confirmed - not just visually observed, and not left as "probe the field, never submit."** A criteria-sheet
row marked PASS/FAIL/N/A on the strength of looking at a form, a post composer, a reaction button, a
comment box, or an Add/Edit/Delete control - without ever having actually clicked it - is not a completed
check; mark it UNVERIFIED instead and say so, rather than guessing a verdict from appearance alone. Concretely,
for every such control found on an in-scope screen: perform the action for real (post, react, comment,
search, add/edit/delete a record), confirm the result (ideally via a reload, not just an in-session visual
check), and - for anything you created solely to test with - delete/revert it afterward per the
disposable-test-entity discipline, the same way a real write ("does Add Meeting actually save?") is already
handled below. Only a control that is genuinely destructive/irreversible against **shared, non-scratch**
data (or one this run isn't scoped to reach) may be left as "observed, not exercised" - and even then, say so
explicitly in the report rather than letting the gap pass silently. If time runs out before every control on
every in-scope screen has been exercised, state exactly what wasn't reached rather than implying full
coverage.

**Every criteria-sheet check and every custom-feature check must be forcibly tested end-to-end before it
can be marked FAIL - a single click with an immediate visual check is not sufficient and produces false
negatives.** A control that looks like a dead click is just as likely to be a testing artifact (page not
yet hydrated) as a real defect - before concluding a control is broken:
1. **Trust Playwright's built-in auto-waiting by default - don't hand-roll a settle-check.** Every action
   tool (`browser_click`, `browser_type`, `browser_fill_form`, etc.) already waits for the target to be
   visible/stable/enabled/receiving-events before acting, within `--timeout-action`. This replaces the old
   manual "check once, retry once" discipline for the common case - see
   `references/playwright-crawl-procedure.md`'s Speed section for the full decision tree (when to add
   `browser_wait_for`, when to use `browser_verify_text_visible`/`browser_verify_element_visible`, and the two
   cases that still genuinely need a fixed-time wait).
2. **Check `browser_network_requests` (filtered to the endpoint you'd expect) immediately after a click that
   should have done something but visibly didn't.** No request at all is real evidence of a dead click; a
   request that fired but produced no obvious UI change might just mean the app doesn't render feedback for
   that action - don't conflate the two.
3. **Verify persistence with a real reload**, not just an in-session visual check, for anything that's
   supposed to save (settings, profile edits, created records) - an optimistic-UI update can look like
   success while never having reached the server.
4. **Retry once with a fresh navigation** before filing a FAIL - reproduce it twice, not once, exactly as
   you would for a magic-link or other backend-triggered defect. A control that fails once but works on
   retry is not a confirmed bug.
Only after these steps still show no effect (no request fired, no state change, no persistence) should a
row be marked FAIL. If time is genuinely short, it's better to mark a row UNVERIFIED with a note than to
mark it FAIL off a single fast click.

5. **Playwright's `browser_click` targets a `ref` from a live snapshot and auto-waits for real actionability,
   which removes most of the old synthetic-click-artifact risk** (a click reported as "successful" with zero
   real effect and zero network request - a failure mode previously confirmed for a different, more primitive
   MCP browser tool's synthetic click delivery). The dual-click verification dance that used to be mandatory
   here shrinks to one check: if you have specific reason to suspect a stale `ref` (you clicked something from
   a snapshot taken before an intervening DOM change), re-`browser_snapshot()` and retry once against the
   fresh ref before concluding anything - don't assume a "click did nothing" result is real without at least
   ruling out a stale ref first. See `references/playwright-crawl-procedure.md`'s "Before finalizing any
   dead-click FAIL" section for the full detail.

**Testing a link/button that's supposed to open a new tab (`target="_blank"`, "Apply", "View posting",
etc.) needs a different check than same-page state changes - checking the tab list *after* clicking is
not reliable**, since a newly-opened tab can be missed if it wasn't being watched for at the moment it
opened. Before concluding such a control is broken:
1. First confirm via a quick DOM check whether the element genuinely has a real destination (e.g. an
   `<a>` with a non-empty `href` and `target="_blank"`, or an onclick handler that calls `window.open`).
   If it does, that's strong evidence the control is wired correctly even if a new-tab check comes back
   empty - a real href is not decorative.
2. Call `browser_tabs({action: "list"})` immediately before the click to record the current tab count, then
   click, then call it again - a genuine new-tab open should show up in this before/after comparison, which
   is more reliable than a single post-click check.
3. If both a real href/`window.open` call is confirmed AND a before/after tab-count comparison shows no new
   tab, that's real evidence of a defect. If only the tab-count check is inconclusive but the href/handler is
   confirmed real, don't mark it FAIL - mark it PASS (a verified destination is verified functionality) or
   UNVERIFIED with a note about the tooling limitation, not a bug.

### 3. Confirm coverage gaps within scope
The screen→area mapping itself already happened in Step 2a's discovery pass. Read `references/coverage-areas.md`
if you haven't yet, then check for gaps **within the confirmed scope only**: any in-scope SOP area or
selected custom page with **no matching captured screen** from Step 2c - that's a coverage gap, meaning
either the feature doesn't exist on this platform (fine, will show as N/A) or you missed a hidden-nav route
(check the platform spec's endpoint list, or ask the user for one, and capture it before finishing this
pass). For a partial run, do **not** flag SOP areas/custom pages that were never selected in Step 2b - those
are out of scope by the user's own choice, not a gap.

**Forcibly visiting every in-scope client-specific/CUSTOM page is HIGH PRIORITY - treat it as mandatory, not
optional or best-effort.** For a full run this means every custom page discovered in 2a; for a partial run
it means every custom page the user selected in 2b. Don't leave any in-scope nav item or spec-listed endpoint
unvisited just because it doesn't map to one of the 8 SOP areas. Any in-scope nav link, sidebar section, or
spec-listed route that isn't one of the 8 standard screens (career tools, AI features, simulations,
client-branded sub-products, etc.) still gets a full visit + real interaction attempt (click primary
buttons/CTAs, open forms, exercise the main workflow) - not just a screenshot. These are exactly the surfaces
a generic checklist misses, and they're just as likely (often more likely, being newer/less-tested code) to
carry real bugs - client-specific pages are frequently where the most valuable, most-overlooked findings
live. Treat "I didn't get to the in-scope custom pages" as an incomplete run, not an acceptable gap. If time
is short, prioritize breadth across in-scope custom pages over additional depth on already-covered SOP areas - custom-page coverage should never be the thing sacrificed to save time.

Every custom page gets its own "Custom Feature Areas" section in the criteria sheet (per
`references/coverage-areas.md`'s Custom Feature Areas guidance and `references/report-template.md`'s table
shape) - bugs found there are first-class findings, not an afterthought folded into "Out-of-Criteria Bugs."
Only use the Out-of-Criteria Bugs section for defects that don't map to *any* screen-level section at all
(e.g. a cross-cutting console error); a custom page's own bugs belong in that page's own section.

**Consolidated admin panels hide entire surfaces behind tabs, not separate URLs - enumerate every tab.** Many
apps have a single "Manage"/"Admin Panel" entry point (e.g. `/admin?tab=users`) whose sidebar/tablist switches
between many distinct panels (Command Center, Coaching, Courses, Onboarding, Settings, Analytics, Support,
Surveys, Tiers, Users, etc.) without changing to a separately-linked page you'd discover by following normal
nav. If you find a "Manage" link or an `/admin`-style route, click through **every tab in its tablist** - don't assume that reaching the panel once (e.g. via a "User Management" nav link that happens to land on one
tab) means you've covered the whole thing. A tab with a name that looks like it matches an open criteria row
(e.g. "Onboarding") may turn out to be about something else entirely (e.g. dashboard welcome-video/raffle
settings, not per-user onboarding status) - don't assume the mapping from tab name to criteria row without
actually opening it and checking what's inside.

### 4. Assess - six hats + SOP criteria
**Only assess screens/areas within the confirmed scope (Step 2b).** For a partial run, skip SOP areas and
custom pages that were not selected entirely - do not backfill them with N/A/UNVERIFIED rows; they simply
don't appear in this run's assessment or outputs.

For every captured screen, Read all three viewport screenshots plus the HTML, then evaluate two layers:

1. **SOP check items** (`references/coverage-areas.md`) for that screen's area - assign **PASS / FAIL /
   N/A** per Desktop/Tablet/Mobile. Honor tag rules: ADMIN rows only under the admin role (N/A elsewhere),
   TIER rows once per tier tested, CUSTOM rows only if the spec enables that feature (N/A otherwise). Also
   run the **Cross-Cutting checks** (final sweep, visual consistency, responsive) on every screen.
2. **ISTQB review dimensions** (`references/review-dimensions.md`, §3.1–3.10) for open-ended defects, UX
   friction, accessibility, security observations, and product gaps that the fixed checklist doesn't cover.

**Nitpick the UI on every screen - don't wait for something to look broken to flag it.** Actively check every
one of these, not just whatever happens to catch your eye:
- **Fonts** - sizes too small to read comfortably, inconsistent type scale across screens, mismatched font
  weights/families for similar text roles.
- **Padding** - cramped/insufficient padding inside cards, buttons, inputs, modals; excessive/wasteful
  padding creating dead space; padding that differs between visually-similar components.
- **Margins & whitespace** - uneven gaps around sections, awkward dead space, inconsistent breathing room
  between the same kind of element on different screens.
- **Spacing & distances between elements** - inconsistent gaps in lists/grids/nav items, elements sitting
  too close together or too far apart relative to their visual grouping.
- **Alignment & symmetry** - text/icons/buttons not aligned to a consistent grid or baseline, off-center
  elements that should be centered, asymmetric layouts where the design implies symmetry.
- **Visual coordination** - mismatched colors, fonts, border-radius, icon styles, shadows, or hover/focus
  states across otherwise-similar UI elements.

The bar is "does this look deliberately designed, polished, and coordinated," not just "does it function" - file these as UI findings (per `references/finding-types.md`) even when nothing is technically broken.

**This UI nitpick sweep is FORCED and MANDATORY, and it is HIGH PRIORITY - not an optional nice-to-have.**
Specifically: after finishing the SOP criteria-sheet pass (Step 4) and the custom-page pass (Step 3), run a
**dedicated final nitpick pass over every single page/screen covered in this run** - every SOP area screen
and every custom page - checking fonts/padding/margins/spacing/alignment/visual-coordination on each one
specifically, even if you already glanced at it earlier. Do not skip this step, do not fold it silently into
earlier passes and call it done, and do not treat it as lower priority than functional testing - a page that
passes every functional check can still fail this pass, and both matter. Record every nitpick found, however
minor, as its own UI finding (per `references/finding-types.md`) - don't discard small ones as "not worth
noting."

For every issue found, classify it with `references/finding-types.md` - a SOP finding type (Bug / Note /
Question / Idea) **and** an ISTQB Category + Severity + Priority. Apply the mindset and golden rules in
`references/istqb-principles.md` throughout (don't assume a feature works because it exists; back every
finding with a concrete observation; keep exploring until no new functionality surfaces).

### 4b. Push verification coverage (95% of criteria-sheet cells is a HARD MINIMUM, exhaust every avenue first)
**If 3-worker parallel mode was chosen, each persona-worker does its own version of this step against its
own role/tier's rows inside its own isolated browser, per `references/parallel-mode.md` - the 95% target
still applies, but to the merged sheet after all workers report back, not to any single worker's partial
coverage.** What follows is the classic single-agent path.

A criteria sheet that's mostly UNVERIFIED isn't useful. **95% of criteria-sheet cells as non-UNVERIFIED
(PASS/FAIL/N/A) is a hard minimum, not a target to approach** - for a partial run this 95% is computed
**over in-scope cells only** (the areas/pages selected in Step 2b); out-of-scope areas have no cells at all
in this run's sheet, so they don't factor into the percentage either way. Regardless of scope, do not
present a run as finished, and do not move to Step 5 (Write outputs), until the live percentage clears 95%.
If genuine external-side-effect blockers
(real emails, features absent from the platform) would keep you under 95% even after exhausting every
technique below, say so explicitly to the user before finalizing rather than silently shipping a sub-95% sheet.
Once scratch/dev is confirmed (Step 1), don't stop at
"the button is present" - actively drive verification up using these techniques, always cleaning up after
yourself. Treat UNVERIFIED as a last resort, not a default: before marking any row UNVERIFIED, ask "is there
any safe, reversible way to actually exercise this?" - if yes, do it, even if it takes an extra click or two
to find the entry point.
- **Disposable test entities.** Create a throwaway record (user, channel, call/booking) via the app's own
  Add/Create UI, exercise the actions you need against it (Edit, Deactivate/Reactivate, Mark
  complete/incomplete, Export, etc.), confirm the result, then delete/revert it. One disposable user can
  usually cover Add/Edit/Deactivate/Reactivate/Export in a single create→test→delete cycle instead of one
  cycle per action.
- **"Nothing exists yet to test against" is a reason to create something, not a reason to mark N/A/UNVERIFIED.**
  If a row needs an existing recurring call/scheduled call/booking/entity and none currently exists, that's
  exactly the disposable-test-entity case above - create one through the app's own UI (a call that repeats
  weekly, a booking for tomorrow, etc.), exercise the specific row you're testing against it (the delete-prompt,
  the RSVP flow, whatever it is), observe the result, then delete/revert it. Only fall back to N/A/UNVERIFIED
  if creating that entity turns out to be genuinely unsafe (real side effects you can't undo) or you've tried
  and the creation flow itself is broken (which is itself a finding, not a reason to skip the row silently).
  Don't write "no X exists to test this" as a Notes explanation without first having tried to create one.
- **Member/tier perspective without a second set of real credentials.** If the admin UI has an impersonation
  action (e.g. "Log In As User"), try it - but verify it actually navigated/changed session state before
  trusting it; if it's a silent no-op, that's itself a bug (file it) and fall back to signing out and
  signing back in with the disposable test account's own credentials to get the member view. Use this to
  resolve ADMIN-tag restriction rows (confirm non-admin nav hides admin surfaces), TIER-tag rows (confirm
  locked/unlocked state and channel visibility for that tier), and any flow only reachable from the member
  side (e.g. ticket creation may not exist in the admin/staff view at all).
- **Avoid unnecessary real-world side effects.** Prefer "Set Password" over "Send Invite", and admin-set
  passwords over "Send Magic Link"/"Reset Password", whenever the UI offers a non-email path to the same
  verification - real invite/magic-link/reset emails should only be sent if the user explicitly confirms an
  address that's safe to email.
- **If a Gmail (or other mailbox) MCP connector is available for an inbox the tester actually owns, use it to
  verify email-based flows end-to-end instead of leaving them UNVERIFIED.** Trigger "Send Magic Link,"
  "Reset Password," "Send Invite," etc. against the tester's own admin account (it's usually listed as a row
  in User Management) or a `+alias` of an inbox they control, then search/read that mailbox via the MCP to
  confirm the email actually arrived, check its branding/content, and (for magic links) follow the link to
  confirm it logs in. This safely covers rows that would otherwise require spamming a real member's inbox.
  If the mailbox connector's token has expired, tell the user it needs reauthorization (via their connector
  settings, not a tool call) rather than guessing or skipping the row silently. **User Management's
  self-targeting actions (Send Magic Link/Reset Password/Log In As User) are commonly disabled for the
  admin's own currently-logged-in account, and every *other* listed user is usually a real external person you
  don't have permission to email.** Don't leave admin-triggered email rows UNVERIFIED just because of this - create a **disposable test user via Add User whose email is a `+alias` of the tester's own inbox** (e.g.
  `ahmed.a+mlptest@scalingeasy.com` routes to `ahmed.a@scalingeasy.com`), then trigger Reset
  Password/Send Magic Link/Send Invite against *that* new user (not your own logged-in account, not a real
  member). Check the mailbox MCP for delivery and branding, then delete the disposable user when done. That's
  a genuinely safe target the self-service login-screen path can't substitute for, since it exercises the
  admin-initiated flow specifically, not the self-service one. Separately, if the row under test is really
  about the self-service flow (e.g. "Magic link flow works end-to-end" from Login & Auth rather than
  User Management), the disabled self-targeting menu items on your own admin row are simply expected product
  behavior, not a bug - trigger that flow from the public login screen instead (e.g. "Email me a magic link")
  to test against your own inbox directly.
- **Watch for mojibake corrupting tokens/links inside emails read via a mailbox MCP.** Some mailbox connectors'
  HTML parsing can silently replace a byte (e.g. an `=` in a query-string token) with a replacement character
  when returning the email body - the visible text still *looks* mostly right, but the exact token value is
  wrong, so clicking/reconstructing the link 404s or reports "invalid/expired" even on a fresh, unused token.
  If a magic-link/reset-link click-through fails immediately after a fresh send, don't conclude the feature is
  broken - try once more, and if the same corruption pattern repeats, treat delivery + branding as verified
  (real evidence) but note the final click-through step as unverified due to a tooling limitation, distinct
  from a product defect.
- **A CDN/embed block (e.g. a video/iframe provider serving a "couldn't verify your connection" or similar
  bot-detection message) is not a slow-load issue - don't assume more waiting fixes it, but do verify that
  once**, with an explicit wait (2-5s) and ideally on a second, different item (different lesson/video), before
  concluding it's a persistent automated-browser block rather than a transient one. If the identical block
  message reappears after a real wait on more than one item, that's strong evidence it's CDN-level bot
  detection, not a product bug - record it as UNVERIFIED with that evidence and note it needs a real
  non-headless human session to confirm, rather than re-trying indefinitely.
- **Inherit verdicts across viewports when the difference is genuinely data, not layout.** A topic dropdown,
  a permission check, or a locked/unlocked badge is driven by the same data regardless of viewport - once
  confirmed on one viewport, it's reasonable to mark tablet/mobile the same with a one-line note ("data-driven,
  not viewport-dependent") rather than re-clicking through the identical flow three times. Reserve fresh
  per-viewport checks for things that are actually layout-sensitive (forms, panels, multi-column layouts) - and if you spot a real responsive bug (e.g. a panel that overlaps at exactly one breakpoint), file it.
- **Track your live percentage** (non-UNVERIFIED cells ÷ total cells across desktop/tablet/mobile) as you go.
  If you're still under ~70% after the initial crawl and one round of write-path testing, look at what's
  driving the gap - usually either missing tier/member coverage (fix with the technique above) or shallow
  tablet/mobile assessment (fix by extending confirmed desktop verdicts per the point above) - and do another
  pass rather than stopping early. Rows that are blocked by real external side effects (sending an email,
  something that doesn't exist on this platform) are legitimate to leave UNVERIFIED/N/A; don't fabricate a
  verdict to hit the number.
- **Explicitly test name changes and password changes from both directions**: (a) as the **admin**, editing
  another user's name/password via the User Management Edit/Set Password actions, and (b) as the **user
  themselves**, editing their own name via Settings > Profile and their own password via Settings > Security
  - confirm each edit actually saves and propagates everywhere that value is displayed (dashboard greeting,
  community messages, admin user list, etc.), not just that the form accepted the input.
- **Don't flag your own test data as a bug.** If a display name, email, or other value looks like leftover
  test data (e.g. a literal edited-looking name), that may simply be a value *you or the user* set while
  testing the edit flow in this same session - check whether editing it actually worked correctly (which is a
  pass, not a bug) before writing it up as a data-integrity defect. When in doubt, ask the user rather than
  assuming it's a leftover/seed-data problem.
- **A feature can be hidden behind in-page sub-navigation, not just top-level nav.** Don't conclude a checklist
  item is untestable just because it's not visible on first load of a screen - many features live behind a
  secondary tab/button *within* a page (e.g. a "Recordings" tab inside the Calendar/Calls screen, sub-tabs
  inside a Settings panel, a "Folder" drill-down). Click through every visible tab/sub-tab/folder on a screen
  before deciding a feature is absent or unreachable.
- **Disposable file uploads are safe and should be exercised, not skipped.** For any checklist item that's an
  upload control (profile picture, attachment, resource link, etc.), generate a small throwaway test file
  (e.g. a tiny solid-color PNG) and actually upload it, then confirm the change persisted after a reload - "the button exists" is not verification; "I uploaded something and it visibly took effect" is. Only skip
  this if there's no safe way to revert (e.g. no remove/reset control) - note that limitation rather than
  leaving the row UNVERIFIED without having tried.
- **Exhaust every view mode/entry point before marking a feature FAIL (missing) or leaving it UNVERIFIED.**
  If a feature might be gated behind a toggle (grid vs. list view, a "select mode," a kebab/overflow menu),
  check all of them before concluding it doesn't exist - e.g. a bulk-select checkbox might only appear in
  list view, not grid view. Only mark FAIL once you've actively looked in every place the control could
  reasonably live.

### 4c. Final audit - before declaring the run done
**This step always runs exactly once, in the main/orchestrator turn - never inside a persona-worker - regardless of whether Step 2c/4b ran classic single-agent or 3-worker parallel.** In parallel mode it runs
against the already-merged criteria sheet/findings (see `references/parallel-mode.md`'s merge phase), not
against any individual worker's partial output.

Run this checklist once, after coverage clears 95% and before writing/uploading outputs. Skipping it lets
stale or unverified claims slip into the delivered report - this has actually happened in past runs.
- **Re-reproduce every single confirmed bug one more time, not just the ones that feel shaky.** Once the
  findings list is "done," go back through it top to bottom and redo each repro live - same steps, fresh
  page state - before the report goes out. Don't reserve this only for findings you already suspect are
  weak; a bug that seemed rock-solid can turn out to be mis-scoped (e.g. actually platform-specific, or
  present on one tab but not another) or occasionally a flat-out false positive from a testing-tool artifact
  (see the DOM leaf-node and dead-click sections above) - and the only way to catch either is to run it
  again with fresh eyes, not to re-read your own earlier notes and nod along. On the SCALEDOS run
  (2026-07-23), this final pass caught a bug that was really only reproducible on one of two platform tabs
  (the original write-up implied both) - the fix was a one-line scope correction, but it would have shipped
  wrong without the recheck. Update severity/scope/screenshots based on what you see this time, not what you
  wrote the first time.
- **Screenshot existence audit.** For every row with a FAIL verdict, confirm its referenced screenshot
  filename actually exists on disk in `bug-screenshots/` - don't trust that writing the filename into
  `criteria-sheet.json`'s Notes/Screenshot field means the file was actually captured and copied. A
  filename can get referenced in a note (e.g. "see calendar_delete_recording_fail_desktop.png") without the
  screenshot ever having been taken, especially when a bug was found and described in prose before the
  screenshot step was reached. Script the check: collect every non-`-` screenshot filename from
  `criteria-sheet.json`, `ls bug-screenshots/`, and diff. Any referenced-but-missing file means either
  capture it now (ideally by re-reproducing the bug, which doubles as a fresh confirmation) or fix the
  reference.
- **Re-verify any FAIL that depended on a multi-step interaction you only clicked once.** A "click X, see Y
  happen" claim is more trustworthy than "click X, nothing happened" - but multi-step flows (click a
  button that reveals an inline Confirm/Cancel row, then needing a second click on Confirm) are easy to
  under-click and mistake the still-collapsed second control for "no prompt appeared at all." If a FAIL's
  own evidence trail doesn't show you clicking all the way through a multi-step action, that's a signal to
  redo it once headed before finalizing, not just carry the earlier note forward. This exact mistake
  produced a false "delete never shows a confirm" finding in a past run - the confirm row was there,
  collapsed, and simply not clicked.
- **A "confirmed headed" note written earlier in the same run is not automatically still true** - if you
  changed technique, retested a neighboring flow, or simply have doubt, spend one more repro cycle
  confirming it rather than trusting your own prior note at face value. Two profile-page findings (name-edit
  persistence, delete-recording confirmation) were both corrected from FAIL to PASS on a second look in the
  same session that had originally marked them FAIL "confirmed headed" - the first pass was simply wrong,
  not the platform.
- **Re-verify any FAIL whose only evidence is a DOM text-absence check** (e.g. "searched the page for note
  text X, found nothing") **before finalizing it - this class of check has its own false-positive failure
  mode, distinct from the dead-click one.** A `querySelectorAll` scan filtered to leaf nodes (`children.length
  === 0`) will silently return empty for real, visible text whenever the target phrase is split across a text
  node and a nested inline element (e.g. a note containing a bolded button-name mid-sentence) - no single leaf
  node then contains the full substring, regardless of what's actually on screen. See
  `references/playwright-crawl-procedure.md`'s "Before marking UI text/copy missing" section for the check to run.
  This produced a confirmed, retracted false-positive bug on the SCALEDOS run (2026-07-23) - the finding was
  only caught because the user asked for a post-hoc cross-check of screenshots against bug descriptions, which
  should not be the safety net. Any FAIL of the shape "X text/note never appears" needs a screenshot taken at
  the moment of the check, not just a DOM query, before it's written up.
- **Never bypass the app's own UI with a direct authenticated API call (fetch with an extracted auth
  token, a raw PATCH/DELETE against the backend) to manufacture a test condition, without asking the user
  first.** This reads as an unauthorized direct production-database write even in scratch/dev, and may be
  blocked by the environment's own safety classifier. If retriggering a one-time gate (e.g. a
  guidelines-acceptance modal, an onboarding-complete flag) seems to require this, stop and ask - don't
  just do it because it's technically reachable via the same anon key + bearer token the app itself uses.
  If the user does approve it, revert the field to its original value afterward and say so explicitly in
  the finding's notes.
- **A one-time "has the user seen this" gate (e.g. a mandatory guidelines/ToS modal) may not be controlled
  by a single DB timestamp field alone** - resetting the obvious field and reloading (or even a full
  logout/login) may not bring the modal back if there's additional client-side state or a different
  server-side check involved. If you need to retest this class of feature, the only reliable way is a
  genuinely fresh account that has never triggered it, not resetting a flag on an existing account.

### 5. Write outputs
**This step always runs exactly once, in the main/orchestrator turn - never inside a persona-worker.** In
3-worker parallel mode, the files below are written from the single merged criteria sheet/findings
produced by `references/parallel-mode.md`'s merge phase, not assembled separately per worker.

**For a partial run, all three outputs below contain only the in-scope SOP areas/custom pages selected in
Step 2b - do not add placeholder rows, N/A stubs, or stub sections for excluded areas.** The
`release-readiness-report.md` Executive Summary must state the run's scope up front - e.g. "Partial QA run - scope: Community/Channels, Custom: AI Coach page. The remaining SOP areas were not assessed in this run." - replacing any implied "whole platform" framing. `findings.json` likewise only contains findings from
in-scope screens.

Write all three to `qa-runs/<slug>/`:
- **`release-readiness-report.md`** - per-screen sections + final assessment + scorecard. Use the exact
  structure in `references/report-template.md`.
- **`criteria-sheet.md`** + **`criteria-sheet.json`** - every SOP check item with a Desktop/Tablet/Mobile
  verdict. Never leave a device cell blank. The table has **six columns**:
  `Check Item | Desktop | Tablet | Mobile | Notes | Screenshot` - the Tags column is dropped in favour of
  the Screenshot column. **Only put a screenshot in a row that has a FAIL verdict on at least one
  viewport** - PASS/N/A/UNVERIFIED rows get `-`; a sheet where every row carries a screenshot is noise.
  **Bug screenshots live in their own `qa-runs/<slug>/bug-screenshots/` folder, physically separate from the
  general `qa-runs/<slug>/screenshots/` crawl folder** - copy (don't move) each bug row's evidence file into
  `bug-screenshots/` the moment you mark that row FAIL, and reference it in `criteria-sheet.json`'s
  `screenshot` field as a **bare filename** (e.g. `"admin_community_mobile.png"`), never a `screenshots/...`
  path - the bare filename is what tells you (and the Doc-export placeholder) to look in `bug-screenshots/`,
  not the general folder. Do this as you go, not as a batch cleanup at the end: it's much easier to catch a
  wrong-viewport or wrong-control screenshot (see the sanity-check rule below) right when you copy it than to
  reconstruct it later. `screenshots/` stays the full, unfiltered crawl archive (every viewport, every role) - `bug-screenshots/` is the curated subset a reviewer actually needs to look at.
  **Every `bug-screenshots/` image must have a red circle/box drawn around the exact spot the bug is
  visible** - a reviewer shouldn't have to hunt across a full-page screenshot to find what's wrong. This is a
  mandatory finishing step for every visual (non-network-level) FAIL, done the same moment you copy the file
  into `bug-screenshots/`. Default recipe (`--caps=devtools`; same technique also documented in
  `references/playwright-quick-mode.md`):
  1. `browser_snapshot()` to get the buggy element's ref (e.g. `e71`).
  2. `browser_highlight({target: "e71", element: "<short description>", style: "outline: 4px solid red;
     outline-offset: 2px;"})` - draws the overlay directly on the live element at real device pixels
     (confirmed live: the custom `style` param renders exactly as given - a thick red box precisely around
     the target, no manual `getBoundingClientRect()`/`devicePixelRatio` scaling needed at all).
  3. `browser_take_screenshot({filename: "..."})`, copy into `bug-screenshots/` per the folder discipline
     above.
  4. `browser_hide_highlight({target: "e71", element: "<same description>"})` before the next capture, so
     overlays don't bleed across screenshots.
  **Two cases still need a manual `browser_evaluate`-based fallback instead of `browser_highlight`:**
  - **CSS `text-transform`'d text with no matching accessibility node** (e.g. a heading rendered in all-caps
    via a stylesheet rule, where the real DOM text is mixed-case) - `browser_snapshot` won't give you a clean
    `ref` for the raw text node. Walk text nodes directly via
    `document.createTreeWalker(document.body, NodeFilter.SHOW_TEXT)` and use
    `document.createRange().selectNodeContents(node).getBoundingClientRect()`, then draw the box with a
    minimal image library at that rect (still need HiDPI scaling here, since this path bypasses
    `browser_highlight`'s automatic real-pixel targeting).
  - **A fleeting/timing-based bug** (e.g. a flash-of-unstyled-content that self-corrects in 1-3s) - by the
    time you snapshot+highlight, the glitch is gone. Grab the *container's* stable rect from the resolved
    state (any time, via `browser_evaluate`), then apply that same box to the screenshot that happened
    to catch the bug mid-glitch.
  **Video evidence for multi-step or intermittent functional bugs** (a click-sequence, a form flow, anything
  a single screenshot can't show): use `browser_start_video({filename: "qa-runs/<slug>/bug-screenshots/<id>.webm", size: {width, height}})`
  before reproducing the bug, `browser_video_chapter({title, description, duration})` to mark named
  checkpoints (e.g. "Actual dropdown options" right before the moment that matters - the card appears as a
  blurred-backdrop title screen in the recording, useful for narrating what the viewer is about to see),
  `browser_video_show_actions()`/`browser_video_hide_actions()` to toggle an on-screen action-callout overlay
  while driving the repro, and `browser_stop_video()` to finalize (confirmed live: writes an actual `.webm`
  file at the given project-relative path, same file-write sandboxing as screenshots). **Use this especially
  for a suspected-Critical bug before writing it up as confirmed** - a repro that only fails once and can't be
  reproduced on a same-session retest should not ship as a Critical finding on a single click-retry error
  alone; a full video of a clean, repeated repro (or the retry attempt showing it does *not* reproduce) is
  much stronger evidence either way than a text description of a diagnostic log line. Confirmed live: a
  "Critical, root-caused via Playwright's click-retry diagnostic" finding was retracted in the same run after
  it failed to reproduce twice on a careful manual retest immediately after - the diagnostic log alone had
  been treated as sufficient evidence prematurely. Video (or at minimum a second/third live repro attempt with
  network-level confirmation) is the corrective habit going forward for any single-observation bug claim,
  Critical severity especially.
  **Notes column: only write something for a bug row (a FAIL verdict) or a row worth flagging as an idea - leave every plain PASS/N/A/UNVERIFIED row's Notes as `-`.** Don't restate verification steps or reasoning
  for passing rows in the delivered sheet; that detail belongs in your own working process, not the output.
  A bug row gets a tight one-line description of what's broken/missing (optionally pointing at a finding ID).
  Bug screenshots go **inline in the Screenshot cell** of the bug row itself (not in a separate section
  below the table). For local `criteria-sheet.md`/PDF use `<img src="<abs path>" width="480" style="max-width:100%; height:auto;">`; for a Google Doc export use a text placeholder instead (see below).
  Append an **"Out-of-Criteria Bugs"** section after all criteria tables: any Bug/UI finding from
  `findings.json` that does **not** map to any SOP checklist row still needs to be recorded here (id,
  severity, one-line finding, impact, recommendation, and its screenshot placeholder) - don't let a real
  defect go missing just because the fixed checklist has no row for it. Full layout in
  `references/report-template.md`.
  **This section is exclusively for findings that have no corresponding criteria-sheet row at all - never
  duplicate a finding that's already represented in a row's Notes/verdict.** Before adding any finding here,
  check whether it's the reason behind an existing FAIL/N/A verdict somewhere in the criteria tables above
  (e.g. a finding about a broken password-reset flow belongs folded into the "Password reset works" row's
  Notes, not repeated again at the bottom). Concretely: build the set of finding IDs already referenced by
  a criteria row (by ID mention or by being the clear basis for that row's verdict), then only the findings
  **not** in that set go in this section. A finding appearing in both places is a bug in the report itself - the reader ends up seeing the same defect twice and can't tell if they're different issues.
  **Every entry in this section MUST carry a screenshot placeholder - no exceptions, even for findings
  that feel "meta" or process-related (a methodology note, a reachability summary) rather than a specific
  visual bug.** If the finding doesn't have an exact matching capture already, take (or reuse) the closest
  representative screenshot of the screen/state the finding describes rather than omitting the field - a screenshot of the right page with a one-line caption pointing at what to look for is far better than no
  image at all. Copy it into `bug-screenshots/` and reference the bare filename, same as any criteria-row
  bug screenshot. Before finalizing this section, grep it for any entry missing a screenshot line and fix
  it - this is a common gap when a finding gets added late (e.g. corrections/retests appended after the
  main pass) and the screenshot step gets skipped in the rush to wrap up.
  **Console/network-error findings need special handling - don't just reuse an unrelated page screenshot
  as a lazy placeholder.** A finding about a backend 400/500 or a malformed API call has no visible on-page
  symptom, so a screenshot of "the page it happened on" (e.g. the dashboard) shows nothing related to the
  bug and is misleading evidence - a reviewer looking at it can't tell what they're supposed to see. Instead:
  1. Reproduce the error and pull the exact text via `browser_console_messages`/`browser_network_requests`
     (status code, endpoint, response body) - never paraphrase it, quote the real string.
  2. Render that real text as a small standalone HTML page styled like a browser DevTools console/network
     panel (dark background, monospace font, red error rows, the actual URL/status) - a `data:text/html`
     URL works well since `file://` is blocked in this environment.
  3. Navigate to it and screenshot it like any other page, then copy into `bug-screenshots/` same as normal.
  This gives a real, honest visual artifact backed by the actual captured error text, instead of either an
  unrelated screenshot or a misleadingly "real-looking" fabricated devtools image with invented text.
- **`findings.json`** - flat array of
  `{id, area, screen, role, device, finding_type, category, severity, priority, title, steps, expected, actual, impact, recommendation}`.
  `id` (e.g. `BLOCKER-01`, `NOTE-04`) is required - `references/report-template.md`'s Out-of-Criteria Bugs
  section headers and any in-row Notes reference ("see BLOCKER-01") depend on every finding having one.

### Exporting to Google Docs - mandatory, automatic part of finishing every run, full or partial
**This is not an optional step the user has to ask for.** The moment `release-readiness-report.md`,
`criteria-sheet.md`/`criteria-sheet.json`, and `findings.json` are written, immediately continue into this
section in the same turn and produce the Google Doc - full runs and partial runs alike. The Google Doc *is*
the deliverable a human actually reads; local markdown/JSON files alone are not a finished run. Check
`claude mcp list` for `google-workspace ... ✔ Connected` without waiting to be told to. If it's connected,
create the doc (or update the existing one, if this is a later pass in the same run/session - see "True
in-place editing" below) before presenting results in Step 6. Only skip this and go straight to Step 6 if
`google-workspace` genuinely isn't connected or fails to connect - and if so, **say that explicitly** in
Step 6's summary ("Google Doc export skipped: `google-workspace` MCP not connected - only local files were
produced") rather than silently ending the run with just local files and no explanation.

**If the `google-workspace` MCP server is connected (`claude mcp list` shows
`google-workspace ... ✔ Connected`), use it as the default/primary tool for every Google Doc upload and
every subsequent edit to that same doc.** This server (`workspace-mcp`, run via `uvx`) replaced the earlier
`google-documents` server (`@domdomegg/google-documents-mcp`) in this environment - do not reference or try
to reconnect the old one, it was deliberately removed. `google-workspace` is a stdio server (no separate
HTTP process to babysit, unlike the old one) and exposes real Docs-native table/heading/image primitives
that the old server's `insertTable`-cell-by-cell approach couldn't do reliably at scale.

**Real tool names** (`ToolSearch` for exact schemas the first time you use this server in a session - they're
deferred like any other MCP tool):
- `create_doc` - create a new blank doc.
- `get_doc_content` - read back the document's text, to verify state before/after an edit.
- `modify_doc_text` - insert/replace/format text, with tab/segment targeting and link management.
- `find_and_replace_doc` - find-and-replace across the whole doc.
- `insert_doc_elements` - add tables, lists, page breaks.
- `update_paragraph_style` - **real heading styles and paragraph formatting** (the old server could only
  fake headings with `##`-prefixed plain text - this one can actually apply Heading 1/2/3, spacing, etc.).
  Use this for every report section header instead of a literal `##` marker.
- `insert_doc_image` - insert images from Drive or a URL (local `file://` paths are NOT supported per the
  underlying Docs API - see "Screenshot embedding" below).
- `create_table_with_data` - **create a real native table directly from row/column data in one call**, not
  a manual `insertTable` + per-cell `insertText` sequence. This is the preferred way to build the criteria
  sheet (114+ rows) - pass the full row data at once rather than hand-computing cell indices.
- `debug_table_structure` - inspect/troubleshoot a table if `create_table_with_data` output looks wrong.
- `batch_update_doc` - atomic multi-step operations (named ranges, section breaks, etc.) for anything the
  higher-level tools above don't cover.
- `inspect_doc_structure` - analyze the document's current structure and safe insertion points before
  editing - use this instead of guessing index math.
- `manage_doc_tab` - **can populate a doc/tab directly from markdown - but confirmed NOT to convert markdown
  pipe tables into real Docs tables (2026-08-17, recipes-website run).** A 21K-character `populate_from_markdown`
  call containing 16 pipe tables (a Findings table, a Scores table, and 14 criteria-sheet tables) reported
  success and correctly built every heading/paragraph, but `inspect_doc_structure` afterward showed `"tables":
  0` - every pipe table had been inserted as flat, literal `| cell | cell |` text instead. **Use this tool
  only for prose sections (Executive Summary, narrative paragraphs, headings) - never for anything with a
  table in it.** Build every table with `create_table_with_data` instead (see below), regardless of how small
  or "obviously simple" it looks - don't gamble on markdown conversion working this time.
- `update_doc_headers_footers`, `list_document_comments`, `manage_document_comment`, `export_doc_to_pdf` - available if a run needs them; not part of the standard qa-review export flow.

**True in-place editing is the main win**, same as before. A QA run is rarely one-and-done - corrections,
retests, and late findings come in after the doc already exists. Keep editing the *same* document (by its
ID) via `modify_doc_text`/`find_and_replace_doc`/`batch_update_doc` instead of creating a brand-new file/URL
every time something changes. Only call `create_doc` once per genuinely new run.

**Screenshot embedding - confirmed working recipe (2026-08-15, learnsudo quick-mode run).** `insert_doc_image`
only accepts a Drive file ID or a public `http(s)://` URL - never a local path. Getting a `bug-screenshots/`
PNG actually inline (not a text placeholder) takes exactly these steps, in order, every time:

1. **Copy the local file into `~/.workspace-mcp/attachments/` first.** `create_drive_file`'s `fileUrl:
   file://...` rejects any path outside that directory with "path is outside permitted directories" - this
   isn't optional or configurable per-call, the file must physically live there (a plain `cp` via Bash is
   enough). Skip straight to this step rather than trying the original `bug-screenshots/` path and hitting
   the error first.
2. **Upload with `create_drive_file`**, `fileUrl: "file:///Users/.../.workspace-mcp/attachments/<name>.png"`,
   `mime_type: "image/png"`. Returns a Drive file ID.
3. **Make the Drive file link-shareable with `set_drive_file_permissions`, `link_sharing: "reader"`, before
   ever calling `insert_doc_image`.** Skipping this step is the single most common cause of failure: Docs
   returns `"There was a problem retrieving the image. The provided image should be publicly accessible..."`
   even though the file ID is completely valid and owned by the same account - the Docs API fetches the
   image over HTTP under the hood and needs it to be publicly readable to do so. (These are QA screenshots of
   a test/scratch environment, not third-party or end-user personal data, so this is a normal, low-risk step
   for this workflow - not one to second-guess per image.)
4. **Always pass both `width` and `height` to `insert_doc_image`, never `width` alone.** The tool defaults
   `height` to `0`, and Docs rejects that outright: `"Invalid object size: height must be greater than 0 if
   specified."` A full-page desktop screenshot is roughly 16:9 - `width: 400, height: 225` reads well inline;
   halve both (`300x169`) for two images meant to sit side by side in the same finding.

If step 3 is ever refused (a future safety classifier decision, or the user explicitly doesn't want the
image made link-shareable even transiently), don't fight it - ask the user how they want to handle
screenshots (text placeholder, a private Drive link instead of an inline image, or proceeding with sharing)
rather than retrying the same blocked call. If the whole upload path is skipped or fails for any other
reason, fall back to the text-placeholder convention: `[📷 INSERT SCREENSHOT: <filename> - from bug-screenshots/]`, and say so explicitly in your final report rather than silently shipping placeholders.

**`create_table_with_data` - confirmed unreliable in this environment (2026-08-15), despite the tool's own
description recommending it as the preferred one-call approach.** Two separate calls in the same run (an
8×4 and a 16×3 table) both reported `"ERROR: Could not find table after creation"` - but in both cases the
table *shell* (correct row/column count) had actually been created with every cell left empty, not a total
failure. Don't retry the same call expecting it to work, and don't conclude the table wasn't created either - check first:
1. Call `debug_table_structure` (pass the right `table_index` if there's more than one table already in the
   doc) - it reports each cell's exact `insertion_index` and `current_content` (`'\n'` = empty, confirms the
   shell-only failure mode above).
2. Populate every cell with `batch_update_doc`'s `insert_text` operations, one op per cell, each targeting
   that cell's `insertion_index` directly - **in reverse order, last row/last column back to the first
   (0,0)**. This is the critical part: inserting into an earlier cell shifts every index after it, so filling
   forward corrupts every subsequent target; filling backward leaves every not-yet-touched index untouched
   until its own turn. A single `batch_update_doc` call with all cells' `insert_text` ops (already in reverse
   order) populates the whole table in one round-trip.

**Building a report with many tables (a criteria sheet with 10+ areas): the marker-based pattern, confirmed
reliable at 14-tables-in-one-doc scale (2026-08-17, recipes-website run).** Doing `inspect_doc_structure`
after every single table/image insert to find the next one's index is correct but slow at this scale. Instead:
1. **Write the entire report's prose in one pass** via `batch_update_doc`'s `insert_text(end_of_segment: true)`
   - every heading, paragraph, and list item, in final reading order - with a short, distinctive placeholder
   line standing in for each table and image, e.g. `⟦TBL:findings⟧`, `⟦TBL:login⟧`, `⟦IMG:bug02⟧` (any
   marker text that won't collide with real report content works - unique per table/image).
2. **One `inspect_doc_structure(detailed: true)` call** locates every marker's exact `start_index`/`end_index`
   in a single round-trip, instead of one lookup per table.
3. **Build every table via `create_table_with_data` + the reverse-fill recipe above, processing markers from
   the bottom of the document to the top** (highest `start_index` first). Insert each table at its marker's
   `start_index` - since nothing below an unprocessed (lower-index) marker has been touched yet, every
   remaining marker's original index recorded in step 2 stays valid throughout the whole pass. Do the same
   bottom-to-top ordering for `insert_doc_image` calls against the `⟦IMG:...⟧` markers (remember the confirmed
   `+1`-per-image index shift when locating that marker's now-shifted position to delete next).
4. **Clean up every leftover marker paragraph in one final batch**, again bottom-to-top within that batch
   (Google's API applies a batch's operations in list order, so listing `delete_text` ops from highest index
   to lowest keeps every remaining one valid as you go) - one `batch_update_doc` call with 10+ `delete_text`
   ops is fine.
5. **Apply every heading style in one last batch** of `update_paragraph_style` calls - these don't change
   document length, so order doesn't matter and a single detailed structure scan from step 2 already has every
   heading's indices (re-scan once more first if steps 3-4 shifted things you need exact positions for).
This turns an O(n) sequence of insert-then-rescan round-trips into roughly 4 batched phases regardless of how
many tables the report has.

**Manual index tracking for building a doc via repeated `batch_update_doc`/`end_of_segment` appends** (needed
whenever `manage_doc_tab`'s markdown auto-conversion isn't used or isn't trusted for a section): each
`batch_update_doc` response reports `"Document length: N"` after the call. The next append's start index is
always `N - 1` (content occupies `[1, N-1]`; `N` itself is the doc's own trailing terminator, not a usable
insertion point) - so precompute every subsequent heading/paragraph's start/end index in one local pass
(e.g. a quick Python one-liner summing string lengths from that starting position) instead of calling
`inspect_doc_structure` after every single insert. Inserting an inline image via `insert_doc_image` adds
exactly `+1` to the next `N` (confirmed repeatedly this run) - account for that one extra index before the
next text append that follows an image. Still call `inspect_doc_structure` at natural checkpoints (after a
table, after a run of several appends) to catch any drift before it compounds across many operations.

**The `N - 1` rule above only holds for appends at the true end of the document.** The moment you're
inserting into the *middle* of an existing doc (e.g. rebuilding one section - Findings - while leaving
Recommendation etc. below it untouched), `"Document length: N"` reflects the **whole document**, including
all the untouched content after your edit point - using it to back-derive "how long was the text I just
inserted" will be wrong and compounds fast. For mid-document inserts, always compute the next index by
direct arithmetic on your own input string's length (`your_index + len(your_text)`), never by reading it back
out of the reported total document length.

**Inserting new text at the exact index where an existing styled paragraph (e.g. a `HEADING_1`) begins can
make the entire inserted block inherit that heading style**, not just the line you intended as a heading - confirmed directly (2026-08-15): a multi-paragraph insert landed at an old "Findings" heading's start index,
and every paragraph in the new block rendered as `HEADING_1` until corrected. After any insert at a
pre-existing paragraph's start, reset the whole newly-inserted range to `NORMAL_TEXT` first, then reapply
`HEADING_1`/`HEADING_2` surgically only to the lines meant to be headings, using indices computed from your
own input strings (see above) - don't assume only the text you explicitly restyle will carry the new
formatting.

**A one-character index miscalculation during a multi-step mid-document rebuild shows up as a single stray
orphaned character in its own paragraph, with a nearby heading missing that same character from its start**
(e.g. "UI Nitpick Summary" split into a stray "U" paragraph earlier in the doc plus a headingless "I Nitpick
Summary" later) - this is a symptom of one insert landing 1 index early/late relative to an existing
paragraph boundary, not a sign of deeper corruption. Always do a full `get_doc_as_markdown` or
`get_doc_content` read-through after a multi-step rebuild before considering it done, specifically checking
that every heading that should have a leading `#`/`##` still does. `find_and_replace_doc` (no index math
needed) is the safest way to fix a confirmed stray-character defect once located - but match on **single**
`\n` between paragraphs, not a blank-line/double-`\n` gap: `get_doc_content` and `get_doc_as_markdown` render
paragraph breaks differently (the latter inserts visual blank lines the actual document doesn't contain), and
a `find_text` built assuming double-`\n` separators will silently match 0 occurrences.

**Never bulk-delete a large index range in a doc you didn't build in this same turn without first checking
what's actually in that range.** `delete_text`/a wide `batch_update_doc` delete operates on character
*positions*, and an inline image occupies a position exactly like a character does - a range-delete aimed at
"stale placeholder text" will just as happily delete a real image the user manually inserted into that same
range, with no separate confirmation or warning. This actually happened on the SCALEDOS run (2026-07-23): a
"clean rebuild" delete meant to clear out drifted placeholder text also silently wiped screenshots the user
had pasted in by hand, and the mistake wasn't caught until the user pointed it out. Before any delete wider
than a single paragraph on a doc that may have been touched outside your own tool calls, call
`inspect_doc_structure` and actually look for non-text elements in that range (inline objects/images), not
just text previews - and when in doubt, ask before deleting rather than after. Google Docs' built-in Version
History (File → Version history) is the recovery path if this happens anyway - point the user to it
immediately rather than trying to reconstruct lost content yourself.

**Write every findable piece of doc prose like a person telling a colleague what they found, not a form
being filled in.** This applies to bug write-ups, criteria-sheet notes, and the report narrative alike - avoid the "Steps: 1)... Expected:... Actual:..." template voice as the *only* voice; a reader should be able
to tell what happened and why it matters without translating clinical shorthand. Keep the facts precise (the
exact repro steps, the exact observed behavior, real severity/priority) but write the connective tissue - why it matters, what a real user would experience, what's surprising about it - the way you'd actually
explain it out loud. Precision and a human voice aren't in tension; do both.

**Never use an em dash (`—`) anywhere in any output this skill produces** - not in `findings.json`, not in
`criteria-sheet.md`/`release-readiness-report.md`/`quick-scan-report.md`, not in the Google Doc export, not
in a chat reply summarizing a run. Use a plain hyphen with spaces (` - `) for the same parenthetical/break
purpose instead, or restructure the sentence with a comma, colon, or period. This applies regardless of how
naturally an em dash would fit the narrative-voice rule above - narrate the finding richly, just without that
specific character. If a large body of text is generated in one pass, do a final check for `—` before
treating the output as done (e.g. `grep -c "—" <file>` for a local file, or a `find_and_replace_doc` pass
after a Google Doc export) rather than relying on having typed it correctly throughout.

**Setup gotchas actually hit when installing this server (Apple Silicon Mac, `uvx` install path) - check
these first if `claude mcp list` shows `google-workspace ... ✘ Failed to connect`:**
- `workspace-mcp` depends on `cryptography`, which needs a compiled wheel. If no prebuilt wheel is available
  for the resolved Python version/arch, `uv` tries to build it from source via Rust/maturin, which fails on
  an outdated Cargo (`lock file version 4` requires a Cargo newer than ~1.78) - fix with `rustup update`.
- Even with a current Rust toolchain, if `uvx` resolves to an x86_64 (Rosetta/Intel Homebrew) Python instead
  of a native arm64 one, the build fails with `error[E0463]: can't find crate for core` for the
  `x86_64-apple-darwin` target. Fix: `uv python install cpython-3.12.13-macos-aarch64-none` (or current
  equivalent), then pin the server's launch command explicitly:
  `uvx --python cpython-3.12.13-macos-aarch64-none workspace-mcp --tool-tier core`. Don't rely on bare
  `uvx workspace-mcp` picking the right interpreter on a machine with multiple Pythons installed.
- **Never pass `GOOGLE_OAUTH_CLIENT_SECRET` as a raw value on the `claude mcp add` command line or as a
  literal `--env` value** - this persists the plaintext secret into shell history and the MCP config file,
  and the harness's own safety classifier will (correctly) block it. Instead, write a standard Google OAuth
  `client_secret.json` (the `{"installed": {"client_id": ..., "client_secret": ..., ...}}` format) to a file
  with restrictive permissions (`chmod 600`), and pass only `GOOGLE_CLIENT_SECRET_PATH=<path-to-that-file>`
  as the env var - this is a path, not a secret, so it's safe to store in the MCP config.
- The Client ID/Secret themselves come from Google Cloud Console → APIs & Services → Credentials → the
  OAuth 2.0 Client for this app. If the user only has the Client ID (Console redisplays this) but not the
  Secret (Console only shows this once, at creation or reset time), they need to click "Reset Secret" there
  to get a fresh one - there's no way to recover an old plaintext secret from Console after the fact.
- **Redirect URI mismatch is a likely first-auth failure** - the `client_secret.json`'s `redirect_uris` entry
  must exactly match an Authorized Redirect URI already configured on that OAuth client in Google Cloud
  Console (commonly `http://localhost:<port>/oauth2callback`, where `<port>` matches `WORKSPACE_MCP_PORT`,
  default 8000). If first login fails with `redirect_uri_mismatch`, this is the first thing to check.
- A freshly-added MCP server's tools won't appear via `ToolSearch` until the Claude Code session restarts - same rule as any other MCP server. `claude mcp list` showing `✔ Connected` is necessary but not sufficient.

### 6. Present
Summarize for the user: the run's **scope** (Full QA, or the specific partial list from Step 2b), Executive
Summary, pass/fail counts from the criteria sheet, the Overall QA Score and Overall Release-Readiness Score
(/100), Overall Risk Level, and the Final Recommendation - one of **Ready for Production / Ready with Minor
Fixes / Ready After Major Fixes / Not Ready for Release**. For a partial run, state plainly that the
scorecard and recommendation apply only to the in-scope areas, not the whole platform. **Include the Google
Doc's link** (created automatically in Step 5) as part of this summary - or, if it was skipped because
`google-workspace` wasn't connected, say so explicitly here rather than letting the doc's absence go
unmentioned.

## Rules
- Never run write or destructive tests against shared/production data - scratch/dev only, or read-only +
  UNVERIFIED.
- **Once scratch/dev is confirmed (fresh or via a persisted `platform-specs/<slug>.md` flag), treat the
  session as having full read/write access to everything on the platform for the rest of the run - admin
  panels, every role's account settings, every write action (Add/Edit/Delete/Export/Send/Deactivate),
  disposable test users, disposable test entities of every kind.** Don't re-hesitate or re-gate individual
  actions "just in case" once that confirmation is in place - the only ongoing constraints are the mechanical
  ones already covered elsewhere: clean up every disposable entity you create, don't email a real user you
  don't have permission to email (use a disposable test user + a `+alias` of an inbox you own instead, per
  Step 4b), and redact secrets before including them in the report. Don't default anything to UNVERIFIED
  because it's a write action - that's exactly the coverage scratch/dev access exists to unlock.
- Redact secrets, tokens, and passwords from screenshots and logs before including them in the report.
- Do not assume a feature works simply because it exists - verify it fulfils its purpose.
- Test every flow **end-to-end**, not just the first step, and on all three device sizes.
- Explore every role/tier until no new functionality can be found.
- Mark **N/A** for features intentionally absent from this platform, or for a checklist item whose entry
  point you made a genuine effort to locate (checked the obvious screens, tabs, and menus it would live in)
  but couldn't find at all. A broken-or-missing-but-expected feature whose entry point you **did** find and
  directly confirmed is absent/broken is **FAIL**, never N/A - the distinction is whether you located the
  actual control and it's missing/broken (FAIL) versus you searched reasonably and never found where the
  control would even be (N/A).
- **Don't leave a row UNVERIFIED indefinitely once you're reasonably (~50%+) confident the feature simply
  isn't reachable in this build.** After a genuine search effort per Step 4b, if you still can't locate the
  entry point for a checklist item, convert it from UNVERIFIED to **N/A** and write **exactly `Not found.`**
  in the Notes column (per the Notes-column rule above, this counts as worth flagging even though it's not a
  confirmed bug) - this is the standard phrasing for every N/A-because-not-located row, not a paraphrase or a
  longer explanation; don't leave it silently UNVERIFIED with no explanation either. Reserve genuine
  UNVERIFIED for rows where you know exactly where the control is and what it does, but are intentionally not
  triggering it (e.g. it would send a real email to an address you don't control).
- Never leave a Desktop/Tablet/Mobile cell blank in the criteria sheet - if genuinely unverifiable, record
  why.
- **95% of criteria-sheet cells as non-UNVERIFIED (PASS/FAIL/N/A) is a hard minimum**, not a target - do not conclude the run below this line once scratch/dev write-path and member-perspective testing
  (Step 4b) are available. Cells blocked by genuine external side effects (real emails, features absent
  from the platform) are legitimate exceptions - don't force a verdict just to hit the number, but do
  exhaust the safe techniques in Step 4b before leaving something UNVERIFIED, and keep testing until you
  clear 95% or have a concrete, explicit reason (stated to the user) why a specific set of cells can't go
  higher.
- If a spec file exists for the platform (`platform-specs/<slug>.md`), read it before starting - it
  determines which tiers and CUSTOM rows apply.
- **A partial-QA run's Overall Release-Readiness Score and Final Recommendation apply only to the in-scope
  areas** (Step 2b) - state this caveat explicitly in the report; never imply full-platform readiness from a
  partial run.
