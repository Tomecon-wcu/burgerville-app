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
  regulated) outcome shown as an *additional*, separately-labeled comparison point. The
  monopoly's own outcome fades in and is approached by its own growing connector arrow *from
  the previous season's own monopoly/regulated outcome* — see the nuanced note under Section 5
  before assuming all connector arrows near the monopoly dot were rejected; a specific,
  different kind was deliberately added back.
- **Part II (perfect competition) now shows a Firm Cost Diagram on BOTH the Market Info and
  Results pages** — one representative firm's short-run MC/ATC/AVC curves, at its actual land
  size, with the market price it faces drawn as a horizontal line at the *same pixel height* as
  the price on the market chart. Built as a standalone prototype first, iterated on with the
  professor's direct feedback, then integrated — and integration surfaced a real string of
  concrete bugs, each independently found and fixed (see Section 5 for the specifics of each):
  a chart-stability bug where a pure demand shock made static cost curves visibly appear to
  shift, a case where a pure fixed-cost change (no market-side shift at all) meant the firm
  chart never rendered anything, the "Watch the Adjustment" slider staying hidden in that same
  case even though the firm's own ATC curve had genuinely moved, the Results page never getting
  the feature at all on first integration (it's a separate, self-contained drawing function from
  the Market Info tab's — see the existing note on this below), and a layout bug once it did
  where the two Results-page charts overflowed and misaligned vertically. All confirmed fixed
  and tested; this feature is now in a solid, thoroughly-verified state on both pages.
- **The game now has a proper end state.** Once `game_state.current_season` advances past the
  last season with real data (all 16 seasons of `test-run-vf26` played through), the app shows
  a clear "The Game Has Ended" screen with a normal-looking "Season 16" tab a student can click
  into. Getting this fully working required removing an *older*, separate code path in
  `bootstrap()` that was silently intercepting the flow before the newer, more complete handling
  ever ran — see Section 5's note on this, since it's a good example of a class of bug worth
  watching for (an old fix looks broken not because it's wrong, but because something else gets
  there first).
- **Part II's Results page got two more targeted fixes**: the land-investment input now hard-caps
  at the student's credit limit as they type (previously only checked, with an alert, at submit
  time), and the "Optimal" column's stated-output/stated-marginal-cost values are now marked
  with an asterisk and a footnote clarifying these are derived from the student's *own* stated
  cattle/land, not an independent value — students get credit for correctly applying the
  production function to their own inputs, not for guessing a plausible-looking number.

**A real data-corruption bug was found and fixed** — not a display bug, but wrong values
actually written to the database by every completed Part III season: `equilibrium_history`
stored that season's *monopoly* outcome instead of the true competitive equilibrium, corrupting
the "baseline" reference for whichever season came next. Fixed going forward in
`run-current-season`; a one-time repair function (`repair-equilibrium-history`) corrects
already-corrupted rows for seasons run before the fix. **Confirmed and run on `test-run-vf26`
only** — if any Part III seasons have been run on `eco112-fall26`, the same repair still needs
running there too before trusting that instance's Part III chart/continuity data. This has not
been confirmed done as of this writing. See spec Section 18.7 for the full story.

**`test-run-vf26` testing is complete** — all 16 seasons (0–15) have been played through to the
end, including both Part III modules, and the game correctly reaches its "Game Has Ended" state.
A rollback procedure now also exists to reset the test run back to a *specific* earlier season
(rather than only a full wipe-and-reload) — see Section 7 — which surfaced an important schema
detail worth knowing: `game_state` tracks mutable baseline fields (`current_price`,
`current_quantity`, `num_other_firms`), not just `current_season`. Any future rollback or reset
work needs to restore all of these together, not just the season number, or the "next" season
would silently compute against a stale baseline.

**Not yet built: a guided instructor interface.** Right now, setting up each season means
writing SQL by hand. The professor raised starting this as **a new, separate chat within this
same project**, specifically so it would have its own room rather than competing with the large
amount of student-facing context already built up here — but that switch hasn't actually
happened yet as of this writing. Instead, a substantial amount of additional student-facing
polish work (the Firm Cost Diagram integration and its bug-fix trail, the game-over flow fix,
the two Part II Results-page fixes above) continued in this same conversation after the
instructor-interface intention was raised. Whoever picks this up next should treat the
instructor interface as the next major piece of work, but shouldn't assume a clean handoff
already happened — check Section 8 for the honest, current state of that transition.

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
         — get-market-info: current market state, secrecy-preserving shock hints, the true
           monopoly/regulated optimum for a completed Part III season (including which PART
           the *previous* season itself was, via `monopolyPart` — needed so the "previous
           outcome" label is correct even when the previous season was a different Part III
           module than the current one), and a graceful `{gameOver: true, lastSeasonNumber}`
           response instead of throwing when asked for a season past the end of the dataset
         — run-current-season: scores everyone, advances the game (instructor-only)
         — submit-prediction: upserts a submission + sends confirmation email; validates the
           submitting season matches game_state before checking part, so a stale browser tab
           gets a clear "please refresh" rather than a confusing mismatch error
         — repair-equilibrium-history: one-time-use, corrects already-corrupted Part III
           equilibrium_history rows from before the Section 18.7 fix (see spec for why this
           exists as a separate function rather than a one-off SQL script)
```

None of these five Edge Functions changed during the most recent round of work (the Firm Cost
Diagram integration, the game-over fix, and the two Part II Results fixes were all purely
client-side, in `Burgerville.html`) — worth confirming directly which layer(s) an upcoming
change actually touches before assuming an Edge Function redeploy is needed (see Section 7's
note on this).

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
| Rollback `test-run-vf26` to a specific season | `rollback_1_verify.sql`, `rollback_2_execute.sql`, `rollback_3_confirm.sql` — run in that order; see Section 7 |
| Edge Functions | `supabase/functions/{get-parts-overview,get-market-info,run-current-season,submit-prediction,repair-equilibrium-history}/index.ts` |
| Shared engine/scoring logic | `supabase/functions/_shared/*.ts` (Engine, MarketPrediction, MonopolyEngine, ScoringPartI/II/IIIProfit/IIIRegulation) |
| Firm Dashboard prototypes (reference, pre-integration) | `Burgerville_Firm_Dashboard_Prototype.html` (Part II), `Burgerville_Monopoly_Dashboard_Prototype.html` (Part III, both modules) — both now integrated directly into `Burgerville.html`; historical reference, not deployed anywhere |
| Firm Cost Diagram prototype (reference, pre-integration) | `Burgerville_Cost_Diagram_Prototype.html` — standalone, interactive (scenario picker, its own "Watch the Adjustment" slider) build used to iterate on the design with the professor before integrating into `Burgerville.html`; also historical reference now |
| Diagnostic queries | `diagnose_part_ii_firm_scoring.sql`, `diagnose_season_13_part_mismatch.sql` (read-only; the pattern to reach for when a symptom's real cause isn't obvious from the client alone) |
| Test suite (Node, run locally) | `test_*.js` — 35+ files, 380+ checks, covering auth, submission, resubmission, both dashboards, Part II/III display and scoring derivations, the equilibrium-aware chart logic, font-scaling regressions, the game-over end state, and the Firm Cost Diagram's layout/reveal/stability behavior on both Market Info and Results |
| Full project history & validation record | `Burgerville_Engine_Spec.md` |
| Original step-by-step deployment instructions | `Burgerville_Supabase_Deployment_Guide.md` |
| GitHub repo | `tomecon-wcu/burgerville-app`, served at `tomecon-wcu.github.io/burgerville-app/`, **Pages source: "Deploy from a branch"** (no custom `.github/workflows/` file exists — see the deployment-timeout note in Section 6 before suggesting any workflow-file edit) |
| Supabase project | `oxhojojzwvnfzsfpybyf` (org: Tomecon-wcu), instance ID `eco112-fall26` |
| Email | Resend, verified domain `funconomics.org` |

## 5. Key design decisions worth knowing before you change anything

- **`instance_id` is on every table**, even though only one instance (`eco112-fall26`) exists
  today. This is the seam for real multi-instance support later (many classes/instructors on
  one deployment) — don't remove it even though it looks redundant for a single class. **This
  seam is exactly what the instructor interface work (Section 8) will need to actually use** —
  right now nothing in the UI lets you pick or create an instance, even though the data model
  already supports it.
- **RLS is defense in depth, not the only line of defense.** Edge Functions re-validate things
  like season locking and part matching even where RLS would also catch it. Don't remove a
  server-side check just because "RLS already covers this" — the whole point is that both
  layers independently hold. The same philosophy now also applies client-side: the land
  investment field's credit limit is enforced live as the student types *and* re-checked at
  submit time — the second check isn't redundant, it's what still catches it if the first one
  is ever bypassed (a manipulated DOM, a stale page, etc.).
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
- **The land investment field's credit limit is a hard, live cap, not just a warning.** Typing
  one digit too many (e.g. $500,000 instead of $50,000) snaps the field straight back to the
  actual limit on every keystroke, with an explicit "Capped at your credit limit" message — a
  meaningfully stronger prevention than only checking (and alerting) once, at submit time, which
  is what existed before and still remains as a defense-in-depth backstop.
- **The Results page's "Optimal" column for stated output and stated marginal cost is
  asterisked**, with a footnote explaining these values are computed from the student's *own*
  stated cattle/land inputs (via `computeActualFirmOutcomeClient`), not an independent value —
  this is a check on whether they correctly applied the production function, not a number they
  could look up or invent. The footnote only renders when at least one row's Optimal value
  actually carries the asterisk (checked directly against the rendered string), so it doesn't
  show up on Parts I or III's results tables.
- **The Results page's chart and "correct answer" derivation are self-contained**, not reusing
  the Market Info tab's drawing functions or its secrecy-preserving Edge Function logic. This
  was a deliberate choice to avoid risking already-validated, shipped code — see spec Section
  17.3. **This has a real, repeatedly-confirmed maintenance cost, and it came up again
  concretely**: the Firm Cost Diagram was built and fully working on the Market Info tab, but
  the Results page simply never got it at all on first integration, since nothing in
  `drawResultsChart()` — a genuinely separate function with its own canvas setup — was touched.
  Had to be built again as its own function (`drawResultsFirmCostDiagram`), deliberately simpler
  than the Market Info version since Results always shows the fully-resolved state (no slider,
  no fade-in reveal needed). If you change one chart's design, check whether the other one needs
  the identical change — this has been missed and caught after the fact more than once now.
- **A chart-stability bug: a chart's own axis scale must never be recomputed per-frame from the
  currently-animating state.** The Firm Cost Diagram's x-axis (`qMax`) was originally
  recalculated every frame from the firm's *currently interpolated* output — which meant a pure
  demand shock (cost parameters completely unchanged) still made the MC/ATC/AVC curves visibly
  appear to shift, purely because the axis they were plotted against kept shrinking and growing
  underneath them as price animated toward its new value. Fixed by computing the axis domain
  once, from the full old-and-new range, exactly the same principle the market chart's own
  domain (`computeDomain`) already followed. If you add any new animated chart, this is the
  first thing to check: does the axis scale itself ever change mid-animation from something
  other than the fixed start/end states?
- **A pure fixed-cost change (e.g. land rent) produces NO market-side shift at all — and that's
  economically correct, not a bug** — fixed costs affect ATC but never `marginalCost` or
  `marketSupply`, since they don't factor into the profit-maximizing output decision. This
  exposed a real bug though: `draw()`'s no-shift branch returned early *before* ever reaching
  the call to draw the firm cost chart, so the chart simply never rendered anything in this
  case, even though the firm's own ATC curve had genuinely moved. Confirmed directly against
  Season 8's real dataset parameters (a 10% land-rent increase) before and after the fix.
- **The "Watch the Adjustment" slider's visibility is decided by TWO independent things, kept
  deliberately separate**: `detectShift`/`side` (does the *market* have something to animate)
  and a newer `detectFirmCostShift` (do the *firm's own cost parameters* — calf_price,
  land_rent, Other_land, A_tech, Alpha_land, Beta_cattle, interest — differ at all). The slider
  shows if either is true. Kept as two separate checks rather than folding the firm-cost check
  into `side` itself, specifically so the market chart's own arrows and gap indicator continue
  to correctly stay hidden when only the firm's costs moved, not the market.
- **The exact same fixed-inline-canvas-width bug recurred on the Results page** after being
  fixed once already on the Market Info tab's main chart. `drawResultsChart` and the newly-built
  `drawResultsFirmCostDiagram` were both setting `canvasEl.style.width = CW + 'px'` directly,
  which silently overrides the responsive CSS rule the same way the original bug did — harmless
  when a chart has a card entirely to itself, but it forces overflow the moment a second chart
  needs to share that space. If you ever see a canvas overflowing or misaligned within a
  flex-row layout in this app, check for this exact pattern first (`grep` for
  `canvasEl.style.width` — there should be none left anywhere in the file).
- **Two side-by-side charts need MATCHING header structure, or their canvases won't align
  vertically even with sizing correct.** The firm chart's "Representative Firm — Short-Run
  Costs" heading had no counterpart above the market chart, so the two canvases started at
  different Y positions purely because one flex column had extra text pushing its canvas down
  and the other didn't. Fixed by giving the market chart a matching header ("Market" /
  "Aggregate supply and demand for the whole market"), shown only when the firm chart is also
  present (`marketChartHeader` on Market Info, `resultsMarketChartHeader` on Results — both
  toggled alongside the existing firm-chart-visibility logic, not as a separate concern).
- **An older, no-longer-relevant code path can silently intercept a newer one — worth actively
  checking for, not just assuming the newest code is what's running.** The game-over screen
  looked broken (wrong message, no working season navigation) despite the newer, more complete
  handling for it being fully built and tested. The real cause: `bootstrap()` had its own older,
  separate early-return check for the same "game has ended" condition, written before the newer
  handling existed, that fired first and returned before `loadSeason()` — the function the newer
  code lived inside — was ever called. Removed the older code path entirely. If a feature that
  should work "just doesn't seem to," checking for an older code path handling the same
  condition is worth doing before assuming the newer code itself is wrong.
- **The Part III chart's monopoly dot is a separate, distinct thing from a "market-shift"
  connecting arrow — the distinction matters and has been gotten wrong in both directions.**
  Two different kinds of arrow have come up:
  - A connector from the *competitive* outcome to the *monopoly* outcome, within the same
    season — this was tried, then explicitly and deliberately removed. A monopoly's outcome
    doesn't transition from the competitive one; it's a separate choice being compared against
    a hypothetical. **Don't add this one back** without re-confirming that's actually wanted.
  - A connector from the *previous season's own* monopoly/regulated outcome to the *current
    season's own* monopoly/regulated outcome — genuinely different, added later, and kept. This
    shows how the monopoly's own choice changed as the underlying market shifted, the same way
    the market chart's black arrows already show the competitive equilibrium's own before/after.
    It fades in and grows in step with the "Watch the Adjustment" slider, exactly like the
    market-side arrows.

  If asked to "connect the two dots" again, the question to ask first is *which* two dots —
  competitive-to-monopoly (don't), or previous-monopoly-to-current-monopoly (already exists).
- **When comparing two Part III Regulation seasons specifically, the short-run competitive
  arrows and dot are deliberately suppressed** on both charts. Since price=ATC regulation *is*
  mathematically the long-run competitive outcome (at a much larger, long-run firm count — see
  the `longRunFirmCount`/long-run supply curve feature), the short-run market's own tiny
  movement (at the market's actual, much smaller firm count) isn't part of what's being
  compared in that specific case. Detected via `oldMonopolyOutcome.longRunFirmCount &&
  monopolyOutcome.longRunFirmCount` both being present — i.e., both the previous and current
  outcome being Regulation, not Profit. A Profit-to-Regulation or Profit-to-Profit transition
  still shows its short-run arrows normally; only Regulation-to-Regulation suppresses them.
- **A previous season's own Profit (unregulated) outcome, shown as historical context on a
  later Regulation season's chart, is deliberately labeled "Unregulated Monopoly Outcome"**, not
  just "Monopoly Outcome" — more precise once it's being held up against a regulated one, and
  accurate by definition (a profit-maximizing monopoly is unregulated). Getting this backwards
  (deriving the "previous" label from the *current* season's own part rather than the previous
  season's actual part) was a real, subtle bug — fixed by having `get-market-info` explicitly
  report which part the previous season was (`oldState.monopolyPart` / `baseline.monopolyPart`).
- **`equilibrium_history` must always store the true competitive equilibrium, never a monopoly
  or regulated outcome** — this was violated for a real stretch of the project (spec Section
  18.7) and caused genuine data corruption, not just a display glitch. If you ever touch
  `run-current-season`'s result-writing logic, re-read that section before changing what gets
  written there.
- **The Firm Cost Diagram (Part II) intentionally shares its price axis with the market chart**
  — the same dollar price lands at the identical pixel height on both, confirmed directly
  (pixel-level, not just visually) rather than assumed, on BOTH the Market Info and Results
  pages independently. `pMax` is computed once by the market chart and passed into the
  cost-diagram drawing function rather than each computing its own.
- **`game_state` tracks mutable baseline fields, not just `current_season`.** Discovered while
  building the season-rollback procedure (Section 7): `current_price`, `current_quantity`, and
  `num_other_firms` are the baseline the *next* season's calculations actually use, and they get
  overwritten every time a season runs. Resetting `current_season` alone without also restoring
  these three from the correct prior season's `equilibrium_history` row would silently leave the
  "next" season computing against a stale, wrong baseline — worth remembering for any future
  reset, rollback, or instance-management work, especially the instructor interface.

## 6. Known simplifications — flagged on purpose, not forgotten

- **Instructor auth is a shared secret header** (`X-Instructor-Secret`), checked by
  `run-current-season`. Fine for one instructor; needs a real per-instructor login system
  before this is used by more than one person managing their own class.
- **Adding a season is manual SQL** — insert a row into `season_parameters` before that
  season's deadline. There's no reminder system. **This is exactly the gap the instructor
  interface (Section 8) is meant to close.**
- **Adding a student to the roster is a two-step manual process**: they have to sign in once
  first (which creates their `auth.users` row), and only then can you look up their ID and add
  them to `participants` with a team assignment. No self-service team-joining exists.
- **Multi-instance support is schema-ready but not built out** — the seam (`instance_id`
  everywhere) exists, but there's no UI for creating a new instance, and the client hardcodes
  a single `INSTANCE_ID` constant rather than letting it vary per URL or per login. **Directly
  relevant to the instructor interface** — worth deciding early whether that interface should
  start exposing this seam (letting an instructor manage/select their own instance) or continue
  assuming a single hardcoded one for now.
- **GitHub Pages deployment can occasionally time out with no code-level cause — and this has
  now been directly confirmed, not just suspected.** A "Deploy to GitHub Pages" job sat at
  `deployment_in_progress` for 10+ minutes and was auto-cancelled by GitHub's own timeout,
  despite the build step itself succeeding cleanly in seconds — and it recurred multiple times
  in one day. A live web search confirmed a genuine, widespread GitHub Actions/Pages outage on
  that date (GitHub's own status updates showed degraded Actions performance starting mid-day,
  with Pages added to the affected list shortly after, mitigations rolling out over several
  hours). This repo uses the plain "Deploy from a branch" Pages source — there is no custom
  `.github/workflows/*.yml` file to tune a `timeout-minutes` value on, so generic AI-suggested
  fixes proposing to edit a workflow file don't apply here. If this happens again: check
  githubstatus.com directly before assuming it's something wrong with this specific repo or
  file — a "Re-run failed jobs" retry, or simply re-uploading the file again, has been the
  effective fix once GitHub's own incident clears.

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
A reusable script pattern (avoids re-typing this every season, and reports which season number
was actually just scored so you can confirm it matches what you expected):
```bash
RESPONSE=$(curl -s -X POST https://oxhojojzwvnfzsfpybyf.supabase.co/functions/v1/run-current-season \
  -H "Authorization: Bearer <anon key>" \
  -H "X-Instructor-Secret: <instructor secret>" \
  -H "Content-Type: application/json" \
  -d '{"instanceId": "eco112-fall26"}')
echo "$RESPONSE" | python3 -m json.tool 2>/dev/null || echo "$RESPONSE"
```

**If a `curl` command like the above gets stuck on a `dquote>` prompt instead of running:**
this is almost always macOS's "smart quotes" silently converting a straight `"` into a curly
one during copy-paste — visually near-identical in a terminal, but not recognized by the shell
as a real quote character, which throws off quote-matching. Press Ctrl+C to get unstuck, then
sidestep the problem entirely by writing the JSON body to a file first, or by saving the whole
command as a `.sh` script and running `bash script.sh` — since the file's own contents, once
downloaded, never pass through a copy-paste step, there's no quote-mangling opportunity at all:
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
writing, this has been run and confirmed on `test-run-vf26` only; **`eco112-fall26` has still
not been checked for this issue** and shouldn't be assumed clean if any Part III seasons have
run there. Worth doing before the live class reaches Part III, if it hasn't already.

**Roll `test-run-vf26` back to a specific season** (not a full wipe — preserves real submission
history for every season before the rollback point): run `rollback_1_verify.sql` first and
check its output looks like what you'd expect, then `rollback_2_execute.sql`, then
`rollback_3_confirm.sql`. Deletes all downstream data (submissions, results,
`equilibrium_history`, `firm_dashboard_state`) for every season from the target season onward,
and restores `game_state`'s baseline fields (`current_price`/`current_quantity`/
`num_other_firms`) from the prior season's own `equilibrium_history` row — not hardcoded values.
Currently written for a rollback to Season 8 specifically; adapt the season-number literals in
all three files for a different target season. See Section 5's note on why `game_state`'s
baseline fields matter here.

**Add a student to the roster** (after they've signed in once):
```sql
insert into participants (id, instance_id, student_id, screen_name, team_brand, email)
values ('<their auth UID, from Authentication → Users>', 'eco112-fall26', '<student id>', '<screen name>', '<TEAM>', '<their email>');
```

**Reset the test-run instance to a clean slate** (deletes everything tied to `test-run-vf26`
via cascade, then reloads the full real 16-season dataset fresh — use this for a full reset to
Season 0; use the rollback procedure above instead for resetting to a specific later season
while preserving earlier real submission history):
```sql
delete from instances where id = 'test-run-vf26';
```
then run the full contents of `load_full_test_run.sql`. You'll need to re-add any test
participants afterward (same roster process as above, just with `instance_id = 'test-run-vf26'`)
— deleting the instance removes their roster entries too.

**Deploy a code change to an Edge Function:** edit the file locally, then from the
`burgerville-edge` project folder: `supabase functions deploy --use-api`. Check the output
confirms all five functions redeployed with no errors.

**Deploy a client change:** edit `Burgerville.html` locally (keeping the real `SUPABASE_URL`/
`SUPABASE_ANON_KEY`/`INSTANCE_ID` values filled in), upload it to the GitHub repo, overwriting
the existing file. **Client-only changes need no Edge Function redeploy at all** — a nav-bar
rearrangement, the game-over-flow fix, the entire Firm Cost Diagram build-out and its bug-fix
trail, and the two Part II Results-page fixes were ALL purely client-side and needed only this
step; conflating "did I redeploy Edge Functions" with "did I upload the client" has caused
confusion before when only one of the two was actually needed. If a change touched both a `.ts`
file and `Burgerville.html`, both steps are needed; check which one(s) the actual diff touched
before assuming both are required.

## 8. What's actually next, right now

**The instructor interface is still the next major piece of work, but the "new chat" switch
the professor raised hasn't actually happened yet — be honest about this rather than assuming a
clean handoff already occurred.** The professor asked about starting this work in a new,
separate chat within this same project, specifically so it would have its own room rather than
competing with the large amount of student-facing context already here. That's still the right
plan. But after raising it, the conversation continued in this same chat with a substantial
amount of further student-side work (the entire Firm Cost Diagram integration and its full
bug-fix trail, the game-over flow fix, and the two Part II Results-page fixes above) — so if
you're reading this from a genuinely new chat, don't assume the student-facing side was already
frozen at the point the instructor-interface idea came up. This document is current as of all of
that additional work, so it should still be a reliable starting point either way.

The professor's original stated plan for the instructor interface: a guided form for entering a
season's parameters with a live preview against the engine, replacing the manual SQL in Section
7. Design direction was sketched early in spec Section 11 (written before the Supabase migration
— the shape still holds, the implementation details there are outdated and shouldn't be followed
literally). Worth deciding early, per Section 6 above, whether this interface should also start
exposing the existing `instance_id` multi-tenancy seam (letting an instructor manage or select
their own instance) rather than continuing to assume a single hardcoded one — and per Section
5's newest note, worth remembering that any season-management UI needs to handle `game_state`'s
mutable baseline fields correctly, not just `current_season`.

**Before trusting `eco112-fall26` (the live class) has clean Part III data:** run
`repair-equilibrium-history` against it too (Section 7 above) if any Part III seasons have been
run there — the bug in spec Section 18.7 wasn't instance-specific, it would have affected any
completed Part III season on any instance running the pre-fix code. Not yet confirmed done as
of this writing — worth checking directly rather than assuming either way.

**Worth keeping in mind while building the instructor interface:** the working pattern that
made the recent testing and integration phases productive was tracing a symptom to its *actual*
root cause before writing a fix, even when the first hypothesis looked right — a genuinely long
list of real bugs turned out to be one or two layers deeper than they first appeared (a CSS
class collision, a data-pipeline bug, a per-frame-recomputed axis scale, a branch that returned
before reaching code it needed to run, and a leftover older code path silently intercepting a
newer one all initially looked like "just" display issues). The same discipline will matter for
instructor-facing work: a season-setup form that *looks* right but writes slightly wrong
parameters would be a much quieter, harder-to-spot failure than almost anything on the student
side, since there's no student immediately noticing something's off.
