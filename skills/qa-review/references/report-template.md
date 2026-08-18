# Report Templates

## Per-Screen Report (repeat for every screen, in `release-readiness-report.md`)
```
### Screen Name
Purpose - what it is intended to accomplish.
Functional Findings - working correctly, missing functionality, workflow issues, validation issues,
  edge cases, recommendations.
UI Findings - strengths & weaknesses.
UX Findings - strengths & weaknesses.
Accessibility Findings.
Risk Assessment and Test Coverage Notes - what was and was not tested, and why.

Findings Table: | Severity | Category | Finding | Impact | Recommendation |

Scores (/100): Functional Quality · UI Quality · UX Quality · Accessibility · Overall Screen Score
Risk Level: Low / Medium / High / Critical
```

## Criteria Sheet (`criteria-sheet.md` + `criteria-sheet.json`)
**For a partial-QA run (`SKILL.md` Step 2b), this sheet contains only the SOP areas/custom pages selected
in scope - no rows, stubs, or placeholders for excluded areas.** Everything below in this section otherwise
applies unchanged, just over the in-scope subset.

Mirrors the master Google Sheet layout from the ScalingEasy SOP - one row per check item from
`references/coverage-areas.md`, grouped by coverage area:

```
| Check Item | Desktop | Tablet | Mobile | Notes | Screenshot |
|---|---|---|---|---|---|
| <item text>| PASS/FAIL/N/A/UNVERIFIED | PASS/FAIL/N/A/UNVERIFIED | PASS/FAIL/N/A/UNVERIFIED | <one-line note> | <image or placeholder or -> |
```
Rules: never leave a device cell blank; TIER rows repeat once per tier tested; ADMIN rows are N/A under
non-admin roles; CUSTOM rows are N/A unless the platform spec enables that feature.

**The Tags column is dropped; the sixth column is Screenshot.**

**Notes column: STRICT, MANDATORY rule - FAIL, N/A, and UNVERIFIED rows all carry a Note, and every single
one of those rows MUST have one; only PASS rows are stripped to `-`.** Concretely:
- **FAIL row → Notes is mandatory**, never `-`. Every bug row needs a real note (see the human-description
  requirement below) - there is no such thing as a FAIL row with an empty Notes cell.
- **N/A row → Notes is mandatory**, never `-`. If the feature is genuinely absent by design, say so briefly.
  If the control simply couldn't be located after a genuine search (per the Rules section), Notes must read
  **exactly** `Not found.` - the standard, fixed phrasing for that case, not a paraphrase.
- **UNVERIFIED row → Notes is mandatory**, never `-`. Say briefly what blocked verification (e.g. "needs a
  second real account to trigger from - not set up this pass" or "requires creating a disposable user per
  tier - not done this pass") so the reader knows why it's unresolved, not just that it is.
- **PASS row → Notes is always `-`.** Do not explain why something passed, note it was "data-driven," or
  restate the verification method - that reasoning lives in your own working notes/`findings.json`, not the
  delivered sheet. Before finalizing any Doc/sheet export, explicitly strip Notes from every PASS row that
  has one - this is a required last-pass check, not optional cleanup.

**Every bug-row Note is STRICT, MANDATORY, HIGH PRIORITY on one requirement: it must read as a human
describing what they personally did and what they personally saw - self-contained, not a bare pointer to a
bug ID elsewhere.** A note that only says "Blocked - see BUG-09." or "Same as above - see BUG-02." fails
this bar even though it's short and plain-sounding - it makes the reader go hunt down BUG-09 to find out
what actually happened. Every bug-row note must itself contain, in one or two natural sentences:
1. **What you did** - the specific action, in plain terms ("Clicked Create Call", "Opened the Users tab",
   "Typed a reply and hit Send Reply").
2. **What you saw** - the actual, concrete result ("nothing happened - the form never opened", "the page
   stayed on Overview", "the text just sat there in the box, never got added to the thread").
A bug-ID cross-reference (e.g. " - see BUG-09") can still be *appended* at the end for traceability, but it
is never a substitute for the description - think of it as a footnote, not the content. Good: "Clicked
'Create Call' and waited - no form or modal ever opened, the calendar just sat there unchanged (BUG-09)."
Bad: "Blocked - see BUG-09." When several rows share one root cause, don't collapse them all to "Same as
above" - restate briefly, in your own words, what *this specific* control did when you tried it, even if
the underlying cause is identical to a row above.

**Write every note - in the Screenshot-column notes, the Out-of-Criteria Bugs write-ups, everywhere in the
delivered Doc/sheet - the way a person would describe it to a colleague, not like a machine-generated log
line.** Avoid the "CONFIRMED (2026-07-14, via X tool): ..." boilerplate pattern - drop timestamps, tool
names, and method-tags from the reader-facing text (keep that detail in `findings.json` if it's useful for
your own traceability). Prefer plain, direct phrasing: "There's no way to bulk-delete recordings - checked
both grid and list view" reads naturally; "CONFIRMED (2026-07-14): Checked both Grid view and List view
inside a Recordings folder - no checkbox/select-all/bulk-action affordance exists in either mode" reads like
a bot. Say what's actually wrong the way you'd explain it out loud, in one or two sentences.

**Screenshot column: STRICT, MANDATORY rule - ONLY bug rows (FAIL on at least one viewport) get a
screenshot, and every single bug row MUST have one; there is no such thing as a FAIL row with `-` in this
column.** Never attach a screenshot to a PASS/N/A/UNVERIFIED row. If a FAIL row doesn't have evidence
captured yet, go back and capture it (navigate to the screen, reproduce the exact failing action, screenshot
the failing state, copy it into `bug-screenshots/`) before finalizing the sheet - do not ship a bug row
without one. For bug rows, embed the screenshot **inline** in the Screenshot cell - the table does not have
a separate "Bug Screenshots" section below it:
- Local `criteria-sheet.md` / PDF: use `<img src="<abs path>" width="480" style="max-width:100%; height:auto;">`
- External Doc (Google Docs): use a named placeholder - `[INSERT SCREENSHOT: <filename> - from bug-screenshots/]` - so the user can drag/paste the file in manually (see Doc export guidance below).

If generating a PDF via pandoc/weasyprint, set the Screenshot column's `td` max-width to ≥480px in the
CSS so images are not squeezed by a too-narrow column.

**Before finalizing, sanity-check that every bug row's screenshot filename actually shows that bug** - open the referenced file (or its `findings.json` entry) and confirm it matches the Notes text, rather than
trusting a filename carried over from an earlier pass. A mismatched screenshot (e.g. a row about mobile
title-wrapping pointing at an unrelated tier-verification screenshot) is worse than no screenshot - it
actively misleads whoever reviews the sheet. Concretely:
- **The screenshot's device must match the failing device(s) in the verdict.** If a row is PASS/PASS/FAIL
  (mobile-only failure), the screenshot must be the mobile capture - not a desktop or tablet shot of the
  same screen, even if that's the only screenshot you have on hand. If you don't have the right-device
  screenshot, go capture it rather than substituting a same-page-wrong-viewport one.
- **The screenshot must show the specific control/state the Notes describe**, not just the general screen
  it lives on. "No read-only toggle in the Edit Channel modal" needs the Edit Channel modal open on screen - a screenshot of the channel feed behind it doesn't prove the toggle is missing. If describing a dropdown's
  contents, the screenshot must show that dropdown expanded with its actual options visible.
- **When a row's verdict changes** (e.g. a topic-list finding gets upgraded from an inferred FAIL to a
  directly-confirmed one via a different flow), **re-check whether the original screenshot still applies** - it was often captured for the old evidence path and may show a different data source (e.g. an admin
  filter dropdown with different options than the member-facing form the updated Notes now describe).
- Do this check as an explicit last pass over the whole sheet before shipping it, not just at the moment
  each row is first written - screenshots and notes drift out of sync as findings get refined mid-session.

### Out-of-Criteria Bugs (section, after Bug Screenshots)
Any Bug/UI finding from `findings.json` that has **no** matching SOP checklist row still gets recorded here - don't drop a real defect just because the fixed checklist has no row for it:
```
## Out-of-Criteria Bugs

### <finding id> - <title> (Severity/Priority)
<one-line finding> - <impact> - <recommendation>
<img src="<abs path>" width="480" style="max-width:100%; height:auto;">
```
Skip this figure only if the finding genuinely has no screenshot. Purely informational Notes (not bugs) can
be listed as a short bulleted "Notes / Observations" list at the end without images.

### Exporting to an external Doc (no inline images) - EXACT FORMAT, MANDATORY, HIGH PRIORITY
When producing a Google Doc export of the criteria sheet, match this structure and formatting **exactly** - it is not optional, this is the standard house format for every future run:

**Title** (Google Doc H1): `# **<Platform> Criteria Sheet - QA Bugs & Screenshots (<date>)**`

**Intro line** (plain paragraph, right after the title):
`Bug rows (red) include a screenshot placeholder in the Screenshot column. All referenced files are in the`
`local bug-screenshots/ folder - drag/paste each named file into the Doc at its placeholder.`

**One real markdown table per coverage area** (this becomes an actual Google Docs table on upload, not a
plain-text list - do not flatten it to bullet/line format): each area is its own `## **Area Name**` heading
(H2, bold), followed immediately by a GFM table shaped like this (note the blank header row and the `:-:`
center-alignment row - copy this exactly, it's what the reference format uses):
```
|  |  |  |  |  |  |
| :-: | :-: | :-: | :-: | :-: | :-: |
| Check Item | Desktop | Tablet | Mobile | Notes | Screenshot |
| <item text> | PASS/FAIL/N/A/UNVERIFIED | ... | ... | <note or \-> | <filename or \-> |
```
- **Screenshot column:** for any row with a FAIL verdict on at least one viewport, put the **bare
  screenshot filename** (e.g. `admin_community_mobile.png`) as plain text in that cell - not a bracketed
  `[INSERT SCREENSHOT: ...]` placeholder, just the filename itself, since the user already has the files
  and only needs the name to know which one to drag in. Every PASS/N/A/UNVERIFIED row's Screenshot cell is
  `\-`.
- Copy every screenshot referenced by a bug row or an Out-of-Criteria Bug into
  `qa-runs/<slug>/bug-screenshots/` (original filenames kept, so the cell text matches exactly what the
  user needs to find).

**Out-of-Criteria Bugs section**, after all area tables:
```
## **Out-of-Criteria Bugs**

Real defects found during this pass that don't correspond to any SOP checklist row above.

### **<finding-id> - <title> (<Severity>/<Priority>)**

**Finding:** <one-paragraph description of what's broken>

**Impact:** <one-paragraph effect on user/business>

**Recommendation:** <one-paragraph suggested fix>
```
Repeat the `### **id - title (severity/priority)**` + Finding/Impact/Recommendation block for every
out-of-criteria bug. End the Doc with an `### **Ideas / Future Enhancements**` bullet list (short items,
one line each, e.g. `- **IDEA-08** - <suggestion>`) if there are any.

**Title the Doc** descriptively - platform name + "Criteria Sheet" + date - so it's identifiable at a
glance, matching the H1 above.

Upload with `content_mime_type: "text/markdown"` (not plain text) so the pipe-table syntax converts into a
real Google Docs table rather than rendering as literal text.

`criteria-sheet.json` mirrors the criteria rows as:
```json
{"area": "Login & Auth", "check_item": "...", "desktop": "PASS|FAIL|N/A|UNVERIFIED", "tablet": "...", "mobile": "...",
 "notes": "...", "screenshot": "qa-runs/<slug>/screenshots/... (only if a FAIL verdict exists, else empty)",
 "tags": ["ADMIN"|"TIER"|"CUSTOM", ...]}
```

## Final Assessment (end of `release-readiness-report.md`)
**For a partial-QA run, the Executive Summary and every score below must state the run's scope explicitly**
(the specific areas/pages selected in `SKILL.md` Step 2b) - the scorecard, Overall Release-Readiness Score,
and Final Recommendation describe readiness of the in-scope areas only, never the whole platform.

Produce, in order:
1. **Executive Summary** - concise overview a stakeholder can read in 30 seconds.
2. **Application Overview** - what it is, roles, built vs. unbuilt surfaces.
3. **Overall Quality Assessment.**
4. **Test Coverage Summary** - what was reviewed and what could not be assessed (and why).
5. **Critical Defects · High-Risk Areas · Missing Functionality · Business-Logic Concerns.**
6. **Improvements** - Workflow · Validation · UI · UX · Accessibility.
7. **Security Observations · Regression Risks · Release Risks.**
8. **Recommendations Before Production.**

### Overall Quality Scorecard
| Category | Score (/100) |
|---|---|
| Functional Completeness | |
| Workflow Quality | |
| Validation Quality | |
| UI Quality | |
| UX Quality | |
| Accessibility | |
| Security Observations | |
| Design Consistency | |
| Testability | |
| Overall Product Readiness | |

Then provide:
- **Overall QA Score (/100)**
- **Overall Release-Readiness Score (/100)**
- **Overall Risk Level** - Low / Medium / High / Critical
- **Final Recommendation** - one of: Ready for Production / Ready with Minor Fixes / Ready After Major
  Fixes / Not Ready for Release
- **Prioritised Findings** - Must Fix Before Release / Should Fix Before Next Sprint / Can Be Deferred
- **Future Enhancements**
