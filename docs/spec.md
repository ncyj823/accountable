# Spec: the Accountable MVP

The requirements and the design for the MVP described in `docs/intent.md`. The intent is the authority on what we are building; this spec says how it works and how it looks. The stack is fixed by `docs/TECH_STACK.md` and the look by `docs/BRAND.md`.

## Concerns

Things the intent leaves open, or where this spec has made a call that is worth checking before the build starts.

1. **There is no brand skill installed.** `docs/BRAND.md` is the only brand reference in the repo, and AGENTS.md points to it for UI work, so this design follows that file. If a separate brand skill exists, say where and the colour and type sections here should be rechecked against it.
2. **The brand gives no darkened Sea Green or Marigold.** It says to darken them for small text on light backgrounds but does not say to what. This spec proposes five text colours derived from the brand palette, each above 4.5:1 on white. They are a proposal, not brand.
3. **The sample data will leave gaps in the charts.** 35 deals spread over six stages gives roughly eight to ten won deals across twelve months, so several months of the line and bar charts will sit at zero. The proposal is to read the intent's figures as a floor and seed about 70 deals: 35 open ones as described, plus closed deals across the past year so every month has something.
4. **The funnel's reading is a choice.** The intent leaves the dashboard charts to the designer. This spec defines each funnel band as the value of deals that have reached that stage, meaning deals at that stage or further along, with won deals counted in every band. Lost deals are left out, so a deal lost at Proposal does not appear in the funnel at all.
5. **Probability is taken as entered, with no range check.** That was decided in the intent. A probability above 100 would inflate expected revenue on the dashboard and the pipeline board. The proposal is that the form shows a quiet note when the figure is outside 0 to 100 but still saves it.
6. **Deleting a contact leaves their deals behind.** The intent cascades a contact delete to that contact's activities only, not to their deals. Those deals keep everything else and show "No contact" until someone picks a new one.
7. **Organization is optional on a contact and on a deal.** The intent says nothing is enforced beyond names, email format and email uniqueness, so requiring an organization would be a fourth rule. A contact or deal with no organization is allowed and shows as "No organization".
8. **Stage-change entries cannot be edited or deleted.** The intent says activities can be edited and deleted, and also that stage changes appear in the timeline by themselves. These two are in tension. The proposal is that entries the app writes are read-only; only notes, calls and emails can be edited or deleted.
9. **Card order on the board needs a field the intent does not mention.** Keeping a card where it was dropped means storing a position on each deal. It is never shown and never edited by hand, but it is in the database.
10. **No internet connection means the fonts ship with the app.** Instrument Sans, Instrument Serif and JetBrains Mono are installed as packages and served from the app, not loaded from Google Fonts. No library is loaded from a CDN.
11. **One close date does two jobs.** The intent words it as the expected or actual close date, so there is one field. When a deal is won or lost the date is read as the actual one, and the dashboard's monthly figures are bucketed by it.
12. **Today comes from the computer running the backend.** Overdue follow-ups, the seven-day window and the twelve-month chart window all use that machine's local date.

## What we are building

A sales CRM for one person, running on their own computer: a React frontend, a FastAPI backend and a SQLite file. Five sections, four kinds of record, no sign-in. Everything typed is kept and is still there after a refresh and after a restart.

## Architecture

```
package.json              root scripts, frontend/ as an npm workspace
pyproject.toml, uv.lock   the backend's dependencies
backend/
  app/
    main.py               the FastAPI app, routers, CORS for the dev server
    db.py                 engine, connection, foreign keys on, schema creation
    tables.py             the four tables, SQLAlchemy Core
    schemas.py            request and response models
    rules.py              stage probabilities, validation, expected revenue
    queries.py            every SELECT, INSERT, UPDATE and DELETE
    dashboard.py          the dashboard's figures
    seed.py               the sample data
    routers/              organizations.py contacts.py deals.py activities.py dashboard.py
  tests/                  pytest
frontend/
  src/
    main.tsx, App.tsx, routes.tsx
    api/                  the typed client and the query hooks
    theme/tokens.css      every colour, size and radius, in one place
    components/           tables, forms, dialogs, panels, charts, timeline, board
    pages/                Dashboard Organizations Contacts Deals Pipeline and the detail pages
  tests/                  Vitest
e2e/                      Playwright Test, one spec per success criterion
scripts/                  the dev and test runners
data/                     accountable.db and test.db, both git-ignored
docs/
```

The frontend holds no data of its own beyond the server's cached responses. Every figure on the screen comes from the backend, so the dashboard and the board cannot drift from the database.

### Running it

Root `package.json` scripts, the same on Mac, Linux and Windows:

| Command | What it does |
| --- | --- |
| `npm install` | installs the frontend workspace and, through a postinstall step, syncs the Python environment with `uv sync` |
| `npm run dev` | starts the backend on 127.0.0.1:8000 and Vite on 127.0.0.1:5173 together, and seeds the database if it is new |
| `npm test` | Vitest, then pytest, then Playwright; fails if any of them fails |
| `npm run seed` | drops the data and writes the sample data again |

Vite proxies `/api` to the backend in development, so the browser only ever talks to one origin. The backend binds to 127.0.0.1, not 0.0.0.0, because there is no sign-in.

The database file is `data/accountable.db`, overridable with `ACCOUNTABLE_DB_PATH`. The tests set that variable to `data/test.db`, so running them never touches real data. On startup the backend creates the schema if it is missing and seeds the sample data if the tables are empty.

## Data

SQLite through SQLAlchemy Core, no raw SQL, `PRAGMA foreign_keys = ON` on every connection so the cascades below are enforced by the database rather than by application code.

Money is stored as a whole number of cents in an integer column and sent over the API as a decimal number of dollars. Expected revenue is worked out in cents and rounded once, at the end, so the totals on the board and the dashboard always add up.

### organizations

| Column | Type | Notes |
| --- | --- | --- |
| id | integer | primary key |
| name | text | required |
| website | text | optional |
| industry | text | optional |
| notes | text | optional |
| created_at, updated_at | datetime | set by the app |

### contacts

| Column | Type | Notes |
| --- | --- | --- |
| id | integer | primary key |
| name | text | required |
| email | text | optional, must be a valid address when filled in |
| phone | text | optional |
| job_title | text | optional |
| organization_id | integer | optional, references organizations, on delete cascade |
| status | text | lead, qualified or customer; defaults to lead |
| created_at, updated_at | datetime | |

A unique index on the lowercased email covers only rows where the email is neither null nor empty, so any number of contacts may have no email while two cannot share one. Email comparison is case-insensitive.

### deals

| Column | Type | Notes |
| --- | --- | --- |
| id | integer | primary key |
| name | text | required |
| organization_id | integer | optional, references organizations, on delete cascade |
| contact_id | integer | optional, references contacts, on delete set null |
| stage | text | New, Qualified, Proposal, Negotiation, Won or Lost |
| value_cents | integer | defaults to 0 |
| probability | integer | percent; set from the stage on every stage change |
| close_date | date | expected while open, actual once won or lost |
| board_position | real | order within its column; never shown |
| created_at, updated_at | datetime | |

### activities

| Column | Type | Notes |
| --- | --- | --- |
| id | integer | primary key |
| kind | text | note, call, email or stage_change |
| contact_id | integer | optional, references contacts, on delete cascade |
| deal_id | integer | optional, references deals, on delete cascade |
| description | text | required |
| happened_at | datetime | required |
| follow_up_date | date | optional |
| follow_up_done | boolean | defaults to false |
| written_by_app | boolean | true for stage_change; those entries are read-only |

Every activity hangs off exactly one contact or one deal.

### Deleting

The cascades run in the database. Deleting an organization removes its contacts and its deals, and removing those removes the activities on them, so one delete clears the whole branch. Deleting a contact removes that contact's activities and empties the contact field on their deals. Deleting a deal removes that deal's activities.

Before anything is deleted the frontend asks `GET /api/{type}/{id}/impact`, which returns the counts of each kind of record that will go, and shows them in the confirmation: "Deleting Northwind Trading also deletes 4 contacts, 3 deals and 21 activities. This cannot be undone."

## Rules

**Stage probabilities.** New 10, Qualified 25, Proposal 50, Negotiation 75, Won 100, Lost 0. Whenever a deal's stage changes, in either direction, including reopening a closed deal, its probability is set to the new stage's figure. The figure is editable, and an edited figure stays until the next stage change.

**Expected revenue** of one deal is its value times its probability, divided by 100. Expected revenue of a set of deals is the sum of those amounts. Open pipeline totals, on the dashboard and on the board, count the four open stages only.

**Stage changes write themselves.** Any change of stage, from the board or from the deal form, writes an activity on that deal: kind `stage_change`, description "Moved from Proposal to Negotiation", `happened_at` now, `written_by_app` true. It shows in the deal's timeline and in the dashboard's recent activity, and cannot be edited or deleted.

**Late deals.** A deal is late when its stage is one of the four open ones and its close date is before today. Late deals are marked in the deals table and on their pipeline card.

**Follow-ups.** An activity with a follow-up date that is not done is a task. The dashboard lists tasks whose date is today or earlier, which are overdue, and those within the next seven days. Marking one done takes it off the list and leaves the activity in the timeline.

**What the app refuses to save.** A missing name on an organization, contact or deal, or a missing description on an activity. An email address that is filled in and not valid. An email address another contact already uses. Each refusal names the field and comes back as HTTP 422 with a field and a message, which the form shows under that field. Nothing else is refused: value and probability are taken as entered.

## The API

JSON under `/api`, with cents sent as decimal dollars and dates as ISO strings.

| Method and path | What it does |
| --- | --- |
| `GET /api/organizations?q=&sort=&dir=` | list, searched and sorted |
| `POST /api/organizations` | create |
| `GET /api/organizations/{id}` | one organization with its contacts and deals |
| `PATCH /api/organizations/{id}` | edit |
| `GET /api/organizations/{id}/impact` | what a delete would remove |
| `DELETE /api/organizations/{id}` | delete, cascading |
| `GET /api/contacts?q=&status=&sort=&dir=` | list, with the status filter |
| `POST`, `GET`, `PATCH`, `DELETE /api/contacts[/{id}]` | as above; the detail returns the organization and the timeline |
| `GET /api/deals?q=&stage=&sort=&dir=` | list |
| `POST`, `GET`, `PATCH`, `DELETE /api/deals[/{id}]` | as above; the detail returns the organization, the contact and the timeline |
| `PATCH /api/deals/{id}/placement` | the board drop: new stage and the ids of the cards above and below |
| `GET /api/board` | the six columns with their cards and their totals |
| `GET /api/activities?contact_id=&deal_id=` | a timeline, newest first |
| `POST`, `PATCH`, `DELETE /api/activities[/{id}]` | an entry the app wrote refuses a PATCH or DELETE with 409 |
| `PATCH /api/activities/{id}/follow-up` | mark done or not done |
| `GET /api/dashboard` | every figure on the dashboard, in one response |

Search looks at the name, and also at the industry and website for organizations, the email and job title for contacts, and the organization and contact names for deals.

The board drop sends neighbours rather than a number, so the backend owns the position arithmetic: the new position is the midpoint of the two neighbours, the top or bottom of the column when there is only one, and the column is renumbered when the midpoints get too close to tell apart. A new deal added in a column goes to the top of it.

### The dashboard response

- `months`: twelve entries, oldest first, ending with the current month. Each has the month, the number of deals won and the revenue won, taken from deals in the Won stage bucketed by their close date.
- `open_pipeline`: the count, total value and expected revenue of the deals in the four open stages.
- `won_this_month`: the count and revenue for the current month, for the tiles.
- `funnel`: five bands, New through Won. Each band is the total value of deals that have reached that stage: deals at that stage or further along, with won deals in every band and lost deals in none.
- `recent_activity`: the ten newest activities of any kind, each with its record and a link to it.
- `follow_ups`: every task not done whose date is today or earlier or within the next seven days, soonest first, each marked overdue or not.

## The look

### Tokens

Everything lives in `frontend/src/theme/tokens.css` as custom properties. No colour, radius or font size is written anywhere else.

Brand colours, used as fills and on large areas:

| Token | Value |
| --- | --- |
| `--sea-green` | `#06D6A0` |
| `--marigold` | `#FFC145` |
| `--coral` | `#F78C6B` |
| `--cerulean` | `#118AB2` |
| `--raspberry` | `#EF476F` |
| `--midnight` | `#073B4C` |
| `--sea-mist` | `#CDD8DB` |
| `--page` | `#F0F4F5` |
| `--panel` | `#FFFFFF` |

Text colours. Body copy is Midnight. These are for the places where a coloured word is needed, such as a stage name or an error, and each clears 4.5:1 on white. They are this spec's proposal, not brand (concern 2).

| Token | Value | From |
| --- | --- | --- |
| `--sea-green-text` | `#04815F` | Sea Green darkened |
| `--marigold-text` | `#946200` | Marigold darkened |
| `--coral-text` | `#A6411C` | Coral darkened |
| `--cerulean-text` | `#0D6E8E` | Cerulean darkened |
| `--raspberry-text` | `#C21E45` | Raspberry darkened |
| `--ink` | `#073B4C` | Midnight |
| `--ink-muted` | `#5A7580` | Midnight lightened, for labels and secondary lines |

Radius 16px on panels and dialogs, 10px on buttons, inputs and chips, 999px on status pills. Spacing on a 4px step, with 24px between panels and 20px inside them. Shadows are soft and tinted with Midnight: `0 1px 2px rgba(7,59,76,.06), 0 8px 24px rgba(7,59,76,.06)`. Panels are rounded with no outline, never a border on one edge, and there is no rule across the top of a page. No gradients except a single 12 per cent wash under the revenue line.

### Type

Instrument Sans for everything except the two cases below.

| Role | Size and weight |
| --- | --- |
| Page title | 28px / 600 |
| Panel heading | 18px / 600 |
| Body | 14px / 400 |
| Label and table heading | 12px / 600, uppercase, 0.04em tracking, `--ink-muted` |
| Stat tile figure | 28px / 500, JetBrains Mono |

JetBrains Mono carries money, percentages, counts and dates, in tables, tiles, chart axes and card figures, so columns of figures line up. Instrument Serif appears once per app: the line under the dashboard's title that says how the month is going. Money always shows its currency, as `$42,500`.

### Layout

A 240px sidebar as its own rounded white panel, floating on the page background with a 24px gutter around it, not a full-height bar with an edge. It holds the wordmark and the five links: Dashboard, Organizations, Contacts, Deals, Pipeline. The active link is a Sea Green pill with Midnight text; the others are Midnight on white and take a Sea Mist wash on hover.

To its right, the page: a header row with the page title on the left and the primary action on the right, then the content on a twelve-column grid with 24px gaps. The MVP is light mode and laid out for 1280px and wider, which is what the intent asks for.

### Components

**Buttons.** Primary is Sea Green with Midnight text. Secondary is Sea Mist with Midnight text. Delete is Raspberry with white text. All 10px radius, 36px high, no outlines. The focus ring is a 2px Cerulean outline, offset 2px.

**Tables** are built on TanStack Table. Above each one, a search box on the left and a New button on the right. The heading row is sticky, 12px uppercase labels, and clicking one sorts and shows a small arrow. Rows are 48px, separated by a 1px Sea Mist line rather than banding, and take a Sea Mist wash on hover. Figures are right-aligned in JetBrains Mono. The last column holds edit and delete icon buttons. An empty table says what to add and offers the New button again.

**Forms** open in a centred dialog, 520px wide, white, 16px radius, over a Midnight scrim at 40 per cent. Fields are stacked with 12px labels above. A field that is refused gets a Raspberry-tinted fill and a Raspberry message beneath it naming what is wrong. Save stays enabled; the check happens on the server and the dialog shows what comes back, so the form and the API never disagree.

**The delete confirmation** is the same dialog with the sentence from the impact call, the counts in JetBrains Mono, and Delete in Raspberry as the right-hand button.

**Status pills.** Lead is Marigold, qualified is Cerulean, customer is Sea Green, each with Midnight text, 999px radius, 12px / 600.

**Stage colours** run across the board, the deal records and the funnel, so one stage is one colour everywhere:

| Stage | Colour |
| --- | --- |
| New | Sea Mist, with Midnight text |
| Qualified | Cerulean |
| Proposal | Marigold |
| Negotiation | Coral |
| Won | Sea Green |
| Lost | Raspberry |

The brand's three-colour guidance applies where only three things sit side by side, such as a chart with three series. Six columns need six, so the full palette is used with a neutral at the start, where nothing has happened yet.

**The timeline** is a vertical list, newest first, each entry a coloured dot, the description, the time, and the record it is on. Note is Sea Mist, call is Cerulean, email is Marigold, stage change is Sea Green. An entry with a follow-up carries a chip with its date, Raspberry when overdue, and a tick to mark it done. The app's own stage-change entries have no edit or delete button.

**Charts** are Recharts, with JetBrains Mono axis labels in `--ink-muted`, horizontal grid lines in Sea Mist at half strength, no vertical grid, no chart junk, and a tooltip on a white panel with the same shadow as the cards.

### The five sections

**Dashboard.** A title, one Instrument Serif line beneath it, then:

- Row one, four stat tiles across: open pipeline value, expected revenue, deals won this month, revenue won this month. Each is a label, a JetBrains Mono figure and a one-line note.
- Row two: revenue won by month as a line chart over eight columns, Sea Green stroke with a 12 per cent wash beneath it, twelve points ending with the current month; and deals won by month as a bar chart over four columns, Marigold bars with 4px rounded tops, the same twelve months.
- Row three: the funnel over seven columns, as horizontal bands narrowing from New to Won, each band in its stage colour with the stage name and value on it; and the follow-ups panel over five, overdue ones first in Raspberry, each with its date, its record and a tick.
- Row four: recent activity, full width, the ten newest entries as the same timeline component, each linking to its record.

Together these are the line chart, the bar chart and the visual of the pipeline the intent asks for.

**Organizations.** A table of name, industry, website, contacts, deals and value. Opening one shows its details at the top, then its contacts and its deals as two tables. No timeline, as the intent says.

**Contacts.** A table of name, job title, organization, email, phone and status, with status pills in the last column and a status filter beside the search box: All, Lead, Qualified, Customer. Opening one shows their details and their organization, then their timeline, with a Log activity button above it.

**Deals.** A table of name, organization, contact, stage, value, probability, expected revenue and close date. The stage shows as a pill in its colour. A late deal's close date is Raspberry with a dot before it. Opening one shows its details, the organization and contact it is with, and its timeline, with a Log activity button.

**Pipeline.** Six columns across, scrolling sideways if the window is narrow, each a rounded panel with a 4px bar in its stage colour at the top of the column header. Column heading: the stage name, the deal count, and for the four open ones the total value and expected revenue under it, in JetBrains Mono. A card shows the deal name, the organization, the value, the close date and the main contact, with a Raspberry dot when late, and a left edge in its stage colour. Above the board, the open pipeline's total value and expected revenue, counting the four open columns only; every figure changes as soon as a card lands.

Dragging uses @dnd-kit. A card can be dropped anywhere in a column and stays there. Dropping into Won or Lost closes the deal and dragging it back out reopens it, like any other move. Each move sets the probability from the new stage and writes the timeline entry. Keyboard users can pick up a card with the space bar and move it with the arrow keys, and the deal form's stage field is the second way to do the same thing. Each column has an Add deal button that opens the form with that stage filled in.

## Sample data

`npm run seed` clears the tables and writes the sample data again, from a fixed random seed so the same command gives the same data, with dates worked out from today so the twelve-month window is always full. It writes about 15 organizations across several industries, 40 contacts covering all three statuses, and deals at every stage: the 35 the intent asks for, open and recently closed, plus enough older closed deals that every one of the last twelve months has wins and losses to draw (concern 3). Some open deals are past their close date. Activities cover notes, calls and emails, with follow-ups overdue and due in the coming week, so the dashboard has something in every panel on first run.

## Testing

**Vitest** covers the frontend's own logic: money and percentage formatting, expected revenue, the board and column totals, the funnel bands, the twelve-month axis, the late-deal rule, and how the form shows a refusal from the server.

**pytest** covers the backend: every endpoint; the cascades and the counts the impact call reports; email uniqueness, case-insensitive and allowing many blanks; the name and email refusals and their messages; the probability being set on every stage change in both directions; the activity written on a stage change and its refusal to be edited; the placement arithmetic including a renumber; and every dashboard figure against a fixed seeded dataset.

**Playwright Test** in `e2e/`, against the installed Google Chrome, starting the app itself with its own database and its own ports so nothing else needs to be running. One spec per success criterion below, each checking the browser console is clean.

## Success criteria, and where they are met

| From the intent | Where |
| --- | --- |
| 1. Start with one command and land on the dashboard with sample data | `npm run dev`, the seed on first run, the dashboard sections above |
| 2. Add, search, sort, edit and delete the three records; changes survive a refresh and a restart; the delete confirmation gives counts; the organization cascade; the reset command | the tables, the API, the database cascades, the impact call, `npm run seed` |
| 3. A record with no name, an invalid email or a duplicate email is refused, naming the field | the rules section and the form's field messages |
| 4. The status filter, linked records on each page, and a status that only changes when someone edits the contact | the contacts section; nothing in the app writes a status |
| 5. Dragging changes stage and probability both ways, a card stays where it was dropped, totals move at once, a deal added in a column starts at that stage, late deals stand out | the pipeline section, the placement endpoint, the probability rule |
| 6. Activities and stage changes in the timeline newest first; follow-ups on the dashboard until done, overdue and the next seven days | the activities API, the timeline component, the dashboard's follow-ups |
| 7. The dashboard's figures match the data and react to a win or a new follow-up; the charts cover the last twelve months to the current one | the dashboard endpoint, worked out from the database on each load |
| 8. Every screen sharp, modern and professional, on brand, with a stunning dashboard | the look section, the tokens, the three dashboard charts |
| 9. Used end to end in a real browser before it is called done, with no console errors | the Playwright suite plus a pass through the app by hand |

## Not in this version

Sign-in, user accounts and several people; dark mode and a phone layout; AI; email, calendar and phone integrations; currencies other than US dollars; reports beyond the dashboard; tags and custom fields; changing the stages; import and export. The open questions in `docs/intent.md` stay open.
