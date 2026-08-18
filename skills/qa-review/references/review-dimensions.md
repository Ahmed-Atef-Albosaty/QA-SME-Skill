# Review Dimensions (apply to every screen)

## 3.1 Functional
Purpose of the screen? Then: Is the feature complete? Does every visible control work? Are expected
actions available? Missing capabilities? Can the user complete the intended workflow? Correct navigation,
validations, business rules, calculations, permissions, confirmations? Are success and failure scenarios
handled? Any dead ends, unnecessary steps, or duplicated functionality? Flag incomplete/unfinished
implementations.

## 3.2 Workflow
For each workflow (login, registration, profile, CRUD, search, filtering, sorting, approvals, settings,
notifications, administration): evaluate number of steps, clarity, navigation, completion success, error
recovery, cancellation, back-navigation, and confirmation messages. Identify unnecessary complexity;
recommend simplifications.

## 3.3 Validation (every form)
Required fields, invalid inputs, empty inputs, duplicate data, special characters, long/short values,
boundary values, invalid formats, validation messages, input masking, keyboard entry. Are validations
complete and user-friendly?

## 3.4 UI
Layout, alignment, spacing, visual hierarchy, typography, colours, buttons, forms, tables, cards,
navigation, icons, images, empty/loading/error/success states, responsive behaviour, visual consistency.
Identify inconsistencies and design flaws that hurt usability.

## 3.5 UX
Learnability, discoverability, obvious primary action, feedback, efficiency, simplicity, consistency,
error prevention, error recovery, user confidence, and Nielsen's usability heuristics. Identify friction;
recommend improvements.

## 3.6 Accessibility (WCAG)
Keyboard navigation, focus visibility, colour contrast, touch-target size, semantic labels, screen-reader
compatibility, accessible error messages.

## 3.7 Design System
Consistency of typography, colours, spacing, border radius, shadows, icons, component reuse, naming
conventions, and interaction patterns. Highlight deviations from the design language.

## 3.8 Error Handling
Network failures, invalid requests, missing data, empty results, permission failures, session expiration,
unexpected user actions. Does the user get meaningful feedback?

## 3.9 Security Observations (black-box only)
Sensitive-information exposure, missing authorization checks, missing confirmation before destructive
actions, weak password rules (if observable), insecure file-upload behaviour, missing session-timeout
indicators, missing audit feedback.

## 3.10 Product
Missing features, unfinished features, opportunities for automation / efficiency / engagement / reduced
user effort, and future enhancements that would add value.
