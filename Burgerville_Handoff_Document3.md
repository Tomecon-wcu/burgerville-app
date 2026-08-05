# Burgerville — Hand-off Document

**Purpose of this document:** get anyone (a TA, a hired developer, a future version of
yourself after a break, or a fresh AI assistant session with no memory of how this was built)
productive on this project in five minutes, not five hundred pages. For the full detailed
history of every design decision and every validation performed, see
`Burgerville_Engine_Spec.md` — this document is deliberately shorter and points into that one
where it matters.

---

## 1. What this is

Burgerville is a microeconomics serious game (a beef-market simulation) built for classroom
use. Students see market news each "season," submit a prediction (direction of shifts, price/
quantity estimates, and — in later parts — their own firm decisions), the instructor runs the
season to score everyone and reveal the outcome, and students see their results and team
standings. The game has three parts of increasing sophistication: Part I (market dynamics),
Part II (perfect competition, firm decisions), Part III (monopoly — profit-maximizing and
regulated variants).

## 2. Current status

**Live and working**, for one real class (`eco112-fall26`), as of this writing:
- Students sign in with just an email (a one-time code, no password), submit predictions
  (large quantities typed as the full number with live comma-formatting, not a "billions"
  shorthand — see Section 5), and can revise their submission any time before the deadline.
- Season 1 has been run and scored; Results screen shows individual breakdown (with a
  "Your Answer / Correct Answer" comparison and a market chart, for Parts I/II), team
  standings, and cumulative totals correctly.
- Confirmation emails go out automatically on submission.

**A real scoring bug was found and fixed** (quantity-estimate units mismatch — see spec
Section 17.2 for the full story). One student's Season 1 score in the live class was corrected
as a result; everyone else's was confirmed unaffected before anything was written.

**A dedicated test instance (`test-run-vf26`) exists** for running through the game without
touching real student data — loaded with the actual, complete 16-season historical dataset
(`load_full_test_run.sql`). Seasons 1–5 have been fully tested there and confirmed working;
6–15 (Part II and Part III) remain to be walked through.

**Not yet built:** a guided instructor interface. Right now, setting up each season (and
adding students to the roster) means writing SQL by hand in Supabase's SQL Editor. This is the
single biggest quality-of-life gap and the natural next project (see Section 8 below).

## 3. Architecture at a glance

```
Browser (Burgerville.html, hosted on GitHub Pages)
   │
   ├─ Supabase Auth (email one-time code) — handles login
   │
   ├─ Direct Postgres queries (via supabase-js + Row Level Security)
   │     — reading your own results, the team leaderboard, submitting/resubmitting predictions
   │
   └─ Four Edge Functions (Deno/TypeScript, deployed to Supabase)
         — get-parts-overview: season/part navigation structure
         — get-market-info: current market state + secrecy-preserving shock hints
         — run-current-season: scores everyone, advances the game (instructor-only)
         — submit-prediction: upserts a submission + sends confirmation email
```

**Why this split, briefly:** anything RLS can fully express as a simple row-ownership rule
(read your own data, write your own row) is a direct client query — no server code needed.
Anything requiring logic beyond that (hiding a secret value that RLS alone can't selectively
mask, running the scoring engine, sending an email with a provider API key that can't be
exposed to the browser) goes through an Edge Function. See spec Section 14 for the full
reasoning.

## 4. Where everything actually lives

| What | Where |
|---|---|
| Client (the whole app, live class) | `Burgerville.html` — one file, uploaded to GitHub Pages |
| Client (disposable test instance) | `Burgerville_TestRun.html` — identical except `INSTANCE_ID`, points at `test-run-vf26` |
| Database schema + RLS policies | `schema.sql` |
| Resubmission/deadline migration | `migration_allow_resubmission.sql` (already applied to the live DB) |
| Full 16-season historical dataset loader | `load_full_test_run.sql` (loads `test-run-vf26` only — never run against the live instance) |
| Edge Functions | `supabase/functions/{get-parts-overview,get-market-info,run-current-season,submit-prediction}/index.ts` |
| Shared engine/scoring logic | `supabase/functions/_shared/*.ts` (Engine, MarketPrediction, MonopolyEngine, ScoringPartI/II/IIIProfit/IIIRegulation) |
| Test suite (Node, run locally) | `test_*.js` — covers auth, submission, resubmission, the Results-page comparison table, and comma formatting |
| Full project history & validation record | `Burgerville_Engine_Spec.md` |
| Original step-by-step deployment instructions | `Burgerville_Supabase_Deployment_Guide.md` |
| GitHub repo | `tomecon-wcu/burgerville-app`, served at `tomecon-wcu.github.io/burgerville-app/` |
| Supabase project | `oxhojojzwvnfzsfpybyf` (org: Tomecon-wcu), instance ID `eco112-fall26` |
| Email | Resend, verified domain `funconomics.org` |

## 5. Key design decisions worth knowing before you change anything

- **`instance_id` is on every table**, even though only one instance (`eco112-fall26`) exists
  today. This is the seam for real multi-instance support later (many classes/instructors on
  one deployment) — don't remove it even though it looks redundant for a single class.
- **RLS is defense in depth, not the only line of defense.** Edge Functions re-validate things
  like season locking and part matching even where RLS would also catch it. Don't remove a
  server-side check just because "RLS already covers this" — the whole point is that both
  layers independently hold.
- **The pending season's real shock parameters (`Q_0`, `dQ_dP`, etc.) are never sent to the
  browser.** This is load-bearing for the game's pedagogy (students are meant to predict, not
  read the answer off the wire) and is enforced at the database level (RLS hides the row
  entirely) *and* the Edge Function level (only a rounded magnitude hint is computed and
  returned). If you ever add a new way to read season data, check both layers.
- **Login is a typed one-time code, not a clickable magic link**, specifically because
  university email security scanners pre-fetch and burn clickable links before the real user
  gets to them. Don't "simplify" this back to a link without re-reading spec Section 15.
- **Submissions use upsert (insert-or-update), not insert-only.** Students can change their
  mind and resubmit until the deadline; only the last write before the deadline counts. The
  deadline check (`season_is_open()`) deliberately runs as `SECURITY DEFINER` — this is
  required, not incidental, because the RLS policy that hides the current season's secret
  parameters would otherwise also hide the deadline from this check and make it silently fail
  open. See spec Section 15.1 if you touch this function.
- **The two originally-separate screens (Market Info, Results) are now one page** with a tab
  switcher, specifically so there's one shared login session instead of two independent ones.
- **Large quantities (quantity_estimate and its Part III equivalents) are entered as the full
  raw number** (e.g. `20900000000`), not a "billions" shorthand — deliberate, on pedagogical
  grounds (confronting the real magnitude matters). An earlier version had a genuine bug here
  (the form said "billion lb." but the scoring engine compared raw units) that zeroed out at
  least one real student's correct answer before it was caught and fixed — see spec Section
  17.2 before touching these fields' labels or format again.
- **The Results page's chart and "correct answer" derivation are self-contained**, not reusing
  the Market Info tab's drawing functions or its secrecy-preserving Edge Function logic. This
  was a deliberate choice to avoid risking already-validated, shipped code — see spec Section
  17.3. Part III doesn't have a "correct answer" column yet for the same reason (it needs
  different logic — an optimal outcome, not a direction prediction — not yet ported).

## 6. Known simplifications — flagged on purpose, not forgotten

- **Instructor auth is a shared secret header** (`X-Instructor-Secret`), checked by
  `run-current-season`. Fine for one instructor; needs a real per-instructor login system
  before this is used by more than one person managing their own class.
- **Adding a season is manual SQL** — insert a row into `season_parameters` before that
  season's deadline. There's no reminder system; if you forget, the market info page simply
  errors with "no season_parameters row" when the game tries to advance to it.
- **Adding a student to the roster is a two-step manual process**: they have to sign in once
  first (which creates their `auth.users` row), and only then can you look up their ID and add
  them to `participants` with a team assignment. No self-service team-joining exists.
- **Multi-instance support is schema-ready but not built out** — the seam (`instance_id`
  everywhere) exists, but there's no UI for creating a new instance, and the client hardcodes
  a single `INSTANCE_ID` constant rather than letting it vary per URL or per login.

## 7. Common operational tasks (quick reference)

**Add a new season:**
```sql
insert into season_parameters
  (instance_id, season_number, season_name, part, q_0, dq_dp, a_tech, alpha_land, beta_cattle,
   other_land, calf_price, land_rent, endowment, shock_description, deadline)
values
  ('eco112-fall26', <N>, '<name>', '<I|II|IIIProfit|IIIRegulation>', <Q_0>, <dQ_dP>, 800, 0.5, 0.5,
   30, <calf_price>, <land_rent>, 50000, '<news text>', '<YYYY-MM-DD HH:MM:00+00>');
```

**Run the current season** (scores everyone, advances the game):
```
curl -X POST https://oxhojojzwvnfzsfpybyf.supabase.co/functions/v1/run-current-season \
  -H "Authorization: Bearer <anon key>" \
  -H 'X-Instructor-Secret: <instructor secret>' \
  -H "Content-Type: application/json" \
  -d '{"instanceId": "eco112-fall26"}'
```

**Add a student to the roster** (after they've signed in once):
```sql
insert into participants (id, instance_id, student_id, screen_name, team_brand, email)
values ('<their auth UID, from Authentication → Users>', 'eco112-fall26', '<student id>', '<screen name>', '<TEAM>', '<their email>');
```

**Reset the test-run instance to a clean slate** (deletes everything tied to `test-run-vf26`
via cascade, then reloads the full real 16-season dataset fresh):
```sql
delete from instances where id = 'test-run-vf26';
```
then run the full contents of `load_full_test_run.sql`. You'll need to re-add any test
participants afterward (same roster process as above, just with `instance_id = 'test-run-vf26'`)
— deleting the instance removes their roster entries too.

**Deploy a code change to an Edge Function:** edit the file locally, then from the
`burgerville-edge` project folder: `supabase functions deploy --use-api`.

**Deploy a client change:** edit `Burgerville.html` locally (keeping the real `SUPABASE_URL`/
`SUPABASE_ANON_KEY`/`INSTANCE_ID` values filled in), upload it to the GitHub repo, overwriting
the existing file.

## 8. Suggested next step: instructor configuration screen

The professor's stated plan. This would replace the manual SQL in Section 7 above with a
guided form — the original design direction is sketched in spec Section 11 (written early in
the project, before the Supabase migration, so the concrete implementation details there are
outdated, but the *shape* of what it should do still holds): a form for entering a season's
shock parameters with a live preview against the engine before activating it, rather than
hand-writing `INSERT` statements. Building this against the current schema/Edge-Function
architecture is a natural, well-scoped next project.
