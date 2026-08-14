# Burgerville Instructor Interface — Hand-off Document

**Purpose of this document:** get anyone (a TA, a hired developer, a future version of
yourself after a break, or a fresh AI assistant session with no memory of how this was built)
productive on this track in five minutes, not five hundred pages. It's a companion to
`Burgerville_Handoff_Document.md` (the main student-app/engine track), not a replacement for
it — see Section 2 for how the two relate.

---

## 1. Current state

The instructor interface is a separate single-page app, `instructor.html`, deployed alongside
`Burgerville.html` at `tomecon-wcu.github.io/burgerville-app/instructor.html`. An instructor
can currently, live, on the real Supabase project:

- Sign in via email + one-time code, on a session isolated from the student app.
- Register a profile and create classes (`instances` rows), auto-generating a Class ID
  following the existing `eco112-fall26` convention.
- View all their classes on a dashboard, each with "Run Season," "Manage Roster," and
  "Configure Seasons."
- Upload a class roster via CSV, or add students one at a time — both pre-provision the
  student's account server-side, so they don't need to sign in once first before being
  rostered (previously a real gap; see Section 5).
- Set up a new class's seasons by cloning them from an existing validated instance (defaults to
  `test-run-vf26`). Only Season 0 gets a random perturbation (a small variation so each class's
  numbers differ slightly); every later season's own delta from the template is preserved
  exactly, additively, rather than each season independently re-perturbing its own absolute
  value -- see Section 5 for the bug this fixed and how it was verified.
- Edit any season through a guided form: numeric fields grouped by category, a bulk-apply
  option for the handful of fields (technology, land/cattle shares, interest, endowment) that
  describe market structure rather than a specific season's shock, a live-updating equilibrium
  preview (real engine math, not an approximation), a side-by-side comparison against the
  previous season showing exactly what changed, Part-I field locking, a "roll this change
  forward" option, and Previous/Next season navigation. See Section 6 for the full breakdown --
  this screen grew substantially since the last version of this document.

**Now also built, as of this update — the season edit screen is a genuinely different, more
complete tool than it was:**
- **The equilibrium preview/guardrail** — real `Engine.ts` math (ported and verified
  byte-for-byte behaviorally identical to the source, not reimplemented from memory), computing
  a live price/quantity preview as the instructor types, flagging degenerate configurations
  (price or quantity at or near zero) with a required confirmation before saving one anyway.
  When a tax or subsidy is active, the preview correctly distinguishes what sellers receive from
  what buyers pay — this needed a real correction mid-build; see Section 8.
- **A season-to-season comparison panel** — shows the previous season's own values next to
  whatever's currently typed, with the delta made the visually prominent thing rather than
  requiring mental subtraction. Directly addresses a real, reported confusion: a season's
  absolute demand value could silently include a shock the shock-description text never
  mentioned.
- **A demand-shift field, separate from Q₀ itself.** Q₀ is now always read-only/computed;
  instructors enter the shift directly (in billions, e.g. "4" for four billion pounds), and Q₀
  updates live to match. Imports uses the same billions-entry convention, labeled "(supply
  shift)" to name its actual role in the model.
- **Part I (Market Dynamics) field locking** — production and cost fields disabled and visibly
  explained, not just hidden, since students at that stage have no access to the firm model and
  couldn't reasonably predict those changes. Unlocks live if the Part dropdown changes.
- **Roll-forward** — an opt-in checkbox that, when checked and something actually changed,
  shows an explicit confirmation and then shifts every later season's own value by the same
  additive delta, preserving each one's own originally-designed shock size rather than leaving
  them computed against a baseline that's since moved.
- **Previous/Next season navigation buttons**, and saving now advances directly to the next
  season instead of returning to the list — a natural sequential-editing workflow.
- **Page layout is now a single linear flow**, not a two-column sticky panel: numeric fields,
  then the comparison table showing what changed, then the shock description (with a note
  instructing the message to match the table above) and season name (moved down to sit right
  beside it, with its own note that it should be a brief label, not the full description), then
  Save at the very bottom.

**Still not yet built:** the scoring-weights panel (explicitly deferred, not just delayed — see
Section 9), a results-download view for instructors (see Section 9), and Part III's own
guardrail coverage — the preview currently uses `Engine.ts`'s competitive-market functions only;
`MonopolyEngine.ts`'s `findMonopolyOptimum`/`findRegulatedOptimum` were handed off but aren't
wired in yet.

**Verified by 258 automated checks across 16 suites** — the original 138 across 6 suites, 24
more across the two Run Season suites, and 96 more across 8 further suites built for everything
described above. See Section 7.

---

## 2. Relationship to the main Burgerville track (now consolidated)

**This was a two-chat split; it no longer is, as of this update.** Registration, roster
management, and season cloning/editing (Sections 5–6 below) were built in a separate chat from
the one that built `Burgerville.html` and the shared engine code, on the professor's own
explicit request, specifically to keep the two build efforts from interfering with each other's
context. That was a clean split for most of the build, but it started to strain once real
work here needed the actual, validated engine code rather than fragments of it — the season
wizard's guardrail and the "Run Season" control both depend on `_shared/Engine.ts` and
`run-current-season/index.ts` directly, and reconstructing those from `conversation_search`
snippets risked silently diverging from the validated original.

**The professor's explicit decision, acted on directly:** consolidate the remaining
instructor-side work into the engine track's own chat, rather than continuing to pass files back
and forth as each new dependency came up. The two Edge Functions (`bulk-upload-roster`,
`clone-season-template`), the two migrations, and `instructor.html` itself were uploaded directly
into that chat's workspace — not reconstructed from search snippets — specifically so the
`run-current-season` auth patch (Section 5) could be built against the two functions' *actual*
identity-verification code, not an independently-invented approximation of it.

**What this means going forward:** there is no longer a second, separate instructor-interface
chat to hand work back to. This document continues to exist as the reference for this track's
own history -- the consolidation itself didn't invalidate anything in Sections 4-9 below, though
several of them have since grown substantially for unrelated reasons (new features, not the
consolidation). New instructor-side work happens in the same chat as the engine track, with
direct access to the real files rather than `conversation_search` reconstruction. If a genuinely
separate instructor-track chat is ever started again for some other reason, re-read this section
first: the file-passing friction that motivated the original two-chat split is exactly what
consolidation was meant to resolve.

---

## 3. Architecture at a glance

```
Browser (instructor.html, hosted on GitHub Pages, isolated session from Burgerville.html)
   │
   ├─ Supabase Auth (email one-time code) — same mechanism as the student app, separate
   │  storageKey so signing in here doesn't collide with a student session in the same browser
   │
   ├─ Direct Postgres queries (via supabase-js + Row Level Security)
   │     — creating/reading own instructor profile and classes, reading/editing own
   │       season_parameters, reading own roster, adding/editing own teams
   │
   └─ Three Edge Functions (Deno/TypeScript, service-role, deployed to Supabase)
         — bulk-upload-roster: pre-provisions student auth accounts (no email sent) so a
           roster can be populated before any student has signed in; upserts teams/participants
         — clone-season-template: copies season_parameters + game_state from an existing
           validated instance into a newly-registered class, applying a seeded +/-5%
           variation to q_0 per season
         — run-current-season: NOT owned by this track -- it's the engine track's own scoring
           function, called directly from the dashboard's "Run Season" button. Listed here
           because the dashboard now depends on it, not because it lives in this track's own
           codebase. Its auth was retrofitted (Section 5) to match the ownership pattern the
           other two functions already used, specifically so this call could be made safely.
```

**Why service role for all three:** each needs to act across ownership boundaries that RLS's
per-row model can't express directly -- creating an `auth.users` row nobody has signed into yet,
reading a *source* instance's season data regardless of who owns it, or scoring and advancing a
class's game state. All three verify the caller's own identity and destination-instance
ownership explicitly, in code, before doing anything privileged -- the service role isn't a
shortcut around authorization, it's what makes the actually-necessary authorization check
possible to enforce precisely (see each function's own file comments for the specifics).

Everything else -- profile/class creation, season edits, roster reads -- is a direct client
query against RLS, no Edge Function needed, following the same "RLS where it's expressible,
Edge Function only where it isn't" principle as the student-side track.

---

## 4. Schema additions (this track)

Layered on top of the schema `Burgerville_Handoff_Document.md` already describes
(`instances`, `game_state`, `season_parameters`, `teams`, `participants`). Two migrations,
both already applied to the live project:

**`migration_instructor_interface.sql`** (Step 1):
- New `instructors` table: `id` (FK to `auth.users`), `name`, `email`, `institution`.
- New columns on `instances`: `owner_id` (FK to `instructors`), `semester`, `academic_year`,
  `class_identifier`, `section`.
- Ownership-scoped RLS write policies across `instances`, `game_state`, `season_parameters`,
  `teams`, `participants` -- an instructor can insert/update/delete only where they own the
  parent instance. Read policies stayed mostly as they were (`instances`/`game_state`/`teams`
  were already open-read; `season_parameters` kept its graded-only visibility rule, with one
  addition: the owning instructor can also read their own *pending* season, the one legitimate
  exception to "don't leak the answer key").

**`migration_participants_instructor_select.sql`** (found while building Step 4):
- Adds `participants_select_instructor`, an instructor-owns-the-instance SELECT policy. Step
  1's migration added instructor *write* access to `participants` but no read policy --
  building the roster screen surfaced that an instructor querying their own roster got zero
  rows back. Small, additive, safe on top of what was already applied.

No schema changes were needed for the season-configuration work (Step 5) -- `season_parameters`
already had every field the edit form needed.

---

## 5. Edge Functions (this track)

### `bulk-upload-roster`
Solves the real constraint that `participants.id` is a foreign key into `auth.users`: normally
a student has to sign in once (creating that row) before they can be rostered. This function
pre-creates the account via the Admin API (`auth.admin.createUser`, which does **not** email
anyone -- deliberately not `inviteUserByEmail`, for the same magic-link/scanner reason the
student OTP flow exists in the first place). Students still get their first real email the
normal way, when they request their own sign-in code. Handles both CSV bulk upload and
single-student add through the same code path; re-uploading a known email updates rather than
duplicates; an unrecognized team code auto-creates that team.

### `clone-season-template`
See Section 3. Worth repeating why cloning exists at all rather than hardcoded defaults: this
chat never had access to the validated engine spreadsheet's actual numbers, and fabricating
plausible-looking ones for something instructors will grade real students against was the wrong
kind of guess to make. Cloning from an already-validated instance keeps the database as the
single source of truth instead of a second, possibly-drifting copy of the same numbers. The
randomization is seeded by the *destination* instance id, so it's reproducible -- the same class
always regenerates the same numbers, which matters if a number ever looks wrong and needs
tracing back.

**A real bug here, found and fixed:** the perturbation originally applied an *independent*
random factor to every season's own absolute `q_0`, not just the first one. Two adjacent
seasons' independent noise didn't cancel cleanly, so a template's clean "+4 billion" shock could
come out as "+3-point-something billion" once cloned -- confirmed directly against a real cloned
instance before fixing it. The fix: only Season 0 draws a random factor; every later season's
cloned `q_0` is built by adding the *template's own exact delta* on top of the already-perturbed
running total, so season-to-season deltas are preserved exactly regardless of which random
baseline Season 0 happens to land on. Verified across four different destination instance IDs
(`test_clone_perturbation_fix.js`) -- every one produces an exact delta, not an approximate one.

Both functions follow the CORS pattern already established on the student-side functions
(`Access-Control-Allow-Origin: '*'`, `Access-Control-Allow-Headers` including `authorization`)
from the start, rather than needing the same fix-it-after-the-fact pass those functions needed.

### `run-current-season` (not this track's own file, but now depended on directly)
Retrofitted to use the identical ownership-verification pattern the two functions above already
established -- the caller's own access token identifies them, then the service-role client
checks `instances.owner_id` for the specific instance before scoring anything. This is what
made the dashboard's "Run Season" button (Section 6) possible at all; the shared-secret model it
replaced would have required embedding that secret in client-side code, readable by anyone
viewing the page source. Full detail on this specific change -- the exact before/after, the two
new test suites that verify it -- lives in the engine track's own spec document
(`Burgerville_Engine_Spec.md`, Section 19.8), since it's a change to that track's own file, not
a new function built here.

---

## 6. Client: instructor.html screen inventory

Single HTML file, vanilla JS, no build step -- same architecture as `Burgerville.html`. Screens,
in the order a new instructor moves through them:

1. **Email / code entry** -- OTP login.
2. **Registration** -- first-time: profile (name, institution) + first class in one form.
   Returning instructor: same form, profile fields hidden, reached via "+ New Class."
3. **Dashboard** -- lists owned classes, each with Run Season / Manage Roster / Configure
   Seasons buttons. Run Season requires an explicit confirmation (scores real students,
   irreversible from there), then shows the result -- season number, students scored, whether a
   long-run adjustment applied -- or a clear visible error, directly on the dashboard row.
4. **Roster** -- current roster table, single-add form, CSV bulk upload with a per-row result
   summary (created / updated / error, plus a note on any teams auto-created).
5. **Seasons (empty state)** -- clone-from-source form.
6. **Seasons (populated)** -- table of all seasons with an Edit button each.
7. **Season edit** -- the most substantial screen in this track, grown considerably since the
   original edit form. In order, top to bottom: Part and deadline; a live equilibrium preview
   (price/quantity from the real engine, recomputing on every keystroke, with a required
   confirmation before saving a degenerate configuration, and an explicit seller-vs-buyer price
   breakdown whenever a tax or subsidy is active); demand & policy fields, where Q₀ is always
   read-only/computed and a separate demand-shift field (billions, e.g. "4") drives it, with
   Imports using the same billions convention and labeled as the model's supply-shift analogue;
   Part-I field locking for production/cost fields (disabled with a visible explanation, unlocks
   live if Part changes); costs and market-structure fields, with the "apply to all seasons"
   bulk option for the latter; long-run/show-equilibrium checkboxes and, when a later season
   exists, a "roll this change forward" checkbox; a season-to-season comparison panel (previous
   vs. current vs. delta, unchanged fields shown muted, changed ones bold); season name (a brief
   label, with a note distinguishing it from the fuller text below) and shock description (with
   a note to make the message match what the comparison table above actually shows); and Save at
   the very bottom, which advances directly to the next season rather than returning to the
   list. Previous/Next season navigation buttons sit in the header, reusing rows already fetched
   for the comparison panel and equilibrium preview rather than adding new queries.

All screens share one `render()` helper that replaces a single `<div id="app">`'s contents --
no router, no framework, consistent with the student app's approach.

---

## 7. Testing infrastructure & how to rerun it

Three kinds of tests, all built and run inside this chat's own sandbox (Postgres installed
locally, Node available by default):

**RLS policy tests** (`test_rls.js`, 27 checks) -- run against a real, freshly rebuilt Postgres
16 instance, not verified by reading the SQL. Setup: `simulate_supabase_env.sql` (mocks
`auth.users` + `auth.uid()`), `base_schema.sql` (reconstructs the relevant live schema), then
both migrations from Section 4, then the test script. One genuine finding from this process
worth remembering: Postgres RLS `UPDATE`/`DELETE` policies filter out unauthorized rows
*silently* (`rowCount: 0`), they don't throw -- a test (or any code) checking "did this throw"
instead of "how many rows changed" will pass even when nothing was actually blocked.

**Edge Function logic tests** (`test_bulk_upload_roster.js`, `test_clone_season_template.js`,
18 checks each) -- follow the exact convention already established on the student-side track:
strip the `npm:` import and the `Deno.serve(...)` wrapper out of the real `.ts` file with a
small regex script, leaving pure/injectable functions that take a Supabase client as a
parameter, then test those directly against a fake client object in plain Node. Nothing about
the actual deployed function is reimplemented for the test -- it's the same file, minus the
parts Node can't run.

**DOM-level UI tests** (`test_registration_form.js`, `test_roster_screen.js`,
`test_seasons_screen.js`, 18/24/33 checks) -- use `jsdom` to actually load `instructor.html`'s
inline script into a simulated browser window and exercise it: fill in form fields, dispatch
real `submit`/`click` events, assert on what got rendered and what got called. One recurring
gotcha worth knowing if extending these: the app's `init()` function auto-runs on script load
exactly as it would in a real browser, and if a test also calls a screen-rendering function
directly (bypassing `init()`, to test that screen in isolation), the two can race -- `init()`'s
own `getSession()` call resolves later and silently overwrites whatever the test just rendered.
The fix used throughout: make the *first* call to `getSession()` (always `init()`'s, since it
fires synchronously during script load, before any test code runs) return a promise that never
resolves, and let subsequent calls resolve normally.

All six of the original suites currently pass in full (138 checks). Two more were added for the
Run Season integration, following the same conventions rather than inventing new ones:
`test_run_current_season_auth.js` (10 checks) drives the retrofitted `run-current-season` handler
directly with the "strip `Deno.serve`, test the logic" approach described above -- missing
Authorization header, invalid token, wrong owner, missing instance, and the success path.
`test_run_season_button.js` (14 checks) is a `jsdom` DOM test for the actual dashboard button,
reproducing the exact `init()`-race-condition workaround described above rather than
reinventing it, and confirming the request that goes out carries the instructor's own session
token with no trace of the old shared-secret header.

Eight more suites cover everything built since (96 checks total, same `jsdom` convention
throughout except where noted): `test_equilibrium_guardrail.js` (15) verifies the ported engine
math against the real, validated `.mjs` build directly -- not just that a number renders, but
that it's the *correct* number -- plus the degenerate-configuration confirmation flow.
`test_season_comparison_panel.js` (10) checks the delta table against the exact scenario that
prompted it: a 34B-to-38B Q₀ change showing a clean +4.00 B, not a derived decimal.
`test_part_i_lock_and_rollforward.js` (26) covers field locking (initial state and live
Part-dropdown toggling) and the roll-forward mechanism, including that a later season's *own*
shock size is preserved (an additive shift, not an overwrite) and that declining the
confirmation blocks the entire save atomically, not just the propagation.
`test_demand_shift_field.js` (11) and `test_layout_and_imports_units.js` (10) verify the
Q₀-from-shift computation, the billions-unit conversion for both fields across every consumer
(preview, comparison panel, save), and the actual DOM order of the restructured page --
comparison table before shock description before Save, not just that all three exist somewhere.
`test_season_navigation.js` (9) covers Previous/Next buttons and confirms save-then-advance
re-fetches the next season fresh rather than navigating with stale pre-save data (a real bug
caught while building it -- see Section 8). `test_tax_subsidy_price_labels.js` (11) verifies the
buyer/seller price correction described in Section 8, including a direct cross-check against the
real `Engine.ts` build's own computed equilibrium price.
`test_clone_perturbation_fix.js` (4) verifies the clone-time perturbation fix described in
Section 5.

**258 checks across 16 suites in total** (138 original + 24 from the two Run Season suites + 96
from the eight suites above). None of this test infrastructure is deployed anywhere -- it's a
local verification harness, re-creatable in any future session from
the files listed in Section 10.

---

## 8. Key learnings & principles (this track)

- **The 8-digit OTP bug recurred here.** Same root cause as the student-side track: this
  Supabase project's `GOTRUE_MAILER_OTP_LENGTH` is 8, not the platform default of 6. Any new
  OTP-entry UI needs to not assume a specific length (no hardcoded `maxlength="6"`, no
  "6-digit" copy) rather than being fixed reactively again.
- **Silent startup failures are worse than loud ones.** An early version of `instructor.html`
  had no error handling around client initialization; an unreplaced credential placeholder
  produced an infinite, silent "Loading…" with zero diagnostic information. Every screen-facing
  async entry point now wraps failures in a visible error message with the underlying technical
  detail included, not just a spinner that never resolves.
- **Don't fabricate consequential specifics.** Two separate points in this build where the
  honest move was to ask rather than guess, both since resolved: the validated engine
  spreadsheet's actual default numbers (resolved by cloning from a real instance instead), and
  `_shared/Engine.ts`'s actual exports (resolved once the files were handed off directly and the
  chats were consolidated -- Section 2). Neither would have failed loudly if guessed wrong; both
  would have quietly produced plausible-looking wrong numbers a real class could get graded
  against.
- **RLS write policies need to be tested against real Postgres, not read.** Reaffirmed here
  (see Section 7's UPDATE/DELETE note) -- this was already a known principle from the
  student-side track, and it caught a real test-harness bug (not a real security bug) the first
  time the participants-table policies were exercised end to end.
- **Prototype/verify before handing off**, same as the other track: every Edge Function's pure
  logic and every UI screen's behavior was tested in this chat's sandbox before being presented
  as a deliverable, not just written and assumed correct.
- **Verify domain claims against the actual model, even from the domain expert, before
  implementing them.** Asked to label the equilibrium preview's price as "what the buyer pays,"
  with the seller receiving less by the tax -- checked this against `Engine.ts`'s own code and
  the spec's own formula (`P_buyer = P_seller + Tax - Subsidy`) before writing it, and the
  relationship was actually the reverse: the price the model computes and returns is the
  seller's price; buyers pay more under a tax, less under a subsidy. Implemented the
  economically correct version and explained the discrepancy directly rather than silently
  building either the requested version or a silently-different one. A courteous "here's what I
  found and why" beats matching a stated description that contradicts the underlying,
  already-validated math.
- **A "fresh re-fetch before navigating" bug, caught by testing the interaction, not just each
  piece alone.** Making Save advance to the next season seemed like a simple navigation change,
  but if that same save had just triggered a roll-forward, the next-season row already sitting
  in memory (fetched when the screen first loaded) was stale -- navigating with it showed
  pre-rollforward numbers immediately after the update that changed them. Fixed by re-fetching
  that specific row fresh right before navigating, only once a save might have touched it.
  Wouldn't have been caught testing either feature in isolation.
- **Keep a computed field's underlying element in sync via JS rather than restructuring every
  place that reads it.** When Q₀ became computed-from-shift instead of directly typed, the
  simplest fix wasn't rewriting `readPreviewParams`, the comparison panel, and the save handler
  to understand a new "shift" concept -- it was keeping the (now disabled) `#q_0` input's own
  `.value` programmatically updated whenever the shift field changes, so every existing
  consumer of `#q_0`'s value keeps working completely unchanged. The same pattern applied again
  for Imports' unit conversion. Minimizes the blast radius of a UI-level change and reduces the
  chance of introducing a new bug in already-tested surrounding logic.
- **Don't take existing code at face value just because it's already there and looks
  plausible -- verify it the same way as anything newly written.** Several pieces of this
  track's own code (the clone-perturbation fix, the Part-I locking, the roll-forward mechanism,
  the Next-season button) were found already present, in a more complete state than expected
  going into a given turn. Rather than assuming "already there" meant "already correct," each
  was hand-traced or directly tested before being treated as done -- in one case, testing the
  new equilibrium preview against ordinary, valid season data kept failing for no apparent
  reason, which led to finding that the *existing* field-validation logic had `dQ/dP`'s sign
  backwards (it required a negative slope; the real engine's own formula, and every real season
  in the dataset, uses a positive one). Left uncaught, that pre-existing bug would have blocked
  an instructor from ever saving a correctly-cloned season without first "fixing" it into
  something that would have broken the demand curve. Fixed alongside the guardrail work itself.

---

## 9. Not yet built / next steps

**Newly identified, not yet started:** a way for instructors to download their class's results
at any point in the semester, not just after the game ends. Checked directly rather than
assumed: **neither existing migration grants an instructor any read access at all to the four
`results_part_*` tables** (`results_part_i`, `results_part_ii`, `results_part_iii_profit`,
`results_part_iii_regulation`) -- Step 1's migration covered `instances`, `game_state`,
`season_parameters`, `teams`, and `participants`, and the Step 4 addendum covered
`participants` specifically, but results were never included in either pass. This is the same
kind of gap that `migration_participants_instructor_select.sql` closed for the roster screen --
an instructor currently can't read their own students' scores at all, through any path.

Shape of the actual work: a new RLS SELECT policy per `results_part_*` table
(instructor-owns-the-instance, identical pattern to `participants_select_instructor`) is enough
to make the data readable -- no service-role Edge Function needed, since this is a same-owner
read, not a cross-ownership write. The genuinely non-trivial part is the rollup itself: each of
the four tables has a different column set (each part's own scoring breakdown), so a class-wide
"download results" view most likely means summing each table's own total-points column per
student across however many of the four tables they have rows in, rather than trying to merge
the detailed per-question breakdowns, which don't share a common shape across parts. "Available
at any point" falls out naturally once the read access exists -- `results_part_*` rows only ever
exist for seasons that have actually been scored, so a mid-semester download would correctly
just reflect whatever's been graded so far, no separate "in progress" handling needed.

**Resolved:** `run-current-season`'s shared-secret auth has been retired, and the "Run Season"
dashboard control is built and tested. See Section 5 for the pointer to the full change record.

**Also resolved, since the last version of this document:** the pre-save equilibrium preview/
guardrail is fully built and wired into the season edit form -- live price/quantity preview,
degenerate-configuration confirmation, and the buyer/seller price distinction under tax or
subsidy (Section 8). Along with it: the season-to-season comparison panel, Part-I field locking,
the demand-shift field, Imports' billions-unit conversion, Previous/Next navigation, roll-forward,
and the full layout restructuring described in Section 1. All tested; see Section 7.

**The next real piece of work: Part III guardrail coverage.** The equilibrium preview currently
only calls `Engine.ts`'s competitive-market functions (`solveEquilibrium`, `marketDemand`,
`marketSupply`, `applyLongRunAdjustment`). `MonopolyEngine.ts`'s `findMonopolyOptimum` and
`findRegulatedOptimum` were handed off in full alongside `Engine.ts` but were never wired in --
so a Part III (monopoly/regulation) season currently gets no live preview or degenerate-outcome
guardrail at all, only the same basic client-side sanity checks every part gets. Worth deciding
whether this is worth building given Part III seasons are edited less frequently than Part I/II,
or whether the existing checks are sufficient there.

**Also worth considering, not yet started:** for an already-cloned instance whose season data
predates the Section 5 perturbation fix, there's no automated way to detect or repair the
resulting noisy deltas -- the demand-shift field is the correction mechanism today (an
instructor retypes the intended clean value and saves), but nothing flags that a given instance
might need this. Building an automated detector or repair tool would need the destination
instance's *source template* to compare against, which isn't tracked anywhere in the schema
today -- `clone-season-template` doesn't record which source it cloned from on the destination
instance itself. A real gap if this comes up again.

**Explicitly out of scope, by direct decision (not just unstarted):** the Tier 3 scoring-weights
panel (point values, tolerance percentages, rank-bonus weight). This was genuinely investigated,
not assumed -- confirmed directly that none of it lives in `season_parameters` or any other
table; it's hardcoded module-level constants (`PTS_BASIC`, `PTS_ADVANCED`, `PTS_RANK`,
`PTS_BASIC_PART_II`, `PTS_EXCEL`, `MARGIN_OF_ERROR_P`, `GRAPH_SCALE`) in the engine track's own
`_shared/ScoringPartI.ts`, `ScoringPartII.ts`, and `MarketPrediction.ts`. Making this
instructor-configurable would mean restructuring those scoring functions to accept parameters,
not just adding a UI layer on top of something that already exists -- a real scope decision, and
the professor's explicit call was to leave it hardcoded for now.

**Explicitly out of scope**, unchanged from the original spec: price floor/ceiling season
mechanic, self-service student team-joining, multi-instructor co-ownership of one instance.

---

## 10. File locations

| File | Purpose | Deploy target |
|---|---|---|
| `instructor.html` | The instructor app | `burgerville-app` repo root, alongside `Burgerville.html` |
| `migration_instructor_interface.sql` | Step 1 schema + RLS | Supabase SQL editor (already applied) |
| `migration_participants_instructor_select.sql` | Instructor roster-read fix | Supabase SQL editor (already applied) |
| `supabase/functions/bulk-upload-roster/index.ts` | Roster pre-provisioning | `burgerville-edge` repo, `supabase functions deploy --use-api` |
| `supabase/functions/clone-season-template/index.ts` | Season setup via cloning (includes the perturbation fix, Section 5) | Same as above |
| `Burgerville_Instructor_Interface_Spec.md` | Original design spec for this track | Reference only |
| `test_rls.js`, `simulate_supabase_env.sql`, `base_schema.sql` | RLS verification harness | Not deployed -- local testing only |
| `test_bulk_upload_roster.js`, `test_clone_season_template.js` | Edge Function logic tests | Not deployed -- local testing only |
| `test_registration_form.js`, `test_roster_screen.js`, `test_seasons_screen.js` | DOM-level UI tests | Not deployed -- local testing only |
| `test_run_current_season_auth.js`, `test_run_season_button.js` | Run Season integration tests (auth flow + DOM behavior) | Not deployed -- local testing only |
| `test_clone_perturbation_fix.js` | Verifies Section 5's clone-time perturbation fix | Not deployed -- local testing only |
| `test_equilibrium_guardrail.js` | Preview/guardrail, cross-checked against the real `Engine.ts` build | Not deployed -- local testing only |
| `test_season_comparison_panel.js` | Delta table against the professor's own reported scenario | Not deployed -- local testing only |
| `test_part_i_lock_and_rollforward.js` | Field locking + roll-forward propagation | Not deployed -- local testing only |
| `test_demand_shift_field.js`, `test_layout_and_imports_units.js` | Demand-shift/Imports unit conversion + restructured page's DOM order | Not deployed -- local testing only |
| `test_season_navigation.js` | Previous/Next buttons + fresh-refetch-after-save | Not deployed -- local testing only |
| `test_tax_subsidy_price_labels.js` | Buyer/seller price correction (Section 8) | Not deployed -- local testing only |

---

## 11. Credentials & secrets

The Supabase project URL and **anon** (public) key are saved in Claude's memory for this
project, so a fresh chat shouldn't need to re-ask for them -- they're meant to be embedded
directly in client-side files exactly as they already are in both `instructor.html` and
`Burgerville.html`.

The **service role key** is not, and should never be, saved anywhere in chat memory or in any
client-side file -- it lives only as a Supabase Edge Function environment variable
(`SUPABASE_SERVICE_ROLE_KEY`, injected automatically), accessible to server-side function code
via `Deno.env.get(...)`, never to the browser.

**`INSTRUCTOR_SECRET` is now fully retired**, not just scheduled to be -- `run-current-season`
no longer checks it at all (Section 5), so this environment variable can be safely removed from
the Supabase project once confirmed nothing else still depends on it.
