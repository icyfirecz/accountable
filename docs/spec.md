# Spec: the Accountable MVP

This spec turns `docs/intent.md` into requirements and a design. If the two disagree, `intent.md` wins and this file gets fixed. The stack is in `docs/TECH_STACK.md`, and the look comes from the brand skill (`.claude/skills/brand/SKILL.md`).

## Concerns

These need a decision or attention before or during the build. Where the spec makes a call, it says so, and the call can be changed.

All concerns below were reviewed and agreed on 2026-11-27. Concern 10 was changed: a new deal goes to the bottom of its column.

1. **Google Fonts and no internet.** The brand fonts come from Google Fonts, but the app has to work offline. *Call:* bundle the fonts with `@fontsource` packages so no request leaves the computer.
2. **Six stages, five brand series colors.** The brand gives five colors for things side by side, and the pipeline has six stages. *Call:* the four open stages use Sea Green, Marigold, Coral and Cerulean. Won uses dark Sea Green and Lost uses a neutral grey taken from Midnight. Raspberry is kept for "late" and alerts so it always means trouble.
3. **Bright brand colors as text.** Sea Green and Marigold fail contrast as small text on white. *Call:* darker text versions, listed under Design. Both pass 4.5:1 on white.
4. **Lost reason when dragging.** The intent adds a free-text loss reason, but dragging a card to Lost gives no natural place to type one. *Call:* dropping on Lost opens a small dialog with an optional reason and Skip. The reason is copied into the "Moved to Lost" timeline entry at that moment, so editing it later changes the deal but not the old entry. The maximum length is still open; the spec proposes 500 characters.
5. **Close date on Won and Lost.** The intent says "expected or actual close date" in a single field. *Call:* moving a deal to Won or Lost sets the close date to today unless the user has already set a date in the past. Moving it back to an open stage keeps the date, which the user can edit.
6. **Activities on deals and the contact timeline.** An activity belongs to a contact *or* a deal. A call logged on a deal won't show on its main contact's timeline. *Call:* each activity has exactly one parent. A contact's page shows its own timeline, plus a separate "Deals" list. Showing deal activities on the contact timeline as well is easy to add later if wanted.
7. **Main contact must belong to the deal's organization.** The intent doesn't say. *Call:* the contact picker on a deal only offers contacts of the chosen organization, and the API rejects a mismatch. The main contact is optional, as the required fields in the intent imply.
8. **Changing a contact's organization** when they are the main contact on deals with the old organization would break rule 7. *Call:* the API refuses and names the deals, the same way delete does.
9. **Stage changes in the timeline.** The intent's activity types are note, call and email, but stage changes also appear in timelines and recent activity. *Call:* they are stored as activities with a fourth, system-only type `stage_change`. They can't be created, edited or deleted by hand, and they are never follow-ups.
10. **Card order across moves.** "Stays where it was dropped" needs a stored position per stage. *Call:* each deal has an integer `position` within its stage, and the server rewrites positions in one transaction on every move. A new deal added in a column goes to the bottom.
11. **What "late" means.** *Call:* an open deal (not Won or Lost) whose close date is before today, by the computer's local date.
12. **Pagination with search and sort.** With about 40 deals, everything fits in the browser. *Call:* tables load all rows and search, sort and page in the browser. This needs revisiting once the data grows.
13. **Reset while running.** `npm run seed` while `npm run dev` is up could leave the open page showing stale data. *Call:* the seed replaces the database contents in one transaction, and the page shows the new data after a refresh. This is documented in `STATUS.md`.
15. **Won and Lost columns would grow without limit.** The dashboard counts about 50 won deals a year, so showing them all makes the column very long. *Call:* the Won and Lost columns show only deals closed in the last 30 days, and their count, total and expected revenue cover only those deals. Older ones are on the Deals page. Their headers say "last 30 days". Agreed on 2026-11-27, after mockup C was chosen.
14. **Still open from the interview:** whether tables remember their search, sort and page when you leave and come back. *Call for now:* they don't, and every visit starts fresh.

## Requirements

Requirement IDs (R1, R2, ...) are used by tests and review notes.

### General

- R1. `npm run dev` starts backend and frontend together on Mac, Linux and Windows, and prints the address to open. The app opens on the Dashboard.
- R2. `npm run seed` resets the database to the sample data. A new, empty database gets the sample data the first time the backend starts.
- R3. Every change is saved to SQLite immediately and survives refresh and restart.
- R4. No network requests leave the computer: fonts, icons and libraries are all bundled.
- R5. Light mode only, laid out for screens 1280px wide and up, checked in Google Chrome.
- R6. Money is shown in US dollars with the currency, for example `$12,500` or `$1.2M` on charts. Values are whole dollars.
- R7. No errors in the browser console during normal use.

### Records

**Organization**

| Field | Type | Required | Notes |
|---|---|---|---|
| name | text, max 200 | yes | |
| website | text, max 300 | no | shown as a link |
| industry | text, max 100 | no | free text, with suggestions from existing values |
| notes | text, max 5000 | no | |

**Contact**

| Field | Type | Required | Notes |
|---|---|---|---|
| name | text, max 200 | yes | |
| organization | link | yes | |
| status | lead, qualified, customer | yes | defaults to lead |
| email | text, max 300 | no | checked as an email address if given |
| phone | text, max 50 | no | |
| job title | text, max 200 | no | |

**Deal**

| Field | Type | Required | Notes |
|---|---|---|---|
| name | text, max 200 | yes | |
| organization | link | yes | |
| main contact | link | no | must belong to the deal's organization |
| stage | New, Qualified, Proposal, Negotiation, Won, Lost | yes | defaults to New |
| value | whole USD, 0 or more | yes | |
| probability | 0 to 100, whole percent | yes | set from the stage, can be changed by hand |
| close date | date | no | expected for open deals, actual for Won and Lost |
| lost reason | text, max 500 | no | only shown when stage is Lost |
| position | integer | system | order within the stage column |

Default probabilities: New 10, Qualified 25, Proposal 50, Negotiation 75, Won 100, Lost 0.

**Activity**

| Field | Type | Required | Notes |
|---|---|---|---|
| type | note, call, email (plus system `stage_change`) | yes | |
| contact or deal | link | exactly one | |
| description | text, max 5000 | yes | |
| happened at | date and time | yes | defaults to now |
| follow-up date | date | no | |
| done | yes/no | no | only meaningful with a follow-up date |

- R8. Each record keeps `created_at` and `updated_at`.
- R9. Validation runs in the browser for quick feedback and again in the API, which is the source of truth. Errors show next to the field in plain words.

### Tables (Organizations, Contacts, Deals)

- R10. Every table has a search box that matches any visible text column, case-insensitive, as you type.
- R11. Clicking a column heading sorts by it, and clicking again reverses. The default sort is name A to Z, and Deals defaults to close date with the soonest first.
- R12. Pages of 25 rows, with the page number and total count shown.
- R13. An "Add" button opens a form in a side panel. Each row has Edit, which opens the same panel, and Delete.
- R14. Clicking a row opens that record's page.
- R15. Columns:
  - Organizations: name, industry, website, number of contacts, number of open deals, open pipeline value.
  - Contacts: name, organization, status, job title, email, phone.
  - Deals: name, organization, main contact, stage, value, probability, close date. Late deals get a Raspberry "Late" badge and their close date shown in Raspberry.
- R16. Contacts has a status filter: All, Lead, Qualified, Customer, with a count on each.

### Deleting

- R17. Delete always opens a confirmation that names the record and says what else goes with it:
  - Organization: "Also deletes 3 contacts, 2 deals and 14 activities."
  - Contact: "Also deletes 5 activities." If the contact is a main contact on any deal, the dialog doesn't offer Delete. It lists those deals and says to give each one another main contact first.
  - Deal: "Also deletes 6 activities."
  - Activity: deletes only itself.
- R18. The API enforces the same rules. Deleting a contact who is a main contact returns 409 with the list of deals.

### Record pages

- R19. Organization page: details with Edit and Delete, its contacts, and its deals with stage and value.
- R20. Contact page: details with the organization as a link, their deals, and their timeline. It has a Log activity button.
- R21. Deal page: details with organization and contact as links, stage, value, probability, close date, the lost reason when Lost, and its timeline. It has a Log activity button. The stage can be changed here as well as on the board.
- R22. Timelines are newest first. Each entry shows its type icon, description, date and time, and follow-up status. Notes, calls and emails have Edit and Delete. Stage changes don't.

### Activities and follow-ups

- R23. Log activity opens a form: type, description, when (defaults to now), and an optional follow-up date.
- R24. A follow-up is "open" until marked done. Open follow-ups with a date before today are overdue, and those from today onward are upcoming.
- R25. A follow-up can be marked done from the dashboard or from the timeline.

### Stages and probability

- R26. Changing a deal's stage, from the board, the deal page or the edit form, sets its probability to the new stage's default and adds a `stage_change` activity on the deal: "Moved from Proposal to Negotiation". Moving to Lost adds the reason if one was given: "Moved from Negotiation to Lost: went with a cheaper supplier".
- R27. A probability changed by hand stays until the next stage change.
- R28. Concerns 4 and 5 apply: the Lost dialog and the close date on Won or Lost.

### Pipeline

- R29. One column per stage, in order: New, Qualified, Proposal, Negotiation, Won, Lost. Each column header shows the count, total value and expected revenue for that column. The Won and Lost columns show only deals closed in the last 30 days (concern 15), and their header figures cover only those deals.
- R30. Above the board: total value and expected revenue of open deals (New through Negotiation).
- R31. Cards show deal name, organization, value, probability, close date and main contact. Late cards get the Raspberry badge.
- R32. Dragging a card to any place in any column moves it there. The board updates at once (optimistic) and saves in the background. If saving fails, the card goes back and a message says so.
- R33. Each column has "Add deal", which opens the deal form with that stage set. The new card goes to the bottom of the column.
- R34. Clicking a card opens the deal page.

### Dashboard

- R35. Summary figures across the top: open pipeline value, expected revenue, revenue won this month, and deals won this month. Each shows a small comparison with last month where it makes sense.
- R36. Line chart: revenue won per month for the last 12 months, by close date of Won deals.
- R37. Bar chart: deals won per month for the last 12 months. Lost per month can show beside it in grey.
- R38. Pipeline visual: a funnel of value by stage, New through Won. Value "reaching" a stage counts the deals currently in that stage or any later stage, Won included. Lost is shown apart.
- R39. Recent activity: the 10 newest activities of any type, stage changes included, each linking to its contact or deal.
- R40. Follow-ups: overdue first in Raspberry, then upcoming, each with a Done button and a link to its record.
- R41. Every figure is worked out by the backend from the same data the tables show. A dashboard test checks each one against a known data set.

### Sample data

- R42. About 12 organizations across at least 5 industries, about 30 contacts at all three statuses, and about 40 deals at every stage. That includes Won and Lost deals spread over the past 12 months, open deals closing over the next 3 months, at least 3 late deals, and a few lost reasons.
- R43. About 80 activities across contacts and deals, including stage changes that match each deal's history, at least 3 overdue follow-ups and at least 3 upcoming in the next week.
- R44. Dates are generated relative to today, so the data always looks current.

## Design

### Architecture

```
browser  ──>  Vite dev server (React, :5173)  ──proxy /api──>  FastAPI (:8000)  ──>  SQLite file
```

- `backend/`: FastAPI app, SQLAlchemy Core tables, Pydantic models for requests and responses, and the sample data generator. The database file is `backend/data/accountable.db`, which is ignored by git. Tests point to a temporary database using an environment variable.
- `frontend/`: React with TypeScript and Vite. Data is fetched through TanStack Query, which caches and refetches after every change.
- Root `package.json` runs both with `concurrently`. `npm run seed` calls `uv run python -m backend.seed`.

### Libraries

See `docs/TECH_STACK.md`.

### Database

Tables `organizations`, `contacts`, `deals`, `activities` with the fields above, plus `id`, `created_at` and `updated_at`.

- `contacts.organization_id` → organizations, ON DELETE CASCADE.
- `deals.organization_id` → organizations, ON DELETE CASCADE.
- `deals.contact_id` → contacts, nullable, ON DELETE RESTRICT, which backs R18.
- `activities.contact_id` and `activities.deal_id` are nullable, ON DELETE CASCADE, with a CHECK that exactly one is set.
- `deals (stage, position)` is indexed. SQLite foreign keys are turned on per connection.

### API

All JSON, under `/api`.

| Method and path | Purpose |
|---|---|
| `GET/POST /organizations` | list (with contact and deal counts), create |
| `GET/PATCH/DELETE /organizations/{id}` | one with its contacts and deals, update, delete |
| `GET /organizations/{id}/delete-impact` | counts for the confirm dialog |
| `GET/POST /contacts` | list (with `?status=`), create |
| `GET/PATCH/DELETE /contacts/{id}` | one with org, deals and timeline, update, delete (409 if a main contact) |
| `GET /contacts/{id}/delete-impact` | activity count, blocking deals |
| `GET/POST /deals` | list, create (stage sets probability, goes to bottom of column) |
| `GET/PATCH/DELETE /deals/{id}` | one with timeline, update (stage change logs an activity), delete |
| `POST /deals/{id}/move` | `{stage, index, lost_reason?}`: moves on the board, rewrites positions, logs the stage change |
| `GET /deals/{id}/delete-impact` | activity count |
| `POST /activities` | create on a contact or deal |
| `PATCH/DELETE /activities/{id}` | update or delete (refused for `stage_change`) |
| `POST /activities/{id}/done` | mark a follow-up done |
| `GET /dashboard` | summary figures, monthly series, funnel, recent activity, follow-ups |

Errors return `{"detail": "...", "fields": {...}}` with 400 for validation, 404, and 409 for blocked deletes and organization mismatches.

### Screens and layout

- **Chosen look: mockup C** (`docs/mockups/c-dashboard.html` and `c-pipeline.html`, with screenshots beside them), chosen on 2026-11-27 over mockups A and B. Where this spec and mockup C differ on looks, the mockup wins. On behavior, the spec wins.
- A fixed left sidebar, 224px wide, filled Midnight, with Sea Mist link text. It shows the Accountable wordmark and five links (Dashboard, Organizations, Contacts, Deals, Pipeline), each with a lucide icon, and the current page in Sea Green fill with Midnight text. The page itself stays on the light background.
- Pipeline: three summary tiles above the board (open pipeline in Sea Green, expected revenue in Marigold, late deals in Coral). Each column has a header filled with its stage color showing the stage name and card count, then the column's value and expected revenue. A card being dragged lifts with a shadow, and the spot where it will land shows as a light Sea Green block.
- Content sits on the page background with a page title (Instrument Sans, 28px, semibold) and the main action at top right.
- Tables, record details, timelines and chart blocks each sit in a white panel with 12px rounding, no outline, and a soft shadow. There are no one-edge borders and no rule across the top of the page.
- Forms open in a side panel from the right, 480px wide. Confirmations are centered dialogs.
- Routes: `/` dashboard, `/organizations`, `/organizations/:id`, `/contacts`, `/contacts/:id`, `/deals`, `/deals/:id`, `/pipeline`.

### Brand applied

**Fonts**

- Instrument Sans for everything by default: headings 600 weight, body 400 at 14 to 15px.
- Instrument Serif for the one accent line on the dashboard, a short greeting or tagline under the title, and nowhere else.
- JetBrains Mono for money, percentages, counts, dates in tables, and record IDs, with tabular figures so columns line up.

**Color tokens**

| Token | Value | Use |
|---|---|---|
| `--page` | `#F0F4F5` | page background |
| `--panel` | `#FFFFFF` | panels and cards |
| `--ink` | `#073B4C` Midnight | text |
| `--ink-muted` | Midnight at 65% | secondary text |
| `--sea-green` | `#06D6A0` | primary fill, Midnight text |
| `--sea-green-text` | `#037D5E` | Sea Green as text on white (about 5.1:1) |
| `--marigold` | `#FFC145` | secondary fill, Midnight text |
| `--marigold-text` | `#8A5A00` | Marigold as text on white (about 5.9:1) |
| `--coral` | `#F78C6B` | tertiary fill, Midnight text |
| `--cerulean` | `#118AB2` | information, white text |
| `--raspberry` | `#EF476F` | late, overdue, delete, white text |
| `--neutral` | Midnight at 30% | Lost, disabled |

Stage colors, used in the same order everywhere (board column tags, funnel, charts):

| Stage | Color |
|---|---|
| New | Sea Green |
| Qualified | Marigold |
| Proposal | Coral |
| Negotiation | Cerulean |
| Won | `--sea-green-text` |
| Lost | `--neutral` |

Contact statuses: lead is Marigold, qualified is Cerulean, customer is Sea Green, each as a pill with the matching text color.

Buttons:

- Primary is Sea Green fill with Midnight text.
- Secondary is a white button with Midnight text and a light shadow.
- Destructive is Raspberry with white text.

Charts use flat fills with no gradients. The line chart gets one light Sea Green area fill under the line, and that is the one allowed soft fill.

**Tone**

- Labels are short and plain: "Add deal", "Log activity", "Mark done", "Delete organization".
- No exclamation marks. Empty states say what to do: "No deals yet. Add one to start your pipeline."
- Money always carries `$`.

### Dashboard layout

1. Title "Dashboard" with the Instrument Serif accent line beneath it.
2. A row of four summary tiles filled with brand colors: Sea Green, Marigold and Coral with Midnight text, Cerulean with white text. Each has the figure in JetBrains Mono, a label, and the change versus last month.
3. A two-column row: the revenue-won line chart, wide, and the funnel drawn as a tapered stack with each stage in its stage color.
4. A two-column row: the deals-won bar chart and the follow-ups list.
5. Recent activity, full width.

### Testing

- **Backend (pytest):**
  - every endpoint, including the delete rules, the 409 cases, stage moves and positions, probability reset, the Lost reason, the dashboard figures against a fixed data set, and that the seed covers R42 to R44.
- **Frontend (Vitest):**
  - money and date formatting, "late" and "overdue" logic, the table search, sort and page, and the form validation.
- **End-to-end (Playwright Test, installed Chrome, own database):**
  - starting on the dashboard;
  - adding, searching, sorting, editing and deleting in each table;
  - the contact status filter;
  - dragging a card across columns and within a column, then refreshing;
  - dropping on Lost with a reason;
  - logging an activity with a follow-up and seeing it on the dashboard;
  - marking it done;
  - the blocked contact delete;
  - no console errors on any page.
- `npm test` runs all three. Success criterion 8, a person using it end to end in Chrome, is done by hand before calling the MVP finished.

## Out of scope

As listed in `intent.md`, plus anything not stated above.
