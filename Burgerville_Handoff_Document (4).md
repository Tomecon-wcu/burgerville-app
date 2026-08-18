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
already-corrupted rows for seasons run before the fix. **Confirmed and run on both
`test-run-vf26` and `eco112-fall26`** — the live class had no completed Part III seasons at the
time of that check, so there was nothing for the bug to have touched there; worth re-running once
it actually reaches and completes one, since that was a point-in-time result, not a standing
guarantee. See spec Section 18.7 for the full story.

**`test-run-vf26` testing is complete** — all 16 seasons (0–15) have been played through to the
end, including both Part III modules, and the game correctly reaches its "Game Has Ended" state.
A rollback procedure now also exists to reset the test run back to a *specific* earlier season
(rather than only a full wipe-and-reload) — see Section 7 — which surfaced an important schema
detail worth knowing: `game_state` tracks mutable baseline fields (`current_price`,
`current_quantity`, `num_other_firms`), not just `current_season`. Any future rollback or reset
work needs to restore all of these together, not just the season number, or the "next" season
would silently compute against a stale baseline.

**Instructor interface: substantial progress, now consolidated back into this chat.** The
professor's earlier plan to build this in a separate chat (still referenced below as "Section 8"
context) did happen — that chat produced a working `instructor.html` (registration, roster
upload/single-add, season cloning + editing), two new service-role Edge Functions
(`bulk-upload-roster`, `clone-season-template`), and two schema migrations (an `instructors`
table, `owner_id` and class-metadata columns on `instances`, ownership-scoped RLS policies), all
verified by 138 checks of its own. That track then hit a real dependency boundary — its
season-wizard guardrail and its planned "Run Season" control both needed the actual, validated
engine code and `run-current-season` itself, which that chat had only ever seen in fragments —
and the professor made an explicit call to consolidate the remaining instructor-side work into
this chat rather than keep passing files back and forth. The first integration point is done:
`run-current-season`'s auth was retired from a single shared secret (`X-Instructor-Secret`) to
the same per-instructor ownership check the other two instructor-side functions already used, and
`instructor.html`'s dashboard now has a real, working "Run Season →" button per class. See
Section 5 for the specifics — and, for what came after this point (the instructor track went
considerably further still), the paragraph immediately below and
`Burgerville_Instructor_Handoff_Document.md`. **The old `X-Instructor-Secret`-based curl workflow
for running a season no longer works** — see Section 7's updated instructions.

**The instructor track has since gone considerably further than the paragraph above describes**
— it's no longer accurate to call it "still the next major piece of work" (Section 8 below has
been rewritten to reflect this). `INSTRUCTOR_SECRET` has been fully retired project-wide, not
just for `run-current-season`; the equilibrium-repair function was also retrofitted to the same
per-instructor auth; a results-download view, a season comparison panel, an equilibrium
guardrail, and Part III endowment-locking were all built and shipped. None of that is duplicated
here — see `Burgerville_Instructor_Handoff_Document.md` for the full, current record of that
track. This paragraph stays only to explain why the "still the next major piece of work" framing
below it is now outdated, not as the current status itself.

**The prediction form is now a persistent sidebar, visible on both Market Info and the
Dashboard simultaneously — not confined to one tab.** Previously, students had to do their
calculations on the Firm/Monopoly Dashboard, then switch tabs to Market Info to actually type
in their answers, losing sight of their own work in the process. The form (`#predictionFormSidebar`)
is now a single, shared element that gets dynamically re-parented to sit alongside whichever
tab is actually showing (`dockPredictionSidebar`), sticky-positioned so it stays in view while
scrolling through either the charts or the dashboard's spreadsheet below it — never duplicated,
so there's exactly one source of truth for its state regardless of which tab last touched it.
Getting the width right on the Dashboard side surfaced a genuine, subtle CSS gotcha worth
knowing before touching this again — see Section 5. The sidebar also carries two buttons for
toggling between Market Info and the Dashboard directly, without needing the top-level tab bar.
Two related fixes shipped alongside this: the same "sidebar overlapping the spreadsheet" problem
also existed on the dashboard's own layout (spreadsheet now always stays full-width, chart and
sidebar share a row below it), and Part III's demand/supply prediction sliders — previously
hidden entirely on the reasoning that "the prediction happens through the form" — are back,
since the form has always also asked students to estimate the *competitive* market outcome the
monopoly replaced, which is exactly what those sliders are for.

**Three real, confirmed scoring bugs were found and fixed in Part III's server-side scoring
(`ScoringPartIIIProfit.ts`, `ScoringPartIIIRegulation.ts`), not just tolerance misses.** All
three were traced to their actual root cause with independent, from-scratch verification before
being fixed — see Section 5 for the specifics of each: a marginal-revenue formula that had
already been corrected in the client-side display code but never carried back into the
server-side scoring function that actually determines points; a "quantity" scoring row that
compared the student's own farms/cattle-*derived* output instead of their actual stated guess
(the thing displayed as "Your Answer"), so a student could guess the quantity almost exactly and
still score zero; and a price tolerance band that omitted `Imports` entirely from its
calculation, which is severe enough that the *true, mathematically optimal* price didn't fall
inside its own tolerance band on any season with nonzero Imports — verified directly, not
assumed. **These fixes have not been retroactively applied to any already-scored season** (fixing
the formula going forward doesn't correct what's already stored) — worth deciding whether any
past Part III season needs a manual re-score, the same kind of decision the equilibrium-repair
tooling exists for.

**The Results page no longer offers a button for a season that hasn't been graded yet.**
Clicking into the current, still-pending season's results previously showed the *previous*
graded season's chart and data, but rendered under the pending season's own label and heading —
not an error, just confusing, since it looked like real results for a season that hadn't
happened. The season-selector row is now filtered to only seasons at or before whatever's
actually been graded, with a matching defensive check on the underlying load function itself so
this can't happen via any other path either.

**The Monopoly Dashboard now models Imports, not just the scoring engine.** The professor
identified a real gap: the dashboard's chart plotted the firm's own MC/ATC curves and its
"Monopoly" outcome dot starting from zero output, as if the firm's own quantity and the market's
total quantity were the same thing — true only when Imports is zero. A new, editable `Imports`
cell was added to Section 1 (pre-filled from the season's real value, same pattern as every other
parameter there), and the chart now correctly plots demand/MR against *total* market quantity
while shifting the firm's own MC/ATC curves and its outcome dot onto that same axis by the
Imports amount — the price the monopolist actually faces is `P(Qm + Imports)`, not `P(Qm)` alone.
The marginal revenue formula was re-derived from scratch (not just patched) and verified against
the same MR=MC-at-the-optimum sanity check used for the server-side scoring fix below — they now
agree to floating-point precision. When Imports is nonzero, a note appears directly on the
dashboard flagging the real modeling implication the professor raised: with outside supply the
firm doesn't control, this is closer to a dominant firm facing a competitive fringe than a
textbook pure monopoly — the math already accounts for it correctly either way, but the note
makes the distinction visible rather than silently assumed. With Imports at zero (still the
default for most seasons), every formula behaves exactly as before — verified as an explicit
regression case, not just assumed.

**A season can now be flagged to enter cleanly, without its own Imports/Tax/Subsidy reset
displaying as a market shift.** This came directly out of the Imports work above: the professor
wants the transition into the monopoly rounds to be pedagogically simple — Imports (and Tax/
Subsidy) dropped to zero entering that first monopoly season, without students being told this
looks like a real market shock. A new per-season flag, `reset_market_on` (schema and mechanism
directly analogous to the existing `long_run_on`), makes this possible: when set on a season,
`get-market-info` builds that season's before/after comparison using *that season's own*
Imports/Tax/Subsidy as the "before" state too, rather than the literal previous season's real
values — and re-solves the comparison equilibrium so the displayed "before" dot stays consistent
with the adjusted curve, rather than floating off it. Every downstream consumer (shift arrows,
the market-info magnitude hints, the interactive prediction sliders) inherits this automatically,
with no client-side changes at all — the fix lives entirely in how the server constructs the
comparison baseline. A genuine, separate shock happening alongside the reset (e.g., a real
demand-side event the same season) still displays correctly; only Imports/Tax/Subsidy are
affected. Scoring itself is completely unaffected — every scoring function still reads a season's
own real, stored parameters directly, regardless of this flag. See Section 5 for the full
mechanism and Section 7 for how this is now the actual default for the first monopoly season of
any newly-cloned class, not just a manually-set one-off.

**The Results page's "previous monopoly outcome" label had the same class of bug already fixed
once on the Market Info tab, just in a second, separate place.** The first Regulation season's
chart labeled its own "previous" dot "Previous Regulated Monopoly Outcome" even when the actual
previous season was Profit (unregulated) — because `drawResultsChart` derived that label from the
*current* season's own part, reusing it for both dots, rather than checking what the previous
season's part actually was. The Market Info tab's own transition logic already got this right
(`oldS.monopolyPart`, not the viewed season's own part); the Results page has its own,
self-contained drawing function that simply never received the same fix. Now derives the old
dot's label from `oldState.monopolyPart` directly, matching the already-correct pattern.

**The Results page now shows each season's own "What Happened This Season" text, matching Market
Info.** The data was already flowing correctly from the server — `get-market-info` has always
returned `shockDescription` regardless of whether a season is pending or complete — but
`getSeasonResults()` was fetching it and then silently never including it in its own return
value. A one-line pass-through fix, plus a new panel to actually display it.

**The Firm and Monopoly dashboards now stay accessible for completed seasons, including after
the game is entirely over — a substantial new feature with several rounds of real correction
along the way, worth understanding in full rather than just the final state.** Previously, once
`currentSeasonNumber` advanced past a season with no dashboard-eligible part (Part I, or the game
over entirely), both dashboards fell straight to "not available," regardless of real historical
work sitting in earlier seasons. A new season-selector row (`dashboardSeasonRow`,
`dashboardViewingSeason`) lets a student — or the professor, reviewing on a student's own screen —
browse any past season's dashboard, with genuine read-only enforcement once a season isn't the
live, current one (input disabled at the element level, not just visually; a defensive no-op
guard on the commit function as a second layer; autosave never fires since it's only ever
triggered by that same commit path).

Getting the season *selector* itself right took two corrections. First, it only ever showed
seasons from whichever single part-family group the currently-loaded season belonged to — since
Part II and Part III are always separate, contiguous groups, viewing a Part III season made the
entire Part II history invisible from that screen, not because of any access restriction, just
because the row's own data was scoped to the wrong group. Fixed by combining both dashboard-
eligible groups (`overview.parts` filtered to `'II'`/`'III'`) into one row, with a small,
non-interactive divider between them labeled by part, since the two are genuinely different cell
layouts under the hood. Second, rapid clicking through several season-row buttons (a natural way
to browse this feature, and exactly what "look at any given round" implies) could resolve out of
order — whichever async call happened to *finish* last would win the display, regardless of which
button was actually clicked last. Fixed with a request-token guard (`dashboardRequestToken`),
checked after every `await` point in both `bootstrapDashboard()` and
`bootstrapMonopolyDashboard()`, so a stale call abandons before writing anything once a newer one
has started — verified directly by deliberately making an earlier click resolve slower than a
later one, and the reverse.

**The much larger, and much more iterated-on, piece was getting the actual *parameter values*
right in a historical view — this went through several rounds of genuine correction, not just
one bug fix, and the reasoning behind each is worth knowing before touching this code again.**

The first, reported symptom: a historical dashboard with nothing saved showed the student's own
*market-level* prediction-form submission (their guess at overall market equilibrium, entered on
a completely separate form) as if it were their own firm's quantity — a market has thousands of
firms, so this could be off by orders of magnitude. Fixed by no longer falling back to that
submission at all in read-only mode.

That fix alone didn't fully solve it: a second, deeper bug was hiding underneath. An
unconditional loop wrote every cell's own hardcoded template value — including editable ones —
*before* the read-only logic ever ran, so a cell with genuinely no saved data still showed a
stale, hardcoded default (a literal `"6"` for price, `"20000"` for quantity) rather than actually
going blank. Fixed by having that first loop skip editable cells entirely, since the second,
more-specific loop already handles them correctly on its own.

Then a screenshot surfaced the real, underlying distinction that had been missed: *every*
Section 1 field was blank in an untouched season, not just the ones with no plausible fallback.
Section 1's "given" market parameters (land rent, calf cost, technology, alpha, beta, credit
line) are objective, knowable facts about a season's economy — not something a student invented —
so even an untouched dashboard should show the *real* value, not blank. That's genuinely
different from a cell representing the student's own arbitrary choice or prediction, which has no
"true" value to fall back on at all. `MARKET_PARAM_ADDRS` now distinguishes the two explicitly.

Getting *which* real value to show took one more, more fundamental correction. The first attempt
used the specifically-viewed season's own, individually-revealed true values — so Season 8 showed
its own real land rent, genuinely different from Season 6's. That's wrong for a different reason
than either bug above: the professor was explicit that a dashboard's parameters must **not**
auto-update season to season — only the student's own edits should change what's shown, precisely
because an instructor reviewing a student's dashboard needs to be able to tell whether the student
is actually engaging with their own cost model or just looking at auto-populated numbers that
happen to look plausible. The fix anchors on one fixed reference point instead — the real values
from that Part's own *first* season (`getPartStartingParams()`, cached per part family) — held
constant across every season in the Part, live or historical, unless the student themselves edits
a cell. The one case with nothing to anchor to yet — the Part's own very first season, while it's
still live and pending — correctly falls back to the ordinary previous-Part baseline, the only
option available at that point.

Last, `C7` ("Expected price at equilibrium") itself was misclassified in the market-parameter fix
above. Unlike land rent or calf cost, it's the student's own subjective prediction, not a
knowable market fact — the same kind of student-choice cell `G7` (quantity produced) already
correctly was. It's excluded from `MARKET_PARAM_ADDRS` and blanks when never entered, rather than
showing a computed number (a `7.8691`-style value) no student would plausibly have typed
themselves. This surfaced a second, related correction: the "carry the student's own most recent
prior entry forward" fallback (`findMostRecentCompatibleSave`) had been deliberately skipped in
read-only mode, reasoning that a historical view should only show what that exact season
contained. That reasoning didn't hold up — since the dashboard doesn't reset each round, a value
the student entered once and never touched again genuinely *is* what every later, untouched
season's dashboard contained too, so showing it is accurate, not a fabrication. It now runs in
both live and read-only mode, for every editable cell — market-parameter and student-choice
alike — not just the live season.

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
           response instead of throwing when asked for a season past the end of the dataset.
           Also now respects `reset_market_on` (see Section 2/5) — when a season is flagged,
           the before/after comparison it builds uses that season's OWN Imports/Tax/Subsidy as
           the "before" state too, so a deliberate reset of those fields doesn't display to
           students as a market shift.
         — run-current-season: scores everyone, advances the game. Auth was retrofitted from a
           shared `X-Instructor-Secret` header to the same per-instructor ownership check the
           two instructor-side functions below already use — see Section 5.
         — submit-prediction: upserts a submission + sends confirmation email; validates the
           submitting season matches game_state before checking part, so a stale browser tab
           gets a clear "please refresh" rather than a confusing mismatch error. Also checks
           that the submitting user is actually rostered in the instance before attempting the
           write — previously an authenticated-but-unrostered user (e.g. an instructor testing
           with their own account) fell through to a raw, unfriendly Postgres foreign-key
           violation instead of a clear "you're not on the roster" message; see Section 5.
         — repair-equilibrium-history: one-time-use, corrects already-corrupted Part III
           equilibrium_history rows from before the Section 18.7 fix (see spec for why this
           exists as a separate function rather than a one-off SQL script)

Browser (instructor.html, hosted on GitHub Pages alongside Burgerville.html, isolated session)
   │
   ├─ Supabase Auth (same OTP mechanism, separate storageKey so it doesn't collide with a
   │  student session in the same browser)
   │
   ├─ Direct Postgres queries — own profile/classes, own season_parameters, own roster
   │
   └─ Two Edge Functions (Deno/TypeScript, service-role, deployed to Supabase)
         — bulk-upload-roster: pre-provisions student auth accounts (no email sent) so a roster
           can be populated before any student has signed in
         — clone-season-template: seeds a new class's seasons from an already-validated
           instance (defaults to test-run-vf26), with a small seeded variation per season
```

Both instructor-side functions verify the caller's own identity and destination-instance
ownership explicitly, in code, before doing anything privileged — `run-current-season` now
follows the identical pattern. See `Burgerville_Instructor_Handoff_Document.md` for that track's
own full architecture detail and build history; not duplicated here.

Four Edge Function files have changed across everything described in this document:
`run-current-season` (the auth retrofit), `submit-prediction` (the roster check),
`ScoringPartIIIProfit.ts`/`ScoringPartIIIRegulation.ts` (the three scoring bugs), and
`get-market-info` (the `reset_market_on` mechanism) — each needs its own
`supabase functions deploy --use-api` to take effect. Every other fix described here (the Firm
Cost Diagram, the game-over flow, the two Part II Results-page fixes, the persistent prediction
sidebar, the Part III slider restoration, the results season filtering, the Monopoly Dashboard's
Imports feature) remained purely in `Burgerville.html`, no redeploy needed. Worth confirming
directly which layer(s) an upcoming change actually touches before assuming a redeploy is or
isn't needed (see Section 7's note on this).

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
| Edge Functions (student-side track) | `supabase/functions/{get-parts-overview,get-market-info,run-current-season,submit-prediction,repair-equilibrium-history}/index.ts` |
| Client (instructor interface) | `instructor.html` — deployed alongside `Burgerville.html` in the same GitHub repo |
| Edge Functions (instructor-side track) | `supabase/functions/{bulk-upload-roster,clone-season-template}/index.ts` |
| Instructor-side schema migrations | `migration_instructor_interface.sql`, `migration_participants_instructor_select.sql` — both already applied |
| `reset_market_on` migration + data setup | `migration_add_reset_market_on.sql` (adds the column), `set_season_11_reset.sql` (the actual data change — zeroes `test-run-vf26` Season 11's Imports/Tax/Subsidy and flags it) — both in the instructor-integration folder since they're applied via direct SQL, but the effect is entirely student-facing (`get-market-info`'s behavior); see Section 2/5 |
| Instructor track's own full history | `Burgerville_Instructor_Handoff_Document.md` — companion to this document, not a replacement |
| Shared engine/scoring logic | `supabase/functions/_shared/*.ts` (Engine, MarketPrediction, MonopolyEngine, ScoringPartI/II/IIIProfit/IIIRegulation) |
| Firm Dashboard prototypes (reference, pre-integration) | `Burgerville_Firm_Dashboard_Prototype.html` (Part II), `Burgerville_Monopoly_Dashboard_Prototype.html` (Part III, both modules) — both now integrated directly into `Burgerville.html`; historical reference, not deployed anywhere |
| Firm Cost Diagram prototype (reference, pre-integration) | `Burgerville_Cost_Diagram_Prototype.html` — standalone, interactive (scenario picker, its own "Watch the Adjustment" slider) build used to iterate on the design with the professor before integrating into `Burgerville.html`; also historical reference now |
| Diagnostic queries | `diagnose_part_ii_firm_scoring.sql`, `diagnose_season_13_part_mismatch.sql` (read-only; the pattern to reach for when a symptom's real cause isn't obvious from the client alone) |
| Test suite (Node, run locally) | `test_*.js` — 40+ files, 550+ checks, covering auth, submission, resubmission, both dashboards, Part II/III display and scoring derivations, the equilibrium-aware chart logic, font-scaling regressions, the game-over end state, the Firm Cost Diagram's layout/reveal/stability behavior on both Market Info and Results, `run-current-season`'s retrofitted auth flow (`test_run_current_season_auth.js`) plus the "Run Season" button's actual DOM behavior via jsdom (`test_run_season_button.js`), `test_market_page_items.js` — 78 checks covering the cost-diagram alignment fix, the class-ID header addition, the market/firm prediction-form delineation, the persistent prediction sidebar, and the results-season filtering fix — `test_part_iii_prediction_sliders.js` (13 checks), `test_monopoly_dashboard_imports.js` (20 checks — the Monopoly Dashboard's Imports feature), `test_results_previous_monopoly_label.js` (8 checks — the "previous monopoly outcome" mislabeling fix, Section 2), `test_results_context_and_dashboard_persistence.js` (12 checks — the Results page context panel plus the dashboard cross-season persistence fix, Section 2), `test_dashboard_historical_view.js` (25 checks — the dashboard season-selector and read-only enforcement, Section 2), `test_dashboard_race_condition.js` (5 checks — the request-token guard, verified with deliberately out-of-order async resolution in both directions), and `test_dashboard_part_anchored_params.js` (17 checks — the full parameter-accuracy arc: market-vs-student-choice cells, Part-anchored values, and cross-season carry-forward, Section 2) |
| Edge Function tests (student-side track, Node, run locally) | `test_submit_prediction_roster_check.js` (13 checks — the roster-check fix above), `test_scoring_part_iii_profit_fixes.js` / `test_scoring_part_iii_regulation_fix.js` (10 + 5 checks — the three scoring bugs in Section 2/5), and (new) `test_reset_market_on.js` (21 checks — the `reset_market_on` mechanism, Section 2/5), all run against the real, combined `_shared/*.ts` engine files, not mocks |
| Full project history & validation record | `Burgerville_Engine_Spec.md` |
| Original step-by-step deployment instructions | `Burgerville_Supabase_Deployment_Guide.md` |
| GitHub repo | `tomecon-wcu/burgerville-app`, served at `tomecon-wcu.github.io/burgerville-app/`, **Pages source: "Deploy from a branch"** (no custom `.github/workflows/` file exists — see the deployment-timeout note in Section 6 before suggesting any workflow-file edit) |
| Supabase project | `oxhojojzwvnfzsfpybyf` (org: Tomecon-wcu), instance ID `eco112-fall26` |
| Email | Resend, verified domain `funconomics.org` |

## 5. Key design decisions worth knowing before you change anything

- **`instance_id` is on every table** — this is the seam for multi-instance support (many
  classes/instructors on one deployment), and it's no longer just a theoretical seam: two real
  instances exist and are actively used through it (`eco112-fall26`, the live class, and
  `test-run-vf26`, the disposable test/validation instance), each managed independently through
  `instructor.html`. Don't remove this column even where a query looks redundant for a single
  class — the client itself still hardcodes one `INSTANCE_ID` constant per deployed file rather
  than letting it vary per URL or login (see Section 6), so this seam is doing real work at the
  database/instructor-interface layer even though the student-facing client doesn't yet expose
  it dynamically.
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
- **`run-current-season`'s auth is now per-instructor ownership, not a shared secret.** The
  single `X-Instructor-Secret` header (checked against one project-wide environment variable)
  was retired in favor of the exact pattern the instructor-side `bulk-upload-roster` and
  `clone-season-template` functions already used: the caller's own access token identifies who's
  asking (`userClient.auth.getUser()`), then the service-role client checks `instances.owner_id`
  for that specific instance before doing anything privileged. This was a precise, isolated patch
  to the `Deno.serve` auth block alone — `runSeasonLogic` itself, the actual scoring logic, was
  not touched, and the existing scoring tests were re-run against the patched file to confirm
  nothing there had shifted. **This means the old curl-with-`X-Instructor-Secret` workflow no
  longer works at all** — see Section 7 for the current way to run a season, both via
  `instructor.html`'s new "Run Season" button and via curl with a real instructor session token.
- **`INSTRUCTOR_SECRET` has since been removed from Supabase entirely**, not just retired from
  `run-current-season` — every function that ever depended on it (`run-current-season`,
  `repair-equilibrium-history`) has been retrofitted to the ownership pattern above and deleted
  from the project's environment variables. Any documentation, script, or memory still showing
  an `X-Instructor-Secret` header is stale; Section 7's curl examples have been updated to match.
- **A single, shared UI element can be dynamically re-parented between tabs rather than
  duplicated — the pattern used for both the prediction sidebar and the event panel.** Rather
  than maintaining two copies of the same form or the same "what's happening this season" box
  (one per tab, with the obvious risk of the two drifting out of sync), a single DOM element
  gets moved between different parent containers via `appendChild` depending on which tab is
  active (`dockPredictionSidebar`, `dockEventPanel`). `appendChild` on an already-attached node
  moves it rather than cloning it, so this is genuinely the same element throughout — same
  state, same listeners, nothing to keep in sync because there's only ever one copy. Worth
  reaching for this pattern again before building a second copy of anything that's identical
  regardless of which tab shows it.
- **A flex item's own `flex-basis`/`grow`/`shrink` only take effect when it's a DIRECT child of
  the flex container — nesting it inside an extra wrapper div silently breaks this.** The
  persistent prediction sidebar (`flex: 0 0 380px`, set inline on the sidebar element itself)
  rendered at roughly half its intended width specifically on the Dashboard tab, not Market
  Info. The cause: the docking code appended the sidebar into an empty intermediate wrapper div,
  which then became the *actual* flex item (with its own unconstrained default sizing,
  shrink-to-fit its content) while the sidebar's own flex-basis, now one level too deep, had no
  effect at all. Fixed by docking directly into the flex row itself, not a nested wrapper. Worth
  checking for this exact pattern (an element with its own `flex:` styling, appended into
  something that isn't literally the flex container) any time a flex-sized element behaves
  correctly in one place but not another that should be identical.
- **A design decision that covers half a two-part task can look complete while quietly leaving
  the other half unsupported.** Part III's demand/supply prediction sliders were deliberately
  hidden on the reasoning "the prediction happens through the submission form" — true for the
  monopoly-specific half of that form (price, quantity, MR, MC), but the form has always *also*
  asked students to estimate the competitive market outcome the monopoly replaced, which is
  exactly the same "predict how supply and demand shift" task the sliders exist for on every
  other part. The original reasoning wasn't wrong, just incomplete — it accounted for one part
  of what the form asks and silently dropped a tool for the other part. Restored by giving Part
  III the same interactive prediction chart Parts I/II already had, driven by the same
  competitive baseline the form's own competitive-price/quantity fields are scored against.
- **Three confirmed bugs in Part III's server-side scoring, found while investigating a report
  of "close answers scoring zero," each verified independently before being fixed, not assumed
  from the symptom alone:**
  - The marginal-revenue formula in `ScoringPartIIIProfit.ts` (server-side, what actually
    determines points) still had an error that a client-side display fix had already corrected
    — the client's own comment explained the fix, but it was never carried back into the
    scoring function. Verified two ways: an independent, from-scratch symbolic derivation of
    marginal revenue matched the corrected formula exactly, and — a direct, powerful sanity
    check worth remembering for any future monopoly-pricing work — at the TRUE profit-maximizing
    optimum, marginal revenue must equal marginal cost by definition; before the fix, the
    formula's own output for MR and MC at that exact point didn't agree at all.
  - `pts.quantityLevel` compared the student's farms/cattle-*derived* actual output against the
    optimal band, not their *stated* quantity guess — the thing actually displayed as "Your
    Answer" on that row. A student could state a near-perfect guess and still score zero if
    their separate farms/cattle choice happened to produce something else. The row's own label
    ("Expected monopoly quantity," not "...from your farms/cattle choice" the way the adjacent
    price/MC/MR rows are labeled) confirmed the intent was always a genuine guess-the-quantity
    question, the same pattern as Part I/II's `quantity_estimate` fields.
  - `buildMonopolyPriceToleranceBand` omitted `Imports` entirely from its price calculation,
    while the actual price a monopolist faces always includes it (`inverseDemand(quantity +
    Imports, ...)`). Verified directly, not assumed: with a realistic nonzero `Imports` value,
    the TRUE optimal price didn't fall inside its own tolerance band at all — meaning a student
    who guessed the exact correct price would still have scored zero, on every season with real
    imports. The identical bug, in the identical shape, existed independently in
    `ScoringPartIIIRegulation.ts`'s own `buildRegulationToleranceBand` — not something the report
    was about, found by checking whether the same mistake had been made twice. A second, related
    mismatch in that same function (the quantity band compared an Imports-inclusive actual value
    against an Imports-exclusive band) was fixed alongside it.

  **None of these three fixes have been retroactively applied to any already-scored season** —
  fixing the formula corrects scoring going forward only; a past season's stored points need a
  manual re-score if that's wanted, the same category of decision the equilibrium-repair tooling
  exists for elsewhere in this project.
- **The Results page's season-selector row must be filtered to seasons that have actually been
  graded, not just seasons that exist.** `season_parameters` rows for the current, pending season
  are readable (season number and part aren't secret, even though `Q_0`/`dQ_dP` are), so a naive
  "show a button for every season in this part" listed the pending season too. Clicking it didn't
  error — `get-market-info` returned the most recently completed season's baseline, which then
  rendered under the pending season's own label and heading, looking like real results for a
  season that hadn't happened. Fixed by filtering to `season <= lastGradedSeason` both where the
  buttons are rendered and, defensively, inside the load function itself, so no other call path
  can reach the same confusing state.
- **A separate `window.eval()` call cannot reassign a script's own top-level `let`/`const`
  variable — worth knowing before writing a jsdom test against this codebase again.** Each
  `eval()` call at the script level gets its own independent lexical environment for `let`/
  `const`; a bare assignment in a later, separate `eval()` call silently creates an unrelated
  `window` property instead of reaching the original binding, even though reading it back
  (also via `eval()`) appears to confirm the assignment worked. The actual application's own
  functions, defined in the original `eval()` call, never see the change. The fix: define a
  small setter function *inside* the original script content before the first `eval()` runs
  (e.g. `window.__setResultsOverview = (v) => { resultsOverview = v; }`), so its closure
  correctly captures the real binding — the same pattern already used earlier for exposing
  `aggregateResults`/`buildResultsCsv` from the results-download work. Function *declarations*
  don't have this problem (they become real `window` properties directly); only top-level
  `let`/`const` do.
- **The Monopoly Dashboard's chart needed the firm's own MC/ATC curves shifted onto a shared,
  total-market-quantity axis once Imports exists — plotting them starting at zero output was
  quietly wrong whenever Imports was nonzero.** The demand and MR curves are drawn against total
  market quantity; the firm's own cost curves are a function of the firm's own output. With
  Imports at zero the two axes coincide, which is exactly why this went unnoticed until it
  didn't. The fix shifts MC/ATC and the "Monopoly" outcome dot right by the Imports amount, and
  — this is the part that's actually substantive, not cosmetic — the price the monopolist faces
  is now correctly computed as `P(Qm + Imports)`, not `P(Qm)` alone. Marginal revenue was
  re-derived from scratch (`d(TR)/d(q_firm)` where `TR = P(q_firm + Imports) × q_firm`), the same
  non-doubled-Imports-coefficient result as the server-side scoring fix, and verified the same
  way: at the true profit-maximizing output, MR and MC now agree to floating-point precision.
- **`reset_market_on` (schema and mechanism directly modeled on the existing `long_run_on`) lets
  a season's own Imports/Tax/Subsidy reset without displaying as a market shift to students.**
  The mechanism lives entirely inside `get-market-info`'s baseline/`oldState` construction: for a
  flagged season, the "before" comparison state uses that season's OWN Imports/Tax/Subsidy
  (typically zero) instead of the literal previous season's real values, and the comparison
  equilibrium is re-solved against those adjusted values — otherwise the displayed "before" dot
  would sit off the curve it's supposed to be on, since the cached `equilibrium_history` row for
  the real previous season no longer matches the adjusted params being shown. Every downstream
  consumer (shift-detection arrows, magnitude hints, the interactive prediction sliders) inherits
  this correctly with zero client-side changes, since they all just consume whatever
  `get-market-info` returns. Scoring is completely unaffected — every scoring function still
  reads a season's own real, stored Imports/Tax/Subsidy directly, regardless of this flag; the
  flag only changes how the comparison baseline gets constructed for display. Verified directly
  that a genuine, separate shock (e.g. a real demand change) alongside the reset still displays
  normally — the reset is scoped to Imports/Tax/Subsidy specifically, not a blanket suppression
  of all comparison for that season.

## 6. Known simplifications — flagged on purpose, not forgotten

- **Instructor auth is now per-instructor ownership, not a shared secret — see Section 5.**
  This is now resolved everywhere it applied: `run-current-season`, `repair-equilibrium-history`,
  `bulk-upload-roster`, and `clone-season-template` all use the ownership pattern, and
  `INSTRUCTOR_SECRET` has been deleted from the Supabase project entirely — nothing depends on
  it anymore.
- **`eco112-fall26`'s missing `owner_id` has been resolved** — it was created via manual SQL
  before the instructor-interface schema existed, so `owner_id` was `null` until backfilled by
  hand. The same gap and the same one-time fix came up again independently for `test-run-vf26`
  (also created before that schema existed) — worth checking for on any other pre-existing
  instance connected to the instructor interface later, rather than assuming every instance
  already has an owner.
- **Adding a season to a NEW class is now guided** (clone from a validated instance, then edit
  each field) via `instructor.html`'s season wizard — the manual-SQL path in Section 7 remains
  the way to add a season to an *existing*, already-configured class like `eco112-fall26`, since
  the wizard's cloning flow is specifically for a class's initial setup, not appending later
  seasons to one already running.
- **Adding a student to the roster is now genuinely one step**, via `instructor.html`'s CSV bulk
  upload or single-add form (`bulk-upload-roster` pre-provisions their `auth.users` row, no email
  sent) — resolves what was previously a real two-step gap. No self-service team-joining exists
  either way.
- **Multi-instance support is schema-ready and now genuinely in active use, not just a
  theoretical seam** — `test-run-vf26` and `eco112-fall26` are both real, live instances managed
  through the same instructor interface, each with its own owner, roster, and season data. The
  client still hardcodes a single `INSTANCE_ID` constant per deployed file rather than letting it
  vary per URL or per login (`Burgerville.html` for the live class, `Burgerville_TestRun.html`
  for the test instance, kept in sync as two near-identical files rather than one
  dynamically-scoped client) — worth reconsidering if a third instance is ever needed.
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

**Run the current season** (scores everyone, advances the game). Auth is now a real instructor
session token, not a shared secret — get one via `instructor.html`'s dashboard "Copy access
token" button (copies the currently signed-in instructor's own session token to the clipboard),
or `Application/Local Storage` in the browser console if you'd rather not sign in separately:
```
curl -X POST https://oxhojojzwvnfzsfpybyf.supabase.co/functions/v1/run-current-season \
  -H "Authorization: Bearer <instructor's own access token>" \
  -H "Content-Type: application/json" \
  -d '{"instanceId": "eco112-fall26"}'
```
In practice, `instructor.html`'s own "Run Season →" button (per class, on the dashboard) does
this same call directly and is the easier path day to day — the curl form above is for scripting
or when the button isn't available for some reason. A reusable script pattern for the curl form
(avoids re-typing this every season, and reports which season number was actually just scored so
you can confirm it matches what you expected):
```bash
RESPONSE=$(curl -s -X POST https://oxhojojzwvnfzsfpybyf.supabase.co/functions/v1/run-current-season \
  -H "Authorization: Bearer <instructor's own access token>" \
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
  -H "Authorization: Bearer <instructor's own access token>" \
  -H "Content-Type: application/json" \
  --data-binary @request.json
```
The quoted heredoc delimiter (`'EOF'`) means everything between the two `EOF` lines is treated
as pure literal text — no quote-interpretation, no risk of the same issue recurring.

**Repair corrupted Part III equilibrium data** (one-time use per instance, only needed for
seasons run before the Section 18.7 fix — safe to re-run, always recomputes from source
parameters). Same auth as above — an instructor's own session token, not a shared secret:
```
curl -X POST https://oxhojojzwvnfzsfpybyf.supabase.co/functions/v1/repair-equilibrium-history \
  -H "Authorization: Bearer <instructor's own access token>" \
  -H "Content-Type: application/json" \
  -d '{"instanceId": "test-run-vf26"}'
```
Response lists each affected season's `before`/`after` price and quantity directly — check
these against what you'd independently expect before trusting the fix took effect.
**`eco112-fall26` has since been checked and confirmed clean** — as of the check, it had no
completed Part III seasons yet, so there was nothing for the Section 18.7-era bug to have
touched. That was a point-in-time result, not a standing guarantee — worth re-running once the
live class actually reaches and completes a Part III season, not assumed clean indefinitely.

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

**The instructor interface is no longer "the next major piece of work" — it's built, working,
and consolidated.** The framing in earlier versions of this section (the "new chat" switch that
hadn't happened yet, the instructor interface as an open, unstarted plan) is now stale and has
been replaced. `instructor.html` covers registration, roster management, season setup and
editing with a live equilibrium guardrail, a "Run Season" control, a results-download view, and
per-instructor ownership auth throughout — see `Burgerville_Instructor_Handoff_Document.md` for
the full, current record of that track. This document's own job is just the student-facing app;
it points to the instructor doc rather than duplicating that track's status here.

**The real open items for the student-facing side, right now:**

1. **Decide whether to re-score any already-graded Part III season** in light of the three
   scoring bugs fixed this round (Section 2, Section 5). The formula fixes correct scoring going
   forward only — they don't retroactively touch what's already stored for a season that ran
   under the old, buggy formulas. `test-run-vf26` is the only instance where this has come up so
   far; `eco112-fall26` (the live class) hadn't reached a completed Part III season as of the
   last check, so the question is currently hypothetical there, not yet urgent.
2. **Confirm the fixed `ScoringPartIIIProfit.ts`/`ScoringPartIIIRegulation.ts` have actually been
   deployed** (`supabase functions deploy --use-api` from the `burgerville-edge` folder) — these
   are Edge Function changes, not client-only, so re-uploading `Burgerville.html` alone doesn't
   apply them.
3. **`eco112-fall26`'s Part III equilibrium-history check should be re-run once it actually
   reaches a completed Part III season** — the earlier "confirmed clean" result (Section 7) was
   a point-in-time check against a class that hadn't gotten there yet, not a standing guarantee
   for whenever it eventually does.
4. **`reset_market_on` needs its migration, data setup, and Edge Function deploy actually run** —
   `migration_add_reset_market_on.sql`, then `set_season_11_reset.sql` (zeroes `test-run-vf26`
   Season 11's Imports/Tax/Subsidy and flags it), then `supabase functions deploy --use-api` for
   the updated `get-market-info`. None of these have been confirmed run as of this writing.
5. **`eco112-fall26`'s own Season 11 won't inherit the clean-monopoly-entry default** —
   `set_season_11_reset.sql` only touches `test-run-vf26`, the template every NEW class clones
   from; `eco112-fall26` was cloned before this feature existed, so its Season 11 row is
   untouched. If that class hasn't reached Season 11 yet, the same pattern (adapted for that
   instance) needs running there directly, or the new checkbox can be set by hand through
   `instructor.html`'s season-edit form.

**Worth keeping in mind for whatever comes next, student-side or otherwise:** the working
pattern that's made this project's testing and integration phases productive is tracing a
symptom to its *actual* root cause before writing a fix, even when the first hypothesis looks
right — a genuinely long list of real bugs across this project turned out to be one or two
layers deeper than they first appeared (a CSS class collision, a data-pipeline bug, a
per-frame-recomputed axis scale, a branch that returned before reaching code it needed to run,
a flex item losing its own sizing one wrapper div too deep, a formula fixed in one place but
never carried to its counterpart, a tolerance band silently omitting a real term of the actual
formula it was supposed to bound) all initially looked like something simpler than what they
actually were. The dashboard parameter-accuracy work in Section 2 is maybe the clearest recent
example of this same pattern playing out over several rounds in a row, not just once: what
looked like one bug (market quantities showing instead of firm quantities) turned out to be an
earlier, unconditional write happening before the fix even got a say, then a genuine
misunderstanding of which cells have a "true" value at all, then a wrong choice of *which* true
value, then one more cell simply sorted into the wrong category. Each fix was correct as far as
it went; none of them were the full picture until the last one. The same discipline applies to
anything scoring-adjacent in particular — a formula that *looks* right but is subtly wrong is a
much quieter, harder-to-spot failure than almost anything on the display side, since there's no
visibly broken pixel for a student to notice and report; it just quietly produces wrong points,
exactly as happened here.
