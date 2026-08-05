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

**Live and working**, for one real class (`eco112-fall26`):
- Students sign in with just an email (a one-time code, no password), submit predictions
  (large quantities typed as the full number with live comma-formatting and an adaptive
  magnitude hint — "= 14 billion lb." — not a "billions" shorthand), and can revise their
  submission any time before the deadline.
- All three parts are fully built and displaying consistently: Part I (market direction), Part
  II (perfect competition + firm decisions, with its own in-browser spreadsheet dashboard), and
  Part III's two modules — Profit (students maximize their own profit) and Regulation (students
  operate the monopoly to maximize welfare instead) — which share one dashboard by design but
  have separate submission forms and scoring.
- The Results page shows a real "Your Answer / Correct Answer" comparison with color coding for
  every part, not just Parts I/II — Part III's version required deriving what the student's own
  firm choice actually produced, compared against the true optimal outcome, since none of Part
  III's typed fields are individually scored the way Parts I/II's are.
- The Market Info chart shows the standard competitive-shift visualization (arrows, curves,
  the "Watch the Adjustment" slider) identically across every part, with Part III's monopoly (or
  regulated) outcome shown as an *additional*, separately-labeled comparison point — not a
  replacement for the competitive picture, and specifically not connected to it by any arrow
  implying a transition, since a monopoly's outcome is a deliberate choice being compared
  against a hypothetical, not something the market moves toward.

**A real data-corruption bug was found and fixed** — not a display bug, but wrong values
actually written to the database by every completed Part III season: `equilibrium_history`
stored that season's *monopoly* outcome instead of the true competitive equilibrium, corrupting
the "baseline" reference for whichever season came next. Fixed going forward in
`run-current-season`; a one-time repair function (`repair-equilibrium-history`) corrects
already-corrupted rows for seasons run before the fix. **Confirmed and run on `test-run-vf26`
only** — if any Part III seasons have been run on `eco112-fall26`, the same repair needs running
there too before trusting that instance's Part III chart/continuity data. See spec Section 18.7
for the full story and how it was confirmed fixed.

**A dedicated test instance (`test-run-vf26`) exists** for running through the game without
touching real student data, loaded with the actual, complete 16-season historical dataset. The
professor is actively testing this season-by-season, with concrete, specific feedback each
round — this is the current phase of work, not yet complete.

**Not yet built:** a guided instructor interface. Right now, setting up each season means
writing SQL by hand. This remains the single biggest quality-of-life gap (see Section 8).

## 3. Architecture at a glance

```
Browser (Burgerville.html, hosted on GitHub Pages)
   │
   ├─ Supabase Auth (email one-time code) — handles login
   │
   ├─ Direct Postgres queries (via supabase-js + Row Level Security)
   │     — reading your own results, the team leaderboard, submitting/resubmitting predictions,
   │       autosaving the Firm/Monopoly Dashboard's spreadsheet state
   │
   └─ Five Edge Functions (Deno/TypeScript, deployed to Supabase)
         — get-parts-overview: season/part navigation structure
         — get-market-info: current market state, secrecy-preserving shock hints, and (for
           Part III) the true monopoly/regulated optimum for a completed season
         — run-current-season: scores everyone, advances the game (instructor-only)
         — submit-prediction: upserts a submission + sends confirmation email; validates the
           submitting season matches game_state before checking part, so a stale browser tab
           gets a clear "please refresh" rather than a confusing mismatch error
         — repair-equilibrium-history: one-time-use, corrects already-corrupted Part III
           equilibrium_history rows from before the Section 18.7 fix (see spec for why this
           exists as a separate function rather than a one-off SQL script)
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
| Migrations applied to the live DB | `migration_allow_resubmission.sql`, `migration_add_firm_dashboard.sql`, `migration_add_part_ii_shortage_surplus.sql`, `migration_add_part_iii_profit_missing_scores.sql` |
| Full 16-season historical dataset loader | `load_full_test_run.sql` (loads `test-run-vf26` only — never run against the live instance) |
| Edge Functions | `supabase/functions/{get-parts-overview,get-market-info,run-current-season,submit-prediction,repair-equilibrium-history}/index.ts` |
| Shared engine/scoring logic | `supabase/functions/_shared/*.ts` (Engine, MarketPrediction, MonopolyEngine, ScoringPartI/II/IIIProfit/IIIRegulation) |
| Firm Dashboard prototypes (reference, pre-integration) | `Burgerville_Firm_Dashboard_Prototype.html` (Part II), `Burgerville_Monopoly_Dashboard_Prototype.html` (Part III, both modules) — both now integrated directly into `Burgerville.html`; these stand-alone files are historical reference, not deployed anywhere |
| Diagnostic queries | `diagnose_part_ii_firm_scoring.sql`, `diagnose_season_13_part_mismatch.sql` (read-only; the pattern to reach for when a symptom's real cause isn't obvious from the client alone) |
| Test suite (Node, run locally) | `test_*.js` — 19 files, 249+ checks, covering auth, submission, resubmission, both dashboards, Part II/III display and scoring derivations, the equilibrium-aware chart logic, and font-scaling regressions |
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
  17.3. **This has a real maintenance cost, confirmed in practice**: every chart-design change
  in Section 18.8 (arrow behavior, the monopoly dot, narrative text) had to be independently
  applied to both `draw()` and `drawResultsChart()`, since they're genuinely separate functions.
  If you change one, check whether the other needs the same change.
- **Part III now has a full "correct answer" column**, for both modules — this was still
  outstanding as of Section 17 but is done as of Section 18. Profit's version compares against
  the true profit-maximizing outcome; Regulation's compares the *outcome* of the student's own
  farms/cattle choice against the true price=ATC target, via derived rows, since none of
  Regulation's typed fields are individually scored (see spec Section 18.6 before assuming the
  two modules' Results-page logic is symmetric — it isn't, by design, because their underlying
  scoring isn't).
- **The Firm Dashboard (Part II) and Monopoly Dashboard (Part III, both modules) are one shared
  tool for Part III's two modules, by explicit instructor decision** — the underlying cost/
  production mechanics don't change based on which objective a student is optimizing for. Don't
  build a second dashboard variant for Regulation without confirming that decision has changed.
- **`equilibrium_history` must always store the true competitive equilibrium, never a monopoly
  or regulated outcome** — this was violated for a real stretch of the project (spec Section
  18.7) and caused genuine data corruption, not just a display glitch. If you ever touch
  `run-current-season`'s result-writing logic, re-read that section before changing what gets
  written there.
- **The Part III chart shows the monopoly/regulated outcome as a separate, unconnected
  comparison point — deliberately not linked to the competitive outcome by any arrow.** This was
  tried (a connecting arrow) and explicitly reversed per the professor's own framing: a monopoly
  doesn't transition from the competitive outcome, it's a separate choice being compared against
  a hypothetical. If asked to "show the relationship" between the two outcomes again, don't
  default to an arrow between them without re-confirming that's actually wanted.

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

**If a `curl` command like the above gets stuck on a `dquote>` prompt instead of running:**
this is almost always macOS's "smart quotes" silently converting a straight `"` into a curly
one during copy-paste — visually near-identical in a terminal, but not recognized by the shell
as a real quote character, which throws off quote-matching. Press Ctrl+C to get unstuck, then
sidestep the problem entirely by writing the JSON body to a file first:
```
cat > request.json << 'EOF'
{"instanceId": "eco112-fall26"}
EOF
curl -X POST https://oxhojojzwvnfzsfpybyf.supabase.co/functions/v1/run-current-season \
  -H "Authorization: Bearer <anon key>" \
  -H 'X-Instructor-Secret: <instructor secret>' \
  -H "Content-Type: application/json" \
  --data-binary @request.json
```
The quoted heredoc delimiter (`'EOF'`) means everything between the two `EOF` lines is treated
as pure literal text — no quote-interpretation, no risk of the same issue recurring.

**Repair corrupted Part III equilibrium data** (one-time use per instance, only needed for
seasons run before the Section 18.7 fix — safe to re-run, always recomputes from source
parameters):
```
curl -X POST https://oxhojojzwvnfzsfpybyf.supabase.co/functions/v1/repair-equilibrium-history \
  -H "Authorization: Bearer <anon key>" \
  -H 'X-Instructor-Secret: <instructor secret>' \
  -H "Content-Type: application/json" \
  -d '{"instanceId": "test-run-vf26"}'
```
Response lists each affected season's `before`/`after` price and quantity directly — check
these against what you'd independently expect before trusting the fix took effect. As of this
writing, this has been run and confirmed on `test-run-vf26` only; `eco112-fall26` has not been
checked for this issue and shouldn't be assumed clean if any Part III seasons have run there.

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

## 8. What's actually next, right now

**Immediate priority: finish testing `test-run-vf26`, season by season.** This is genuinely
in-progress, not finished — the professor is working through real seasons on the test instance
and surfacing specific, concrete issues each round (several genuine bugs found this way, not
just cosmetic feedback — see spec Section 18 throughout). If you're picking this up mid-stream,
the working pattern has been: professor finds something wrong on a real screen (usually with a
screenshot), the actual root cause gets traced before any fix is written — several times the
first hypothesis was wrong and the real cause was one or two layers deeper (Section 18.1's CSS
class collision, Section 18.7's data-pipeline bug both being seen initially as "just" display
issues). Don't fix the visible symptom without confirming where it actually originates.

**Before trusting `eco112-fall26` (the live class) has clean Part III data:** run
`repair-equilibrium-history` against it too (Section 7 above) if any Part III seasons have been
run there — the bug in spec Section 18.7 wasn't instance-specific, it would have affected any
completed Part III season on any instance running the pre-fix code.

**Longer-term: the instructor configuration screen.** The professor's original stated plan,
still not started — a guided form for entering a season's parameters with a live preview against
the engine, replacing the manual SQL in Section 7. Design direction sketched in spec Section 11
(written early, before the Supabase migration — the shape still holds, the implementation
details there are outdated). A natural, well-scoped project once the current testing phase
confirms everything else is solid.
