# ScalingEasy QA SOP - Coverage Areas

Source: ScalingEasy Full QA SOP (Google Doc). These are the concrete pass/fail criteria to fill in for
every crawled screen, on top of the open-ended ISTQB review dimensions. Every check item gets a
**TRUE / FALSE / N/A** verdict for **Desktop, Tablet, and Mobile** - never leave a cell blank.

## Tag Rules (apply to every area below)
| Tag | Meaning | What to do |
|---|---|---|
| **ADMIN** | Admin-only action | Skip (mark N/A) when assessed as a standard member. Only evaluate with the admin role. |
| **TIER** | Test across all membership tiers | Repeat once per tier the client has (3 tiers = 3 verdicts per row). |
| **CUSTOM** | Client-specific feature | Only assess if the platform spec says this feature is enabled. Mark N/A otherwise. |

**N/A vs FALSE:** N/A means the feature is intentionally absent from this platform. If a feature should be
there per spec but is broken or missing, mark it **FALSE**, not N/A.

Devices: Desktop = 1920×1080 Chrome window, headed. Tablet = 1024×1366 (iPad Pro 12.9" dimensions), headed.
Mobile = 500×800 - the narrowest viewport the Selenium MCP server (`@angiejones/mcp-selenium`) can produce
(`--window-size` has a confirmed hard floor around 500px wide, both headed and headless; request
`--window-size=500,943` to get exactly 500×800). This is the fixed Mobile target for this skill going
forward, not a true device-preset width - narrower than a 500px viewport isn't achievable with this tooling.

---

## Area: Login & Auth
*Tags:* CUSTOM - Magic link as primary (only if client requested it)

**How to test:** Open the platform logged out → try magic link (if primary) end-to-end → try password
login → try Forgot Password (confirm branded reset email) → inspect all text for spelling errors → check
form centring/padding → repeat per device.

| Check Item |
|---|
| Magic link is set as primary login method (if requested) - CUSTOM |
| Magic link flow works end-to-end |
| Password login option is available and functional |
| Forgot password flow works |
| No spelling errors |
| No padding or layout errors |

## Area: Dashboard
**How to test:** Log in as admin → dashboard loads without error → click every button/link, confirm
correct navigation, use back button → click any checkboxes/to-do items, confirm state saves → trigger any
calendar reminder/notification bell → scan for blank space/uneven padding/overlap → read all text for
spelling → repeat per device.

| Check Item |
|---|
| All buttons functional and navigate to correct pages |
| All checkboxes functional (if any) |
| Calendar reminders / notifications functional |
| No excessive blank space or padding issues |
| No overlapping cards / text / descriptions |
| No spelling errors |

## Area: Courses & Videos
*Tags:* CUSTOM - "Mark as Complete" may not be enabled on all platforms

**How to test:** Navigate to Courses → open a course → play a lesson video → fullscreen it → right-click
video, confirm "Save as"/"Open in new tab" are NOT available (unless spec says otherwise) → complete a
lesson, confirm next lesson unlocks → click "Mark as Complete" if present → assess course UI cleanliness,
lesson list readability, progress visibility → exercise accordions/tabs/sidebar nav → repeat per device
(extra attention to video sizing/touch controls).

| Check Item |
|---|
| Video positioning and padding optimised for viewing |
| Course videos are playable |
| Course videos are expandable (fullscreen) |
| Videos cannot be downloaded or opened in a new window |
| Completing a lesson unlocks the next one |
| "Mark as complete" button works (if applicable) - CUSTOM |
| UI is clean and easy to follow |
| UX is fully functional - all interactions work |

## Area: Calendar / Calls
*Tags:* TIER - Tier visibility rows tested per tier | CUSTOM - RSVP if not universally enabled

**How to test - Meetings:** Add Meeting (title/date/time/call link, tier access) → confirm it appears in
upcoming calls and opens → edit it → delete it → create a recurring call (daily/weekly/monthly) → delete
one instance of a recurring call, confirm "this one" vs "all" prompt → RSVP as a standard member (if
enabled) → check timezone handling → confirm upcoming calls list is clickable.

**How to test - Recordings:** Add Recording (title + link) → confirm it appears → edit it → delete it →
bulk-delete if available → test tier visibility per tier.

| Check Item |
|---|
| Add meeting works |
| Tier access selection enabled |
| Edit meeting works |
| Delete meeting works |
| Call links added are functional and open |
| Recurring calls enabled: daily, weekly, monthly options all work |
| Deleting a recurring call prompts: this one or all |
| RSVP works (if enabled) - CUSTOM |
| Timezone auto-adjusted correctly |
| Upcoming calls visually displayed and clickable |
| Call recordings section is present |
| Add recording works |
| Edit recording works |
| Delete recording works |
| Bulk delete recordings works |
| Tier visibility tested across all tiers - TIER |

## Area: Community / Channels
*Tags:* ADMIN - channel management rows | TIER - tier assignment row | CUSTOM - AI moderation, message editing

**Important:** Most channel-management actions require admin access. Mark non-admin rows N/A when
assessing with a standard member account.

**How to test - Channel Management (Admin):** Add a channel → edit its name → set read-only, confirm a
standard member cannot send messages → delete it → assign a tier, confirm other tiers can't see it → pin
then unpin a channel.

**How to test - Visual:** channel names not truncated at any screen size; padding consistent; text size
matches app system text.

**How to test - Messaging:** send a message → react with emoji → reply in a thread → use the emoji picker →
check date separators and timestamps → edit a sent message (if allowed), confirm "(edited)" shows → @mention
another user, confirm in-app notification → post while on a different page, confirm badge count on return.

| Check Item |
|---|
| Add channel works - ADMIN |
| Read-only option works - ADMIN |
| Delete channel works - ADMIN |
| Edit channel works - ADMIN |
| Channel tier assignment tested across all tiers - TIER |
| Channel names are not truncated or cut off |
| Padding is consistent and correct |
| Text size matches app system text |
| Pin channels works - ADMIN |
| Unpin channel works - ADMIN |
| Bubble notifications appear for new messages |
| In-app notifications bubble when tagged |
| Sending a message works |
| Reactions work |
| Thread replies work |
| Emoji submission works |
| Date separator visible inside chat |
| Messages are timestamped |
| Message editing works (if enabled) - CUSTOM |
| AI moderation enabled and tested - CUSTOM |

## Area: Support / Tickets
*Tags:* ADMIN - internal notes row

**How to test:** As a standard member, create a new ticket (subject/description/topic) → open it, send a
reply → (admin) add an internal note, confirm it's labelled internal and hidden from the member → create a
ticket for each required topic and confirm it routes: Program Content Issue, Platform Question, Community &
Communication, Billing & Account Requests, Bug Report → confirm Bug Report tickets flag/forward internally
→ add an attachment, confirm it uploads and is visible.

| Check Item |
|---|
| Create new ticket works |
| Messaging on a ticket works |
| Internal notes work - ADMIN |
| Topic: Program Content Issue exists and routes correctly |
| Topic: Platform Question exists and routes correctly |
| Topic: Community & Communication exists and routes correctly |
| Topic: Billing & Account Requests exists and routes correctly |
| Topic: Bug Report exists and routes correctly |
| Bug reports sent internally |
| Attachments can be added to tickets |

## Area: User Management (Admin)
*Tags:* ADMIN - all rows in this section require admin access

**Important:** all rows require the admin role; mark all N/A if assessed only as a standard member.

**How to test:** Add a new user, confirm the branded invitation email arrives → edit the user's details →
assign CSA/CSM role, confirm that user cannot access admin-only areas → reset the user's password via
admin, confirm branded reset email → send a magic link via admin, confirm branded email → copy the magic
link and confirm it logs into the correct account in an incognito window → test the Onboarding Status
toggle → deactivate the user, confirm they can no longer log in → export the user list, confirm it
downloads with all expected fields → change a user's password directly via admin.

| Check Item |
|---|
| Add user works |
| Add user invite email branded |
| Edit user details works |
| CSA/CSM permissions restricted properly |
| Password reset works |
| Password reset email branded |
| Send magic link works |
| Send magic link email branded |
| Copy magic link works |
| Pasting magic link logs into account |
| Onboarding status button works |
| Deactivate user works |
| User export works |
| Export includes all required data fields |
| Change password works |

## Area: User Settings / Profile
**How to test:** As a standard member, edit display name → confirm change reflects in header/sidebar → try
editing email (mark N/A + Note if intentionally disabled) → edit phone number, confirm it persists on
reload → set a new password **without** re-entering the current password (per spec); if a "Current
Password" field is required, log as a **Bug** → upload a new profile picture, confirm it appears in the
avatar area → look for social links (Twitter/X, LinkedIn, Website); if absent and spec requires it, log a
**Bug** → repeat per device.

| Check Item |
|---|
| User can edit their name |
| User can edit email and phone |
| User can set password without current password |
| Profile picture upload works |
| Social links can be added |

## Custom Feature Areas
For any screen that doesn't map to the areas above (client-specific pages), duplicate this section under a
name matching the custom page, using the same table shape (Check Item | verdict per device | notes |
screenshot) plus the tag rules and the Cross-Cutting checks below.

Custom pages are identified during `SKILL.md`'s Step 2a discovery pass, alongside the 8 SOP areas - this is
what lets Step 2b offer each custom page as its own selectable item when the user chooses a full vs. partial
QA scope, the same way a whole SOP area is selectable.

**Every custom feature/page gets the same rigor as a standard SOP area - no shortcuts because it's
"just" a custom page:**
- **Full UI nitpick sweep at all three viewports** (Desktop 1920×1080, Tablet 1024×1366, Mobile 500×800) - fonts, padding, margins, spacing, alignment, visual coordination, per the nitpick checklist in
  `SKILL.md` Step 4. A custom page doesn't get a pass on any viewport just because it isn't in the fixed
  SOP list.
- **Every distinct user flow on the page tested end-to-end**, not just "the page loads and looks fine."
  Identify every create/edit/delete/submit/upload/navigate action the page exposes and actually exercise
  each one (using a disposable entity per Step 4b) through to its real, observed result - not just that
  the control is present and clickable. A custom page with 4 buttons needs 4 confirmed flows, not 1.

---

## Cross-Cutting Checks (apply to every area / every screen)

**Final sweep**
| Check Item |
|---|
| Tier-based access tested per spec - TIER |
| Tracking and storage verified (changes persist on reload) |

**Visual consistency (scan every page)**
- No broken images or icons.
- No raw code or placeholder text visible to users (e.g. "Lorem ipsum", `{variable_name}`).
- Font sizes and colours are consistent with the client's branding.
- Buttons have consistent styling (same corner radius, padding, colour scheme).

**Responsive / tablet & mobile**
- Nothing overflows the screen width on any preset.
- Touch targets (buttons, links) are large enough to tap without hitting adjacent elements.
- Modals and drawers open and close correctly on tablet and mobile.
- Scroll behaviour is natural (no scroll-locked pages unless intentional).
- Sidebar navigation collapses or transforms correctly on smaller screens.

## Common Mistakes to Avoid (apply these corrections automatically)
- Skipping tablet or mobile is not acceptable - every check item needs all three device verdicts.
- A finding needs steps → expected → actual to be actionable; never log "doesn't work" alone.
- Never mark N/A for something that should exist but is broken/missing - that's FALSE.
- TIER rows must be assessed per tier, not just as admin.
- Never leave a Desktop/Tablet/Mobile cell blank - if genuinely unverifiable, still write the reason.
