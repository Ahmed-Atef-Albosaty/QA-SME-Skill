# qa-review

An ISTQB-based release-readiness + product/UX review skill for web apps. Drives a real, visible browser
session via a Selenium MCP server to discover every screen on a platform, then assesses it against a
six-hats framework (PM, UX, UI, QA, Accessibility, End User) plus a configurable SOP coverage-area checklist
(ADMIN/TIER/CUSTOM tag rules, Desktop/Tablet/Mobile verdicts), producing a Release-Readiness Report and a
filled criteria sheet.

Also includes a lighter **quick mode** (`/qa-review quick <page_url> <email> <password>`) that scans one page
and its linked subsidiary pages without the full platform-wide discovery pass or SOP scorecard.

## What this does

- Logs into a target web app with credentials you provide, in a real (headed) browser you can watch.
- Discovers every screen/route on the platform (nav links, spec-provided endpoint lists, sitemap, bundle
  inspection, and optionally a backend schema query).
- Captures every in-scope screen at three fixed viewports: Desktop (1920×1080), Tablet (1024×1366), and
  Mobile (500×800 - the narrowest viewport this Selenium server can produce; see Known Limitations).
- Exercises every write-capable control for real (Add/Edit/Delete, forms, toggles) using disposable test
  entities, not just visual observation.
- Classifies every issue found (Bug/Note/Question/Idea + ISTQB category/severity/priority).
- Writes a `release-readiness-report.md`, `criteria-sheet.md`/`.json`, and `findings.json` to
  `qa-runs/<slug>/`, plus screenshots and red-circle-annotated bug screenshots.
- Optionally exports the whole report to a Google Doc with real native tables, if a Google Docs MCP server is
  connected.

## Prerequisites

- **Node.js + npm** (for `npx`, used to run the Selenium MCP server).
- **Google Chrome** installed on the machine that will drive the browser session.
- **Python + [uv](https://github.com/astral-sh/uv)** (for `uvx`), only if you want the optional Google Docs
  export step. Skip this if you're fine with local markdown/JSON output only.
- A **Google Workspace account with the Google Docs/Drive MCP server set up and OAuth-authorized**, again
  only if you want the Doc export.

## Install

```bash
/plugin marketplace add <your-org>/qa-review-plugins
/plugin install qa-review@<your-org>/qa-review-plugins
```

(Replace `<your-org>/qa-review-plugins` with wherever you've hosted this repo.) Installing the plugin also
registers the `selenium` and `google-workspace` MCP servers automatically - you don't need to run
`claude mcp add` yourself. **You do need to restart your Claude Code session once after installing**, since a
freshly-added MCP server's tools only appear in a new session (this applies every time either server's config
changes, not just the first install).

## Usage

```
/qa-review                          # classic full-platform review, asks for inputs
/qa-review <slug> <base_url>        # classic, with slug/URL pre-filled
/qa-review quick <page_url> <email> <password>   # quick mode - one page + what it links to
```

Full mode will ask for: platform name/slug, base URL, one login per role you want tested (at minimum an
admin and a standard member), whether a platform spec file exists, and whether the target is a scratch/dev
environment safe to write-test against.

## Known limitations (confirmed via live testing, not theoretical)

- **True mobile-width viewports (390-428px, real phone widths) are not achievable.** This Selenium server's
  browser-launch option has a hard floor around 500px wide, in both headed and headless mode - so "Mobile" is
  fixed at 500×800 throughout this skill rather than a real device preset. If you need pixel-accurate mobile
  testing, this isn't the right tool for that one pass.
- **Drag-and-drop reordering is known-fragile** with browser-automation tools in general (a documented,
  cross-project pattern, not specific to this skill) - expect to verify it manually or with extra care.
- **Real side-effect flows** (sending actual emails, push-notification browser-permission grants) are
  deliberately left to manual/careful testing rather than automated by default - the skill's own safe-testing
  rules avoid triggering these without explicit confirmation.
- A stale cached ChromeDriver version can occasionally block browser launches with a
  `session not created: This version of ChromeDriver only supports Chrome version <old>` error. Fix:
  `rm -rf ~/.cache/selenium/chromedriver` and retry - this forces a fresh, correctly-matched download.

## Files

- `skills/qa-review/SKILL.md` - the main procedure.
- `skills/qa-review/references/` - detailed sub-procedures: the Selenium tool-by-tool crawl procedure,
  3-worker parallel-mode design, quick-mode procedure, SOP coverage-area checklists, ISTQB review dimensions,
  finding classification, and the exact report/criteria-sheet template.

## License

MIT
