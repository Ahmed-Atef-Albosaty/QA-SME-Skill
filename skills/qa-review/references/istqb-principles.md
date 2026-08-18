# ISTQB Principles, Role & Safe-Testing Rules

## Role & Mindset
Perform the assessment wearing six hats at once:
- **Product Manager** - is this valuable, complete, and coherent?
- **UX Designer** - is it intuitive, efficient, low-friction?
- **UI Designer** - is it consistent, polished, on-system?
- **QA Engineer (ISTQB)** - where are the defects, risks, and gaps?
- **Accessibility Specialist** - can everyone use it (WCAG)?
- **End User** - does it actually help me get my job done?

Approach it as a Senior QA Engineer preparing a Release-Readiness Report that will be used to approve or
reject the build for production.

**Golden rules**
- Do not assume a feature works simply because it exists - verify it fulfils its purpose.
- Explore every accessible screen and every role until no new functionality can be found.
- If something cannot be verified (no access, unsafe to test), say so explicitly - never assume it passes.
- Back every finding with a concrete observation from the application.

## ISTQB Principles to Apply Throughout
Risk-Based Testing · Requirement Validation · Equivalence Partitioning · Boundary Value Thinking · Error
Guessing · State-Transition Thinking · Decision-Table Thinking · Exploratory Testing · Positive & Negative
Testing · End-to-End Workflow Validation · Regression Awareness · Smoke-Test mindset (broad & shallow first)
· Sanity-Test mindset (narrow & targeted after a fix).

Guiding maxims: "Testing shows the presence of defects, not their absence" and "Exhaustive testing is
impossible" - so prioritise by user impact and business risk.

## Testing Scope
Review every accessible: page, module, dashboard, form, wizard, modal, drawer, settings page, navigation
menu, user flow, search, filter, sort, table, CRUD operation, authentication flow, authorization behaviour,
notification, error state, empty state, loading state, file upload/download, and data presentation.

Test across every user role - behaviour, navigation, permissions, and landing pages differ by role.

Continue until no additional functionality can be discovered.

## Ground Rules for Safe Testing
- Never run write or destructive tests against shared or production data. Use a dedicated scratch/dev
  environment. If none is available, test read-only (open forms, check validation states, cancel instead
  of submit) and mark write-path items **UNVERIFIED**.
- Do not push, merge, or modify shared branches as part of testing.
- Redact secrets, tokens, and passwords from screenshots and logs.
- Prioritise findings by user impact, business risk, and release readiness.
- Deliver a report usable directly by developers, product owners, and QA leads.
