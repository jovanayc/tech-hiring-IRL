# Tech Hiring IRL App: Documentation

Built for the Tech Hiring IRL healthtech hiring event (Thu Nov 12, 2026, Polsky Center, Chicago Booth), hosted by The Dev Difference (TDD). Companies: CareLumi, Nestera, Cofactor AI.

No passwords or candidate data are in this document.

For the full event playbook (marketing, emails, assets, timeline) see EVENT_PLAYBOOK.md. For the night itself see RUN_OF_SHOW.md.

## 1. What it is

One small vanilla-JS app (no framework) with a Supabase (Postgres/PostgREST) backend. A single source (`src/app.js`) is built into several pages by `src/build.py`:

| Page | Audience | File | URL |
|---|---|---|---|
| Attendee page | Candidates and company reps at the event | `index.html` | https://jovanayc.github.io/tech-hiring-IRL/ |
| Exit survey | Everyone, via QR | `survey.html`, `survey-qr.html` | .../survey.html, .../survey-qr.html |
| Company page | One private link per company | `shortlist.html#<token>` | https://jovanayc.github.io/tech-hiring-IRL/shortlist.html#TOKEN |
| Team console | TDD team only, password protected | `console.html` | Private repo, and the "Tech Hiring IRL Team Console" artifact in Claude |

Team-only code is stripped from the public build between `/*TEAM-START*/` and `/*TEAM-END*/` markers.

## 2. Attendee experience

- Sign in with first name, last name and the email used to register. A match checks the person in. A non-match is logged as a failed attempt.
- Candidates see their own rotation schedule (rounds and tables) and who they meet. Attendees never see where other people are.
- Company reps see their candidates and can rank them in the exit survey.
- Attendees can leave private notes on people they meet (`submit_note`). Notes can be emailed to the attendee after the event.
- Exit survey: ranking of candidates (company reps) and event feedback. Requires the person to be checked in. Also available standalone through the QR page.
- The reminder switch `surveyOpen` controls whether the survey shows inside the main page. The standalone survey page is always available.

## 3. Team console

Password protected (hash stored in the database, password is never in code). Tabs:

- Check-in and roster: mark people in, edit profiles (title, links, experience, stack).
- Rounds: the rotation (rounds and table assignments).
- Survey: per company, a "By team member" table of each rep's ranking, plus the merged ranking. The merged rule is each candidate's best (lowest) rank across reps, ties broken by average rank, then last name.
- Notes: everyone's private notes.
- Activity: sign-in summary, sign-in log with phone vs laptop, failed attempts (a sign that the link reached unregistered people), devices per person (one person on several devices suggests a shared login), company link activity, company rankings and notes.
- Profiles hold assessment write-ups and integrity details that are internal only. Scores and integrity details are never shown to companies. Public labels are Strong, Good, Weak.

## 4. Company pages

Each company gets a private link containing a token. The page shows only candidates who attended, in the order that company's team ranked them, with:

- rank badge, name, headline, experience and stack, LinkedIn and portfolio links
- a line like "Your team ranked this candidate: Luis #1 · Brian #2"
- a green "Set up next meeting" button (tells TDD to reach out to schedule)
- up and down arrows to reorder, and a notes box; everything autosaves
- a section "Completed assessment but wasn't able to make it to the event" (hidden per company when needed)

Not included: assessment scores, integrity info, emails or phone numbers. The Dev Difference is not an interviewing company and is excluded.

Tracking per link (team-only): opens, time on page, clicks, revisit count, device type. Email link previews can inflate opens, and a device is not a person.

## 5. Backend (Supabase)

Table `public.event_docs(path text primary key, data jsonb, updated_at)`.

- Anonymous reads are allowed only on `people/%`, `rounds/%`, `state/%`.
- Everything private (`private/`, `survey/`, `notes/`, `log/`) is reachable only through security-definer RPCs.

| RPC | Purpose |
|---|---|
| `team_ok`, `team_read(pass)`, `team_write` | Console access to private data |
| `check_email` | Email sign-in match |
| `log_event` | Sign-ins, failures, opens |
| `submit_survey`, `submit_note` | Attendee writes (must be checked in) |
| `company_shortlist(token)` | Company page payload (ranked list, absent section, rank labels) |
| `save_shortlist(token, payload)` | Company edits: order, notes, meet |
| `log_shortlist(token, visit, payload)` | Link tracking |
| `company_rankings(token)` | Rankings for the team |

Key paths: `private/shortlists` (tokens), `private/slr-<slug>` (a company's edits), `private/shortlist-absent` (absent list and hides), `log/*` (activity).

## 6. Setup and deploy

1. Run `supabase-setup.sql` once (base schema).
2. Run the add-on SQL files in this order: `add-notes` → `add-email-login` → `add-activity-log` → `add-company-shortlists` → `add-shortlist-ranking` → `add-absent-section` → `add-next-meeting` → `add-rank-labels`. `add-link-tracking` goes after shortlists. Each `create or replace` supersedes the earlier version of `company_shortlist`.
3. Data loaders: `load-approved-candidates.sql`, `load-company-reps.sql`, `add-people-nov12.sql`, `add-assessment-writeups.sql`, `update-three-profiles.sql`. These contain candidate PII, so keep them in the private repo only.
4. Build: `python3 src/build.py supabase <SUPABASE_URL> <ANON_KEY> out`.
5. Publish: public repo (GitHub Pages, about a minute to deploy, up to 10 minutes of browser cache), private repo, and republish the console artifact after any change to the HTML.

## 7. Repos

- Public: `jovanayc/tech-hiring-IRL` (built attendee, survey and company pages, no PII).
- Private: `tech-hiring-irl-internal-console` (console, source in `src/`, all SQL, run of show).

## 8. Known caveats

- Test people (for example Jane Kelly, Anna Marie (Test)) may still exist in the database.
- Rana's name is inconsistent across records and needs cleanup.
- Console password is shared by the team out of band; rotate it after the event.
