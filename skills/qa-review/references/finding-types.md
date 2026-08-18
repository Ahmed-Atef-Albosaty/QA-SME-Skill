# Finding Types & Defect Reporting

Every issue gets **both** a SOP finding type and an ISTQB defect classification.

## The Four Finding Types (SOP)
| Type | When to use | Required fields | Routes to |
|---|---|---|---|
| **Bug** | Something is broken, doesn't work as expected, or blocks a user from completing a task. | Steps to reproduce · Expected result · Actual result · Screenshot | Engineering |
| **Note** | An observation worth recording that is NOT a defect - the feature works, you're just noting something. | What you observed · Screenshot (optional) | PM review |
| **Question** | The spec or client intent is unclear and you cannot determine pass/fail without clarification. | What is unclear · What you observed | PM (for answer) |
| **Idea** | A genuine improvement suggestion - something that works fine but could be better. | Suggestion description | Design backlog |

**Good bug note style** (one line, for the criteria sheet):
> "Settings > Password: 'Current Password' field required - blocks password change. Per spec: should not require it."

Bad: "Doesn't work." Bad: a full paragraph re-explaining the whole flow. Good: location → what's wrong →
what the spec says.

## ISTQB Defect Report (use for every Bug/Question - full form for the narrative report)
```
Title:            <short, specific>
Module:           <area of the app>
Screen:           <exact screen>
Category:         Functional | Validation | Workflow | UI | UX | Accessibility |
                  Performance | Security | Data Integrity | Business Logic |
                  Navigation | Permissions | Consistency
Severity:         Critical | High | Medium | Low | Suggestion
Priority:         P1 | P2 | P3 | P4
Preconditions:    <setup / account / state needed>
Steps to Reproduce:
  1. ...
  2. ...
Expected Result:  <what should happen>
Actual Result:    <what actually happened>
Impact:           <effect on user / business>
Recommendation:   <suggested fix>
Regression Risk:  <what else might be affected; is a regression pass recommended?>
```

Severity = how bad it is. Priority = how soon to fix. They are independent.

| Severity | Meaning | Priority | Meaning |
|---|---|---|---|
| Critical | Blocks core use / data loss / security hole | P1 | Fix before release |
| High | Major function broken; no workaround | P2 | Fix this sprint |
| Medium | Works but real problem; workaround exists | P3 | Later sprint |
| Low | Minor / cosmetic-plus | P4 | Backlog |
| Suggestion | Improvement idea, not a defect | - | - |

## Reconciling the two systems
- SOP **Bug** → ISTQB defect report, Severity/Priority assigned per the table above.
- SOP **Note** → still capture Category for consistency, Severity = Suggestion, no Priority (informational).
- SOP **Question** → log as a Question finding; do not guess a pass/fail verdict, mark the criteria-sheet
  cell for that item "N/A - pending clarification" and list the open question in the final report.
- SOP **Idea** → Severity = Suggestion, routes to "Future Enhancements" in the final report.

## Findings that don't map to any SOP checklist row
The SOP coverage-area checklist is fixed, but real defects don't always land on one of its rows (e.g. a
responsive layout break at one specific breakpoint, a dead admin control, a contrast issue on a page with no
dedicated checklist item). Never drop these because there's nowhere to put them: still record every such
Bug/UI finding in `findings.json` as usual, **and** list it in the criteria sheet's **"Out-of-Criteria Bugs"**
section (see `references/report-template.md`) with its screenshot - don't silently fold it into a Notes cell
on an unrelated row instead.
