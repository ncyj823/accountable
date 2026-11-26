# Plan: building the Accountable MVP

How `docs/spec.md` gets built, from an empty repo to the nine success criteria in `docs/intent.md`.
No code exists yet, so every file below is new.

Read this with the spec open. Where the two disagree, the spec wins and this plan is wrong.

## Files that change

### Root

| File | What is in it |
| --- | --- |
| `package.json` | the four scripts, `frontend` as a workspace, `concurrently` and `@playwright/test` as dev dependencies |
| `package-lock.json` | the lock for the workspace |
| `pyproject.toml` | Python 3.12, FastAPI, uvicorn, SQLAlchemy, pydantic, pytest, httpx |
| `uv.lock` | the backend's lock |
| `playwright.config.ts` | the Chrome channel, the `webServer` entries, the test database and ports |
| `.gitignore` | add `data/` |
| `docs/STATUS.md` | how to run the app, once it runs |

### Backend

| File | What is in it |
| --- | --- |
| `backend/app/__init__.py` | empty |
| `backend/app/settings.py` | `ACCOUNTABLE_DB_PATH`, the port, and `today()` as a seam the tests can freeze |
| `backend/app/db.py` | the engine, `PRAGMA foreign_keys = ON` on every connection, schema creation, the seed-if-empty check |
| `backend/app/tables.py` | the four tables, their foreign keys and delete rules, and the partial unique index on the lowercased email |
| `backend/app/rules.py` | stage probabilities, expected revenue in cents, the late-deal rule, the follow-up windows, validation |
| `backend/app/schemas.py` | request and response models, cents in and dollars out |
| `backend/app/queries.py` | every statement; no SQL text anywhere else |
| `backend/app/placement.py` | the board's position arithmetic and the renumber |
| `backend/app/dashboard.py` | the six parts of the dashboard response |
| `backend/app/seed.py` | the sample data, and the `--reset` entry point behind `npm run seed` |
| `backend/app/main.py` | the app, the routers, the 422 shape, CORS for the dev server |
| `backend/app/routers/organizations.py` | list, create, read, edit, impact, delete |
| `backend/app/routers/contacts.py` | the same, plus the status filter |
| `backend/app/routers/deals.py` | the same, plus `placement` and `board` |
| `backend/app/routers/activities.py` | the timeline, create, edit, delete, the follow-up toggle, the 409 on app-written entries |
| `backend/app/routers/dashboard.py` | one endpoint |
| `backend/tests/` | `conftest.py` with a fresh database per test, plus the files listed under Proof |

### Frontend

| File | What is in it |
| --- | --- |
| `frontend/package.json`, `vite.config.ts`, `tsconfig.json` | the workspace, the `/api` proxy, strict TypeScript |
| `frontend/index.html`, `src/main.tsx`, `src/App.tsx`, `src/routes.tsx` | the shell and the five routes with their detail routes |
| `frontend/src/theme/tokens.css` | every colour, radius, size and shadow from the spec, and nothing hard-coded elsewhere |
| `frontend/src/theme/fonts.css` | the three faces from `@fontsource`, served by the app, never from a CDN |
| `frontend/src/api/client.ts`, `src/api/hooks.ts`, `src/api/types.ts` | the typed client and the TanStack Query hooks |
| `frontend/src/lib/format.ts` | money, percentages, dates, and the relative times in the timeline |
| `frontend/src/lib/derive.ts` | the figures the screen works out for itself: board totals, funnel bands, the twelve-month axis, late and overdue |
| `frontend/src/components/TopBar.tsx` | the navigation panel, the app-wide search, the primary action |
| `frontend/src/components/PageHead.tsx` | the Instrument Serif title and its one plain line |
| `frontend/src/components/Panel.tsx`, `StatCell.tsx` | the panel and the divided three-up figures |
| `frontend/src/components/DataTable.tsx` | TanStack Table, its own search, sortable headings, row actions, the empty state |
| `frontend/src/components/FormDialog.tsx` | the dialog, the stacked fields, the field messages from a 422 |
| `frontend/src/components/DeleteDialog.tsx` | the impact sentence with its counts |
| `frontend/src/components/Timeline.tsx` | the entries, their dots, the follow-up chip, no actions on app-written entries |
| `frontend/src/components/charts/` | `RevenueLine.tsx`, `DealsBar.tsx`, `Funnel.tsx`, on Recharts |
| `frontend/src/components/board/` | `Board.tsx`, `Column.tsx`, `DealCard.tsx`, on @dnd-kit |
| `frontend/src/pages/` | `Dashboard`, `Organizations`, `OrganizationDetail`, `Contacts`, `ContactDetail`, `Deals`, `DealDetail`, `Pipeline` |
| `frontend/src/forms/` | `OrganizationForm`, `ContactForm`, `DealForm`, `ActivityForm` |
| `frontend/tests/` | Vitest, the files listed under Proof |

### End-to-end

| File | What is in it |
| --- | --- |
| `e2e/fixtures.ts` | a fixture that reseeds before each spec and fails the spec on any console error |
| `e2e/*.spec.ts` | nine specs, one per success criterion, listed under Proof |

### Scripts

| File | What is in it |
| --- | --- |
| `scripts/dev.mjs` | starts the backend and Vite together and waits for both |
| `scripts/test.mjs` | Vitest, then pytest, then Playwright, stopping at the first failure |
| `scripts/check-uv.mjs` | the postinstall check that says how to install uv if it is missing |

## Order of work

Backend first, then the seed, then the screens. The frontend is never built against made-up data, so
no screen is written twice and no figure is worked out in two places.

Each phase ends with its own tests passing and is a commit point.

### Phase 1 — It runs

The root `package.json` with the four scripts, the workspace, `pyproject.toml`, the Vite app, and a
FastAPI app with one `/api/health` route. `scripts/dev.mjs` starts both. `scripts/check-uv.mjs` fails
`npm install` with a readable message where uv is missing.

Done when `npm install` then `npm run dev` on a clean clone opens a page that reads the backend.

### Phase 2 — The database and the rules

`tables.py`, `db.py` with the foreign-key pragma, and `rules.py`. No HTTP yet. The cascades, the
partial unique index, the stage probabilities, expected revenue in cents and the late and follow-up
windows all exist and are tested directly.

This is the phase to get wrong cheaply. Everything above it assumes these are right.

Done when the backend's rule and schema tests pass.

### Phase 3 — The records API

`schemas.py`, `queries.py`, the four record routers. Create, read, edit, delete, search, sort, the
status filter, the impact counts, and the 422 shape with its field and message. The stage change
writes its own activity, resets the probability, and fills in a missing close date when the deal is
closed. An app-written activity refuses a PATCH or DELETE.

Done when every endpoint in the spec's table except `placement`, `board` and `dashboard` is covered by
a passing test.

### Phase 4 — The board and dashboard endpoints

`placement.py` with the midpoint arithmetic and the renumber, `GET /api/board`, and `dashboard.py`
with all six parts of the response. The twelve-month window, the funnel's cap on won deals, and the
open-only totals are decided here, once, on the server.

Done when the dashboard and board responses match hand-worked figures over a fixed dataset.

### Phase 5 — The sample data

`seed.py`: about 70 deals, 15 organizations, 40 contacts, activities with follow-ups, all from a fixed
random seed with dates worked out from `today()`. `npm run seed` resets and rewrites it.

It comes before the screens so that every screen is built against the data it will really show, and
the twelve-month charts have a full year from the first render.

Done when a fresh database gives a dashboard response with no empty month.

### Phase 6 — The shell and the tokens

`tokens.css`, `fonts.css`, the top bar, the page head, the panel, routing, the API client and the
formatting helpers. One page renders with real data.

Done when all five routes resolve, the three fonts load from the app with the network off, and no
colour is written outside `tokens.css`.

### Phase 7 — The tables, the forms and the timelines

`DataTable`, `FormDialog`, `DeleteDialog`, `Timeline`, the three record pages and their detail pages,
and the four forms. Search, sort, the status filter, the field messages, the delete confirmation with
its counts, logging an activity and marking a follow-up done.

This is the largest phase. It can be split per record type if it grows: organizations first, because
its detail page is the simplest, then contacts, then deals.

Done when criteria 2, 3, 4 and 6 can be walked by hand.

### Phase 8 — The board

`Board`, `Column`, `DealCard` on @dnd-kit. Dragging between and within columns, the position kept
after a refresh, the totals moving as a card lands, adding a deal into a column, late deals marked,
and the keyboard path through dnd-kit's keyboard sensor.

Done when criterion 5 can be walked by hand, with a keyboard as well as a mouse.

### Phase 9 — The dashboard

The hero, the three-up figures, the funnel, the bar chart, the follow-ups and the recent activity, on
Recharts, in mockup B's layout.

Done when criteria 1, 7 and 8 can be walked by hand.

### Phase 10 — The end-to-end suite

`playwright.config.ts` with its own ports and database, the console-error fixture, and the nine specs.
`npm test` runs all three suites in order.

Done when `npm test` passes from clean on this machine.

### Phase 11 — The pass by hand

Criterion 9. Someone opens the app and uses it the way a salesperson would, every section and every
action, each screen checked by eye, console open. Passing tests is necessary and is not enough.

## Risks

**Foreign keys are off by default in SQLite.** Every cascade in the spec rests on the pragma, and it is
per connection, not per database. Set it on SQLAlchemy's `connect` event, not once at startup, and
prove it with a test that deletes an organization and counts what is left. Without this the deletes
look like they work and quietly orphan rows.

**A won deal with no close date would vanish.** The monthly charts, the funnel's twelve-month cap and
`won_this_month` all bucket by `close_date`, which the spec makes optional, so a deal dragged into Won
without one would be counted nowhere and the dashboard would not add up. Settled: closing a deal fills
the date in, and anything that still lacks one falls back to the moment it entered the stage. Both
halves are now in the spec under "Closing a deal dates it". Build them together — the fallback alone
leaves the deals table showing a blank date on a closed deal, and the fill alone still loses rows
written around the app.

**Fractional positions collapse.** Repeated drops between the same two cards halve the gap each time
and reach the limit of a float in about fifty moves. Renumber the column whenever a gap falls below a
threshold, and test it by dropping fifty times in the same place rather than trusting the arithmetic.

**Money drifts if it is ever a float.** Cents in the column, cents through the rules, one rounding when
expected revenue is worked out, dollars only at the edge. A single `float` in the middle makes board
totals and dashboard totals disagree by a cent, which looks like a bug in the figures.

**"Today" makes tests brittle.** Overdue, the seven-day window and the twelve-month axis all move every
day, and the seed is written relative to today. Put `today()` in `settings.py` and let the tests freeze
it; never call `date.today()` anywhere else. Tests that hard-code a date will pass today and fail in
December.

**Drag and drop is the flakiest thing to test.** Mouse drags in Playwright depend on timing and have to
be chased. Drive the board in the end-to-end tests through dnd-kit's keyboard sensor, which is a real
user path the spec already requires, and keep mouse dragging to the pass by hand.

**The end-to-end run needs its own everything.** Fixed ports and a shared database file will collide
with a `npm run dev` left running. Give Playwright its own ports, its own `ACCOUNTABLE_DB_PATH`, and a
`webServer` that starts and stops them, so `npm test` needs nothing else running and never touches real
data.

**uv may not be installed.** `npm install` syncing the Python environment is convenient until it fails
on a machine without uv, where the error is unreadable. Check for it first and say how to install it.
Criterion 1 is about someone non-technical getting the app open, and this is where they would stop.

**The fonts have to work offline.** The app has to run with no internet connection, so the three faces
come from `@fontsource` packages. Check that Instrument Sans, Instrument Serif and JetBrains Mono all
exist there in the weights the spec names before Phase 6 depends on them; if one does not, the files
go in the repo instead.

**Phase 7 is big enough to stall.** Three record types, four forms, two dialogs and a timeline is most
of the UI. Build organizations end to end first and let the other two follow its shape, rather than
building every table, then every form, then every dialog.

**Recharts may not give mockup B's funnel.** The chosen look draws the funnel as five vertical bars with
a value above and a stage name and count below. That is a bar chart with custom labels, not Recharts'
own funnel; use the bar chart. Decide this in Phase 9 before building around the wrong component.

**Long names break the 44px serif title and the 206px board columns.** The mockup used short names.
Clamp the page title to two lines, let card titles wrap to two and truncate after, and check it with a
deliberately long organization name in the seed.

## Proof

Every test below is written in the phase that introduces its behaviour, not at the end.

### Unit, backend, pytest

`backend/tests/`, a fresh database per test, `today()` frozen.

| File | What it proves |
| --- | --- |
| `test_schema.py` | the pragma is on for every connection; the partial unique index allows many blank emails and refuses a second real one, ignoring case |
| `test_rules.py` | each stage's probability; expected revenue for one deal and for a set, in cents, rounded once; the late-deal rule either side of today; the overdue and seven-day boundaries |
| `test_validation.py` | a missing name on each of the three records and a missing description on an activity; an invalid email; a duplicate email; each refusal returns 422 with the field and a message; a blank email saves; a probability of 150 saves |
| `test_organizations.py` | create, read, edit, delete; search over name, industry and website; sort both ways on each column |
| `test_contacts.py` | the same, plus the status filter and the detail's organization and timeline |
| `test_deals.py` | the same, plus the detail's organization, contact and timeline |
| `test_cascade.py` | deleting an organization removes its contacts, its deals and all their activities; deleting a contact removes that contact's activities and empties the contact on their deals, leaving the deals; deleting a deal removes its activities; the impact counts match what the delete then removes |
| `test_stage_changes.py` | the probability is set from the stage in both directions, including reopening a closed deal; a probability set by hand survives an edit that does not change the stage and is replaced by one that does; each change writes one activity with the right description; a deal closed with no close date is given today's, by a drag, by an edit and by being added straight into the column, and a deal closed with one keeps it; reopening leaves the date alone |
| `test_activities.py` | create, edit and delete a note, call and email; a PATCH or DELETE of an app-written entry returns 409; the follow-up toggle; the timeline comes back newest first |
| `test_placement.py` | a drop between two cards takes the midpoint; a drop at the top and at the bottom of a column; fifty drops in the same gap trigger a renumber and keep the order; a new deal goes to the top of its column; a drop into another column changes the stage |
| `test_board.py` | the six columns and their cards; each open column's value and expected revenue; the board totals count the four open columns only |
| `test_dashboard.py` | twelve months, oldest first, ending with the current one; the won counts and revenue per month; `open_pipeline` and `won_this_month`; the funnel's five bands never widen, count won deals only from the last twelve months, and leave lost deals out; a won deal whose close date was cleared behind the app's back is still counted, from the month it entered Won; the months add up to the same total as the won deals themselves; `recent_activity` is the ten newest; `follow_ups` holds the overdue and the next seven days and nothing else |
| `test_seed.py` | the seed is deterministic from its fixed seed; it writes every stage and every status; no month in the last twelve is empty; `--reset` clears first |

### Unit, frontend, Vitest

`frontend/tests/`, against `lib/` and the components, with no network.

| File | What it proves |
| --- | --- |
| `format.test.ts` | money as `$42,500` with its currency; cents to dollars; percentages; dates; the timeline's relative times |
| `derive.test.ts` | board and column totals from a payload; the funnel bands in order; the twelve-month axis and its labels; late and overdue |
| `DataTable.test.tsx` | searching filters; clicking a heading sorts and shows the arrow; the empty state offers the New button |
| `FormDialog.test.tsx` | a 422 puts its message under the right field and leaves the others alone; a probability outside 0 to 100 shows the quiet note and still saves |
| `DeleteDialog.test.tsx` | the impact counts appear in the sentence and Delete is the Raspberry button |
| `Timeline.test.tsx` | newest first; the right dot colour per kind; an app-written entry has no edit or delete |
| `stageColours.test.ts` | one colour per stage everywhere, and the status pills |

### End to end, Playwright Test

`e2e/`, against the installed Google Chrome, its own ports and database, reseeded before each spec.
Every spec fails on any console error, which is how criterion 9's console check is kept honest.

| Spec | The criterion it proves |
| --- | --- |
| `01-first-run.spec.ts` | 1 — a clean database, one command, the dashboard with data on every screen |
| `02-records.spec.ts` | 2 — add, find, sort, edit and delete each record; the change survives a reload and a restart; the delete confirmation gives counts; an organization delete takes its contacts, deals and activities; `npm run seed` brings the sample data back |
| `03-refusals.spec.ts` | 3 — a record with no name, an invalid email and a second contact with an email already in use are each refused, naming the field |
| `04-links-and-status.spec.ts` | 4 — the status filter; each record's page shows what it is linked to; a status changes only when someone edits the contact |
| `05-pipeline.spec.ts` | 5 — a deal moved between columns changes stage and probability, both ways, Won and back; a card dropped higher in its own column is still there after a reload; the board and column totals move as it lands; a deal added in a column starts at that stage; late deals stand out |
| `06-timeline.spec.ts` | 6 — an activity and a stage change appear in the timeline newest first; a follow-up shows on the dashboard until it is marked done, with the overdue ones and those due in the next seven days |
| `07-dashboard.spec.ts` | 7 — the figures match the data; winning a deal and adding a follow-up change them; the charts cover the last twelve months to the current one |
| `08-brand.spec.ts` | 8 — the three fonts are loaded from the app; the page background, panels and stage colours are the tokens; the line chart, the bar chart and the funnel are all on the dashboard |
| `09-journey.spec.ts` | 9 — one pass through every section and every action above, in order, with the console clean throughout |

Criterion 9 is not finished by `09-journey.spec.ts`. The intent asks for a person to use the app end to
end in a real browser and check each screen by eye. The spec is the floor; Phase 11 is the criterion.
