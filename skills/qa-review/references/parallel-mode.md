# Parallel mode - 3-worker viewport split

Read this file when `SKILL.md` Step 1b's run-mode question is answered "3-worker parallel" and the user
has given the required `Workflow`-tool opt-in for this turn. This replaces Step 2c and 4b of the classic
path; Steps 1, 1b, 2a (discovery), 2b (scope decision), 3 (coverage-gap check), 4c (final audit), 5 (write
outputs), and 6 (present) are unchanged and still run once, in the main/orchestrator turn - never inside a
worker. **Steps 2a and 2b in particular must complete before any worker is spawned** - the scope decision
determines the in-scope screen list every worker below is handed; no worker ever receives the full
discovery inventory, only the confirmed in-scope subset.

## Default worker assignment: one worker per viewport, each bound to its own persistent account
Each of the 3 workers is bound to one **device preset** (Desktop `"Desktop Chrome"` @1920×1080 / Tablet
`"iPad Pro 11"` / Mobile `"iPhone 15"` @393×852 - real Playwright device presets, not a workaround; see
`references/coverage-areas.md`) **and** one **persistent account** for its entire lifetime - it logs in once
(or boots pre-authenticated from a `--storage-state` file, see "Phase 0" below) and never switches roles,
which also eliminates the login-collision gotcha (`playwright-crawl-procedure.md`'s Per-role login point 7)
the same way the original persona-split design did, just combined with the viewport split rather than
instead of it. This maps directly onto the criteria sheet's own shape (Desktop/Tablet/Mobile columns are
exactly the 3 workers' outputs) while *also* getting simultaneous multi-tier and multi-account coverage for
free:

- **Worker 1 = Desktop (`"Desktop Chrome"` @1920×1080) + the real admin account.** Full crawl of every screen
  at desktop viewport, using the actual admin login. **Also owns all write-path testing**
  (Add/Edit/Delete/Export, admin-triggered email flows, etc.) and is responsible for creating the other two
  workers' disposable accounts before the fan-out - see "Phase 0" below.
- **Worker 2 = Tablet (`"iPad Pro 11"`) + disposable account #1**, created by Worker 1 in Phase 0 as an
  `+alias` of the tester's own inbox (e.g. `ahmed.a+qatablet@scalingeasy.com`) and assigned to one tier the
  spec calls out as high-priority/previously-untested. Full crawl at tablet viewport, from this account's own
  logged-in perspective - this is real TIER-row coverage from an actual member session, not admin-simulated.
- **Worker 3 = Mobile (`"iPhone 15"`, 393×852) + disposable account #2**, a second `+alias`
  (e.g. `ahmed.a+qamobile@scalingeasy.com`) assigned to a *different* tier than Worker 2's. Same shape as
  Worker 2, at mobile viewport.

Pick the two tiers for Workers 2/3 based on the platform spec (prioritize tiers a prior run flagged as
untested or high business weight) and say the full assignment to the user before starting, e.g.: "Worker 1:
Desktop + admin · Worker 2: Tablet + `govtech`-tier disposable account · Worker 3: Mobile +
`it_bootcamp`-tier disposable account."

## Write-path consolidation (avoids tripling disposable-entity churn)
Write actions (create/edit/delete a user, channel, booking, ticket, etc.) are not viewport-dependent - the
same button click does the same server-side thing regardless of screen size. Assigning full write-path
testing to all 3 workers would triple disposable-entity creation/cleanup for no verification benefit, and
risks 3 workers racing to create/delete similarly-named test entities against the same backend. Instead:
- **Worker 1 (Desktop) is the sole owner of write-path testing and disposable-entity lifecycle** (create,
  exercise, delete). Its confirmed PASS/FAIL verdict for a write-path row is what the merge phase uses for
  that row's Desktop cell.
- **Workers 2/3 (Tablet/Mobile) inherit the write-path row's verdict for their own viewport cell** *only when
  the failure/success mode is genuinely not layout-sensitive* (e.g. "Delete confirmation prompt appears and
  works" - same modal, same behavior, any viewport) - per the existing "inherit verdicts across viewports"
  rule in `SKILL.md` Step 4b, applied here across workers instead of within one. Note the inheritance in that
  row's Notes ("data-driven, verified on Desktop by Worker 1").
- If a write-path control is itself laid out differently per viewport (e.g. a modal that's genuinely broken
  or unreachable only on mobile - a real responsive bug), Workers 2/3 must flag that independently rather
  than blindly inheriting Worker 1's Desktop verdict - layout-sensitive divergence is exactly the case the
  inheritance rule exists to exclude.

## Prerequisite
Confirm the 3 named Playwright MCP servers (`playwright1`/`playwright2`/`playwright3`, each its own
`@playwright/mcp` process) exist and are connected - see `playwright-crawl-procedure.md`'s "Parallel setup"
section for the exact `claude mcp list` check and `claude mcp add` commands (including the `--device`,
`--isolated`, and `--storage-state` flags each worker needs) if missing. Do not proceed into the phases below
until all 3 show `✔ Connected`.

## Phase 0 - account setup (sequential, run once, before the parallel fan-out)
**The user supplies working credentials for all 3 workers up front** (Worker 1's real admin login, plus two
member accounts for Workers 2/3, both as `+alias`es of one inbox the user controls, e.g.
`ahmed.a+qatablet@scalingeasy.com` / `ahmed.a+qamobile@scalingeasy.com` - never a real third-party address,
so the Gmail MCP can verify any email-based flow across all of them from that single inbox). This is the
**default** account-setup path - prefer it over creating accounts via the admin Add-User flow, since asking
the admin panel to create these every run has been the single biggest token cost in Phase 0 on past runs (the
Users-tab round trip plus its large Users-table page-source dumps). If the user genuinely has no other
credentials to give you, creating the 2 disposable accounts via Admin > Users is an acceptable fallback -
just say so explicitly and do the two creations back-to-back rather than assuming it away.

If the user hasn't supplied pre-existing credentials yet when a parallel run is requested, ask for them first
(3 sets: admin + 2 member accounts on different tiers, per the platform spec's priority - untested tiers, or
tiers with real business weight); only fall back to admin-created disposable accounts if none exist.

**Mint a storage-state file per persona before the fan-out, rather than live-logging-in inside each
worker.** For each of the 3 accounts: log in live once (per `playwright-crawl-procedure.md`'s "Per-role
login"), save the authenticated session via `browser_storage_state({filename:
"qa-runs/<slug>/.pw-state/<persona>.json"})` - **confirmed live: this path must be inside the project's
working directory, not a home-relative path like `~/.pw-state/...` or `/tmp/...`, or the write is rejected as
outside the server's allowed roots** - then have that persona's worker call
`browser_set_storage_state({filename: "<that file>"})` as its first action (or relaunch the server with
`--isolated --storage-state=<that file>` if you've separately confirmed that CLI flag works on the server
version in use) instead of handing it raw credentials to log in with itself. This sidesteps two real risks a
live login carries on every run: a flaky/500ing auth endpoint blocking a whole worker's entire run, and the
shared-profile login-collision gotcha. Storage-state expires (auth tokens have a lifetime) - if a worker's
first `browser_snapshot()` after booting still shows the login form, the state expired; fall back to a live
login for that one worker and re-mint its file rather than treating storage-state as a hard dependency.

Before the parallel fan-out, do a quick sanity check per account (not a full crawl) - confirm the
storage-state boots pre-authenticated (or the live login succeeds, if minting failed) - a bad credential
still blocks that worker's entire run, so it's better caught now than after 2 workers spend their whole run
failing to log in. Then pass all 3 accounts' email/password/tier/storage-state-path down into the Phase 1
`agent()` prompts.

## Phase 1 - parallel crawl + (Worker 1 only) write-path testing
**Scope the `Workflow` invocation to Phase 1 only - the parallel crawl - and have the script `return` the
3 workers' raw results once `parallel()` resolves.** Do not fold Phase 2 (merge) into the same Workflow
script as another phase/agent() call. The main-turn orchestrator (this conversation) takes the Workflow's
returned results and runs Phase 2 itself, as a normal foreground `Agent` tool call - visible in the chat,
not hidden inside the background workflow. This is a deliberate visibility choice, not a cost/architecture
one: merging is exactly the step where a wrong silent call (a cross-worker conflict resolved the wrong way,
an output file collision, a skipped Google Doc export) is most worth being able to watch happen and step in
on, rather than only finding out from a final notification.

Use the `Workflow` tool's `parallel()` (all 3 viewport workers start together - there's no cross-worker
dependency, so a barrier here costs nothing extra) with 3 `agent()` calls, one per viewport. Each agent
prompt must:
1. State its assigned device preset (Desktop `"Desktop Chrome"` @1920×1080 / Tablet `"iPad Pro 11"` / Mobile
   `"iPhone 15"` @393×852) and its assigned Playwright MCP server (`mcp__playwright1__*` and so on) -
   **explicitly forbid it from calling any other server's tools**. The device preset (plus `--isolated` and
   its `--storage-state` path from Phase 0) is baked into that server's launch command at `claude mcp add`
   time, not chosen per-call - see `playwright-crawl-procedure.md`'s "Parallel setup" section for the exact
   commands. Playwright MCP still has no mid-session device-resize call, so the viewport is fixed for the
   whole session once the server process starts, exactly as before - there's no stray-viewport-left-over
   false-dead-click cause here either.
2. Carry the full context a classic-mode agent would need: the platform base URL, the in-scope screen list
   confirmed in Step 2b (never the full discovery inventory from 2a), the endpoint list (or how
   to discover it, per `playwright-crawl-procedure.md`'s Endpoint discovery), the *full* `references/coverage-areas.md`
   (every SOP area, every ADMIN/TIER/CUSTOM row) filtered down to what this worker's one persistent account
   can actually reach (Worker 1 gets ADMIN rows; Workers 2/3 get their own tier's TIER rows plus every
   non-restricted row - no worker logs into a second account mid-run), its single set of credentials (no
   role-switching, no login-collision risk), and the dead-click/multi-step-verification/new-tab/UI-nitpick
   rules from `SKILL.md` Steps 2 and 4 in full - a worker does not get a lighter QA bar than a classic-mode
   single agent.
3. **Only Worker 1 (Desktop)** gets write-path testing instructions for **admin/shared-backend actions**
   (Step 4b's Add/Edit/Delete/Export on OTHER users' records, admin panels, Community/Calendar moderation
   controls, etc.) and owns disposable-entity cleanup for anything it creates there - explicitly forbid
   Workers 2/3 from touching those, to avoid 3 workers racing to mutate the same shared backend data.
   **Workers 2/3 (Tablet/Mobile) are NOT read-only beyond that narrow admin/shared-data carve-out.** Every
   SOP criteria-sheet row and every custom feature reachable from their own member account must be exercised
   **fully end-to-end** - clicked through, submitted, confirmed via network request + reload persistence,
   exactly as Step 4b requires for any check item - not just screenshotted or described. This explicitly
   includes self-service/account-scoped actions such as: editing their own name/password on
   Settings/Profile; posting, reacting to, and replying in Community channels; RSVPing to or joining a
   Calendar event/call; creating and replying to their own Support ticket; marking their own course
   lesson/quiz progress; and every custom feature (applying to a job on the Job Board, running a Recruiter
   Vault search/outreach action, generating a resume, completing an AI Interview Prep exercise, submitting a
   Help Desk Simulation step, etc.). None of these touch another account's data or a shared admin surface, so
   they carry zero cross-worker race risk. A locked/gated state is a legitimate PASS only when confirmed
   genuinely locked (e.g. a real Career-Journey milestone gate) - an unlocked feature must actually be
   clicked through before its row can be marked PASS.
4. Instruct it to write its screenshots/HTML to **viewport-prefixed paths** so 3 concurrently-running
   workers never collide on a filename: `qa-runs/<slug>/screenshots/<role>_<screen-slug>_<viewport>.png`
   (`<viewport>` = `desktop`/`tablet`/`mobile`, `<role>` = whichever persona was logged in for that specific
   screenshot - same convention the classic path already uses, just guaranteed distinct per worker since
   each worker owns exactly one `<viewport>` value).
5. Instruct it to return a **schema-validated structured result** (use the `schema` option on `agent()`),
   not free text, shaped as:
   ```json
   {
     "viewport": "desktop | tablet | mobile",
     "criteria_rows": [
       {"area": "...", "check_item": "...", "verdict": "PASS|FAIL|N/A|UNVERIFIED", "role": "<persona used>",
        "notes": "...", "screenshot": "<bare filename or '-'>", "tags": "ADMIN|TIER|CUSTOM|''"}
     ],
     "findings": [
       {"id": "...", "area": "...", "screen": "...", "role": "<persona>", "device": "<viewport>",
        "finding_type": "...", "category": "...", "severity": "...", "priority": "...", "title": "...",
        "steps": "...", "expected": "...", "actual": "...", "impact": "...", "recommendation": "..."}
     ],
     "screenshots_written": ["<role>_<screen>_<viewport>.png", "..."],
     "cleanup_confirmed": true
   }
   ```
   Each `criteria_rows` entry is a single verdict for *this worker's one viewport* - the merge phase below
   assembles 3 workers' single-viewport verdicts into each row's Desktop/Tablet/Mobile triple.
6. Instruct **Worker 1 only** to **delete every disposable entity it created** before returning - including
   the two Phase 0 accounts themselves - and to set `cleanup_confirmed: true` only once that's actually done.
   Workers 2/3 set `cleanup_confirmed: true` trivially (they created no entities of their own; their account
   is Worker 1's to delete).

### Cross-account real-time interaction testing (why 3 simultaneous live sessions matter)
Because all 3 workers stay logged in as 3 *different real accounts* for the entire run, this is the first
mode able to genuinely exercise interactions that require two live sessions at once - e.g. an in-app
@mention notification, a live chat reply, a "user joined" presence indicator. A serial single-agent run can
never test this (it only ever has one active session), and it's exactly the kind of item that ends up
"unresolved, needs a fresh angle" in a classic-mode run's spec notes.
- Instruct **Worker 1 (admin)** to, partway through its own crawl, post a real community message/channel
  post that **@mentions Worker 2's and Worker 3's disposable display names by name** (Phase 0 already knows
  these - pass them into Worker 1's prompt), and, separately, send a real chat/DM to each if the platform
  supports one.
- Instruct **Workers 2 and 3** to, later in their own crawl (not immediately - leave a real gap so Worker 1's
  post has time to land), check their own notification bell/inbox/unread-count for the mention and confirm
  its content matches what Worker 1 sent. Prefer `browser_wait_for({text: "<notification marker>"})` over a
  fixed-time wait here - wait on text the mention itself would produce (a badge count, a notification
  string) - or space this check naturally later in the worker's own crawl sequence rather than reaching for
  an arbitrary `{time: N}` delay.
- **Agents in `parallel()` are not synchronized with each other** - there's no message-passing between
  concurrently running `agent()` calls, so this is inherently best-effort timing, not a guaranteed rendezvous.
  If a worker's check comes back empty, don't immediately conclude the feature is broken: note it as
  UNVERIFIED with "cross-worker timing - not confirmed this run" rather than FAIL, unless a second attempt
  (worker re-checks once more before finishing) also comes back empty, in which case treat it as a real
  candidate finding per the usual dead-click-verification bar in `SKILL.md` Step 2c.
- This same pattern covers any other platform feature that's normally hard to test serially: simultaneous
  channel presence ("N members online"), typing indicators, real-time comment/reaction counts updating on
  one viewer's screen while another account posts.

## Phase 2 - merge (always sequential, always the orchestrator, never a worker, never inside the Workflow script)
**Run this as a foreground `Agent` tool call in the main conversation, after the `Workflow` call from
Phase 1 has already returned** - never as a further `agent()`/`phase()` step inside that same Workflow
script. Pass the 3 workers' returned JSON results into this agent's prompt directly (the Workflow result
already has them; no need to re-fetch anything). After all 3 crawl results are in hand:
- **Criteria rows:** for each `(area, check_item)` pair, assemble the Desktop cell from Worker 1's verdict,
  the Tablet cell from Worker 2's, the Mobile cell from Worker 3's. If a Tablet/Mobile worker left a
  write-path row UNVERIFIED (correctly, since it was forbidden from performing the write) and the row is
  confirmed *not layout-sensitive* on Worker 1's Desktop pass, inherit Worker 1's verdict into that cell per
  the "Write-path consolidation" rule above, noting the inheritance in that row's Notes. **Do not silently
  drop a genuine cross-worker disagreement** - if Worker 2 and Worker 3 (or a worker vs. an inherited
  Desktop verdict) reported conflicting FAIL-vs-PASS for what should be the same underlying behavior, note
  the conflict explicitly in that row's Notes rather than picking one silently, since a real viewport-specific
  regression looks exactly like this. Any cell no worker touched (and no inheritance rule applies to) stays
  UNVERIFIED, same as it would in classic mode.
- **Findings:** concatenate all 3 workers' `findings` arrays; de-duplicate by `id` collision (workers should
  not have picked the same finding ID, but if a near-identical finding surfaces at two viewports - e.g. the
  same console error seen on both desktop and mobile - merge them into one entry noting both
  devices/screens rather than keeping duplicates).
- **Screenshots:** every file each worker reported in `screenshots_written` should already be sitting in
  `qa-runs/<slug>/screenshots/` (the workers wrote there directly, viewport-distinct filenames prevent
  collisions) - no copy step needed here; the usual bug-screenshot copy into `bug-screenshots/` still
  happens per `SKILL.md` Step 5's rules, driven off the now-merged criteria sheet/findings.
- Write the merged result to the same in-memory/working structures Step 3 (mapping) and Step 4c (final
  audit) expect, then proceed into Step 4c exactly as classic mode would, against this merged data.

## Phase 3 - Step 4c, Step 5, Step 6 (unchanged, and also always foreground)
Run exactly as documented in `SKILL.md` - final audit, write outputs, present summary - once, against the
merged criteria sheet/findings. **Same visibility rule as Phase 2: do this directly in the main
conversation (your own Write/Edit/Bash calls, or a foreground `Agent` call you can watch), never as
another `agent()` step tucked inside a background `Workflow` script.** Writing the actual
criteria-sheet/findings/report files and the Google Doc export is exactly the kind of step where you want
to see what got written where - not learn about it only from a final notification. Note in the report's
methodology section that this run used 3-worker parallel mode split by viewport (not persona) and which
worker covered which column, so a reader can tell this apart from a classic run.

## Cost/speed reference (carry forward, don't re-derive)
Estimated from first-principles reasoning about duplicated setup, lost cross-worker cache reuse, and the
new merge pass - not measured telemetry. Confirmed acceptable trade by the user who approved this design:
**~20-35% more total tokens, ~2-3x faster wall-clock, at 3 workers.** Overhead scales up somewhat with
more workers (each pays its own setup tax); if a future run considers going beyond 3 workers, re-derive
this estimate rather than assuming it holds unchanged.
