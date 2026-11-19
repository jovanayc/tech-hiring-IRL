# Tech Hiring IRL App: Engineering Documentation

Last updated Nov 18, 2026 (evening). Built by The Dev Difference (TDD) for the Tech Hiring IRL healthtech hiring event (Thu Nov 12, 2026, 5:30 to 7:30pm, Polsky Center, Chicago Booth; companies Nestera, Cofactor AI, CareLumi). It is now an events manager that can run further events: save a finished event, clear the console, import registrants and rehearse with fake data.

No passwords, candidate data or keys other than the public anon key appear in this document or in the public repo.

Related: EVENT_PLAYBOOK.md (marketing, emails, assets, timeline), EVENT_LOG.md (what happened and open items), RUN_OF_SHOW.md (the night).

## 0. What this software is and who uses it

**What it is.** The Tech Hiring IRL app is the event operations tool for The Dev Difference's in-person hiring events. At an event, candidates rotate through small groups, each led by a hiring company's rep. The software checks everyone in, builds the groups and rounds, runs the timer and room screen, collects an exit survey where each person ranks who they want to keep talking to, and then turns those rankings into a private follow-up page for each company. After the event it keeps a permanent record (groups, rankings, link tracking, meeting requests) and can be cleared and reused for the next event.

**What it is not.** It is not a public product or a general applicant tracking system. It holds one event at a time, plus saved copies of finished events. The Dev Difference's AI interviewer (Buddy) and assessments live elsewhere; this app only stores assessment write-ups as internal notes.

**Who uses it, and for what**

| Who | Where they use it | What they do |
|---|---|---|
| The Dev Difference organizers (founders and event leads) | Team console (password) | Set up companies and prompts, add or import registrants, run check-in and rounds, push and close the exit survey, read results, send company pages, review link activity, save the event and start the next one |
| Stage manager and event volunteers | Team console and the room screen | Check people in at the desk, put the room screen on the display, help with late arrivals and the exit table |
| Candidates (attendees) | Attendee page on their phone | Sign in with their registered email, see their own group and station, keep private notes, take the exit survey |
| Company reps at the event | Attendee page and the exit survey | See their group, rank the candidates they met in the exit survey |
| Company hiring teams (for example Nestera, Cofactor AI, CareLumi) | Private company page link | After the event, see the attendees ranked by their own team, reorder, add comments and request next meetings. They never see assessment scores or integrity details |
| Engineers | The repos, SQL files and this document | Deploy, maintain and extend the app (sections 1 to 10 below) |

**Lifecycle of an event.** Set up companies and registrants, rehearse with the test event, check in and run rounds on the night, push the exit survey, review results and send company pages, watch link activity, then save the event, download a backup and start a new event.

## 1. Architecture at a glance

- One vanilla JavaScript app (no framework, no bundler) and a Supabase backend (Postgres plus PostgREST). The browser talks to Supabase with plain `fetch`. There is no server of our own.
- Single source `src/app.js` (plus `app.css`, `engine.js`, `supa.js`) is built by `src/build.py` into separate pages. Team-only code sits between `/*TEAM-START*/` and `/*TEAM-END*/` markers and is stripped from the public build.
- The console is a password-gated view inside the same app. The password is checked inside the database: a SHA-256 fingerprint is stored in `event_secrets`, and every private read and write goes through a security-definer function that checks it.
- Data is a key-value store: table `public.event_docs(path text primary key, data jsonb, updated_at timestamptz)`. A path such as `people/<id>` is a document and the app treats `people/` as a collection.
- `supa.js` implements a small Firestore-like layer (`doc`, `collection`, `onSnapshot`, `set`, `update`, `delete`) on top of PostgREST. It polls every 2.5 seconds (12 seconds when the tab is hidden), merges public rows with `team_read` rows, and notifies subscribers only when something changed. Writes update the local cache first, then call an RPC.

## 2. Pages and repos

| Page | Audience | File | Where |
|---|---|---|---|
| Attendee page | Candidates and company reps | `index.html` | GitHub Pages, public repo |
| Exit survey | Everyone, via QR | `survey.html`, `survey-qr.html` | Public repo |
| Company page | One private link per company | `shortlist.html#<token>` | Public repo (token is the secret) |
| Team console | TDD team only | `console.html` | Private repo, and the "Tech Hiring IRL Team Console" Claude artifact |

- Public repo: `jovanayc/tech-hiring-IRL` (GitHub has moved it; the old URL redirects). Contains built attendee, survey and company pages only. No PII.
- Private repo: `tech-hiring-irl-internal-console`. Contains `console.html`, the source in `src/`, every SQL file, and the docs. Candidate-loading SQL contains PII, so it never goes in the public repo.
- Downloads (CSV and JSON) work only when `console.html` is opened as a file in Chrome or Safari. The Claude artifact viewer blocks downloads.

## 3. Data model (paths in `event_docs`)

Public (readable by anyone with the anon key through the table policy):

| Path | Content |
|---|---|
| `people/<id>` | first, last, type (`candidate`, `rep`, `guest`), company (reps), color (reps), checkedIn, left, linkedin, sample (test data flag) |
| `rounds/r<n>` | n, at, prompt, companies info, stations `[{rep, cands[]}]`, sample flag for test rounds |
| `state/event` | round, endsAt, ended, surveyOpen, surveyClosed, surveyClosesAt, name, date |
| `state/survey` | Editable exit survey questions (title, help, helpRep, helpCand, rating, q1, q2, extra[]). Survives a new event. |

Private (only through RPCs):

| Path | Content |
|---|---|
| `private/<personId>` | email, phone, linkedin, portfolio, title, company (candidate's employer), sourced, notes, roles, level, status, assessment fields, integrity (internal only), seed, score |
| `private/config` | companies (name, wants, about, hiring, site), minutes, prompts, goals. Cleared of companies by Start a new event. |
| `private/links` | company page links: `{token: {company, ev, closesAt, closed}}`. `ev` is `''` for the live event or the slug of the saved event the link shows. Kept when a new event starts. |
| `private/shortlists` | legacy map of token to company name, used only for links not yet in `private/links` (moved into `private/links` when the event is saved and cleared) |
| `private/slr-<slug>` | a company's edits on its page for the live event: order, comments (`{candidateId: [{text, at, company}]}`, append only), notes (latest comment text, kept for old readers), meet, meetAt, meetEst, at |
| `private/aslr-<event>-<slug>` | the same edits made on a link that shows a saved event. Kept when a new event starts. |
| `private/alog-<event>-...` | visits (`sl-...`) and meeting requests (`meet-...`) made on a saved-event link after it was saved. Kept when a new event starts. |
| `private/shortlist-absent` | ids and per-company hides for the "wasn't able to make it" section |
| `survey/<personId>` | type, picks (ordered ids), none, rating, thoughts, comment, answers (extra questions), at |
| `notes/<personId>` | an attendee's running notes |
| `log/*` | activity rows: `signin`, `fail`, `open`, `sl_visit`, `meet_request`, `meet_cancel` (capped at 5000 rows) |
| `archive/<slug>` | one saved event: name, date, savedAt, counts, and every doc as `docs{path: data}` |

`supa.js` `isPrivate` treats `private/`, `survey/`, `notes/` and `log/` as private. `archive/` is never returned by `team_read` or the public policy; it is read only through `list_archives` and `read_archive`.

## 4. Database functions (RPC)

All are `security definer`, `search_path = public`. Team functions check the password fingerprint with `team_ok(pass)`.

| Function | Who | Purpose |
|---|---|---|
| `team_ok`, `team_read(pass)`, `team_write(pass, ops)` | Console | Read private rows, apply `set`/`delete` ops |
| `check_email(person, email)` | Attendee | Email sign-in. Saves the typed email if none is on file |
| `log_event(payload)` | Attendee | Sign-in, failure, open rows |
| `submit_survey(person, payload)` | Attendee, exit table | Must be checked in. Refuses when the survey is closed (`surveyClosed` or `surveyClosesAt` passed) |
| `submit_note(person, payload)` | Attendee | Must be checked in |
| `company_shortlist(token)` | Company page | Ranked candidates, rank labels, absent section, saved order, notes, meetings. Reads the live event or, for a link tied to a saved event, the saved event (through `ed(ev)`). Refuses closed or expired links with "This link is closed." |
| `save_shortlist(token, payload)` | Company page | Saves order and meetings, and appends a comment when the payload has `add: {candidateId: text}` (server stamps the time and company; comments cannot be edited). Writes `log/meet-*` rows for requests and cancels |
| `log_shortlist(token, visit, payload)` | Company page | Visit tracking (opens, seconds, clicks, device) |
| `company_rankings(token)` | Team | Rankings for a company (same event rules as the page) |
| `ed(ev)`, `link_get`, `link_slr`, `link_slrpath`, `link_logpath`, `adopt_archive_links(slug)` | Internal | Helpers from `add-persistent-links.sql`: `ed` returns the rows of the live event or a saved event, `link_get` checks a token (unknown, closed, expired), `adopt_archive_links` points links at the saved event and sets their close date. Not callable by the public. |
| `archive_event(pass, slug, label, event_date)` | Console | Snapshots every row except `archive/*` into `archive/<slug>`, returns counts |
| `list_archives(pass)`, `read_archive(pass, slug)` | Console | List saved events (no data), read one in full |
| `reset_event(pass, slug)` | Console | Refuses unless `archive/<slug>` exists. First calls `adopt_archive_links` and copies post-save company edits and visits to `aslr-`/`alog-` rows. Then deletes `people/`, `rounds/`, `state/` (except `state/survey`), `survey/`, `notes/`, `log/` and `private/` (except `private/config`, `private/links`, `aslr-` and `alog-` rows). Keeps `archive/*` and `event_secrets` |

## 5. Setup, SQL order and deploy

1. Run `supabase-setup.sql` once (table, RLS policy, password fingerprint, base functions).
2. Run the add-on files in this order: `add-notes` → `add-email-login` → `add-activity-log` → `add-company-shortlists` → `add-link-tracking` → `add-shortlist-ranking` → `add-absent-section` → `add-next-meeting` → `add-rank-labels` → `add-meeting-times` → `add-event-archive` → `add-survey-controls` → `add-persistent-links` → `add-company-comments`. Each `create or replace` supersedes earlier versions of the same function (for example `company_shortlist`, `submit_survey`, `reset_event`; `add-persistent-links` replaces the company page functions and `reset_event` again, and on its first run re-opens the links of events you already saved and cleared). All are safe to run again.
3. Data loaders (`load-approved-candidates`, `load-company-reps`, `add-people-nov12`, `add-assessment-writeups`, `update-three-profiles`) contain PII. Private repo only.
4. Build: `python3 src/build.py supabase <SUPABASE_URL> <ANON_KEY> out`. Output: `index.html`, `survey.html`, `console.html`, `shortlist.html`.
5. Publish: public repo (GitHub Pages deploys in about a minute, browser cache up to 10 minutes), private repo, and republish the console artifact after any HTML change.
6. Rotating the console password: update the `team_pass_sha256` row in `event_secrets` with the new SHA-256 hex digest.

## 6. Team console

The console opens with the password and has a left sidebar. The header shows "The {Dev} Difference | IRL Events" and "IRL Events Manager • Team Console". The event name and date appear as an H1 at the top once set. Sidebar items, in order:

1. **Roster and check-in.** Check people in (small green "Here" tag), edit full profiles, mark left early or back, delete. Shows four counts (candidates here, reps here, rounds sent, exit surveys in) on this tab and Groups only. "Add a person" button reveals the full form. "Import registrants from a CSV" accepts a file or pasted text, previews rows (adds, skips duplicates by name or email, flags problems), downloads a template, auto-adds unknown rep companies to Companies. "Paste a list of candidates" is a quick one-per-line import.
2. **Groups.** Show groups builds a preview using `engine.js` (matches roles to company wants, avoids repeat meetings). Reshuffle, tap a person then a station to move, Send round N to phones, timer (start, add or take off minutes, clear), End round, Undo round, Reopen round, late arrivals added to the best open station.
3. **Companies and prompts.** Company list (wants, one-line description, hiring, site), Remove all companies, round minutes and prompts. Prompts alternate by round.
4. **Survey results.** Per company: ranking by team member and a merged ranking (each candidate's best rank across the company's reps, ties by average rank, then last name). Ratings, written feedback (including extra questions) and attendee notes.
5. **Exit survey.** Push to phones (sets `surveyOpen`, closes 48 hours later by default), change the close time, add 24 or 48 hours, no automatic close, Close now, reopen. Edit the questions (headline with `{name}`, instructions, rating, two open questions, extra open questions, restore defaults). The database refuses late answers.
6. **Link activity.** "Which event?" picker (live now or any saved event, filtered by year, month or day; saved events render read-only). Company page links table (company, event it shows, close date, status, Close or Reopen per link) with Close all company links (two taps) and Reopen all (30 days). Collapsible sections with dates and times: company rankings and notes (original vs saved order, moves, meeting requests), meeting request history, company link visits, attendee sign-ins, failed sign-ins and devices.
7. **Events and test data.** Name and date; Save to the database; download everything as JSON and six CSVs (people and profiles, groups by round, surveys, notes, link tracking and sign-ins, company rankings and meetings); saved events list with read-only people and companies tables; Start a new event (pick the saved event, enter the next name and date, type `START NEW EVENT`); Load test event and Remove all test data.
8. **Screens.** Room screen, Exit survey table, Attendee page. Exit survey table and Attendee page log the console out. The Room screen has a faded "Back to console" button.
9. **Tutorial.** Collapsible walkthrough for the team, kept in sync with the features in this document.
10. **Log out**, then a link to thedevdifference.com.

Sessions: the console logs itself out after 5 minutes with no click, key, touch or scroll (checked every 10 seconds and when a sleeping laptop wakes). Browser Back returns to the console, which asks for the password if logged out.

### Test event rules
Load test event creates tagged (`sample: true`) reps, "(test)" candidates, two rounds, surveys, link activity and sign-ins. It only loads into an empty event. Remove all test data deletes only tagged rows.

## 7. Attendee, survey and company pages

- Attendees sign in with first name, last name and the registered email. A match checks them in. Failures are logged. Attendees see only their own group, never where others are. LinkedIn links are tappable for their group.
- Notes: attendees keep one running note, saved automatically and emailed back by the team afterward.
- Exit survey: shown on the phone only when `surveyOpen` is on and not closed. After close the card disappears and the page says it is closed. Answers already in are kept.
- Company pages: each company gets a private token link. The page lists attendees in the company's own ranked order with rank labels ("Your team ranked this candidate: Luis #1 · Brian #2"), headline, experience, LinkedIn and portfolio, reorder arrows, comments (see below), "Set up next meeting" with a confirmation, and a Cancel meeting request button. A "Completed assessment but wasn't able to make it" section lists absent candidates. Internal data (scores, integrity, emails, phones) never appears. External labels are Strong, Good, Weak. The Dev Difference is not shown as an interviewing company. Visits are tracked per link; email link previews can inflate opens, and a device is not a person.
- Comments: a company adds comments on a candidate; each is saved with the company name and a server timestamp and cannot be edited. "Add another comment" opens a new box. Old single notes appear as the first comment. A company page only ever returns that company's own comments (they live in that company's `slr` row). The team sees all of them, grouped by company with timestamps, on the candidate's profile in the Roster tab (read only) and in Link activity, and in the companies CSV.
- Company links do not close when an event is saved or a new event starts. A link keeps showing the event it was made for, using the saved copy, and companies can still reorder, add notes and request meetings there. It closes only when the team closes it (Link activity) or one month after the event date (`closesAt`). Reopen gives a link 30 more days. A closed link shows "This link is closed. Reply to the email you got from us and we will reopen it." New links for a new event are added to `private/links` (or `private/shortlists`) with `ev` empty, and are pointed at the event when it is saved and cleared.

## 8. Privacy and security model

- Anonymous clients can read only `people/`, `rounds/`, `state/` (names, groups, round status, survey questions). Everything else needs the password or a company token through a security-definer RPC.
- Integrity details and assessment raw scores are internal only.
- The console password is never in code or docs. Rate limiting is not implemented, so use a strong password and rotate it when team membership changes.
- Downloaded JSON and CSV files contain emails and phone numbers. Treat them as confidential.
- The public repo must never contain candidate data or SQL with candidate data.

## 9. Testing

- A local mock server (`@electric-sql/pglite` Postgres behind a small Node HTTP server on port 54399) loads `supabase-setup.sql` plus every add-on file and mimics the PostgREST routes. Build against it with `python3 build.py supabase http://localhost:54399 test-anon-key <dir>`.
- UI tests use Playwright with Chrome (`--no-sandbox --disable-web-security`). They cover the console flows (roster, import, groups, surveys, events, link activity, idle log out), the company page, and survey closing at the database.
- When adding an RPC, add it to the mock server's route list or the UI will see "not found".

## 10. Known caveats and open items

- Run `add-company-comments.sql` after `add-persistent-links.sql`, or comments written on the new company page will not be saved and existing comments will not show.
- Run `add-persistent-links.sql` (after the two files below) so company links survive a new event. Until it runs, links from a cleared event stay closed.
- Run `add-event-archive.sql` and `add-survey-controls.sql` before using the Events and Exit survey features fully. Without them, saving fails and late survey answers are blocked only on the page, not in the database.
- Meeting request times before `add-meeting-times.sql` was run are estimates (labeled "by"), because earlier rows stored only the last save time.
- Test people from earlier testing may still exist until you clear the event. Rana's name needs cleanup in the Nov 12 data.
- Cofactor's "keep talking" tag rule is undecided. Nestera and CareLumi rankings emails and attendee notes emails are still to send.
- No automated test suite lives in the repo yet; the checks above are scripts kept outside it.
- Keep the Tutorial tab in `app.js` (`TUTORIAL`) and this document updated together whenever the app changes.
