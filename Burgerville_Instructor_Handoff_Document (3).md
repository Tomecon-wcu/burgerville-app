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
  "Configure Seasons," plus a "Copy access token" button for the instructor's own current
  session -- puts a one-click, no-DevTools path in front of any operational `curl` command
  (like the equilibrium-repair check below) that needs the instructor's own auth, rather than
  requiring them to dig a token out of the browser console by hand.
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
  previous season showing exactly what changed, Part-I field locking, Endowment locking for
  Part III (unlimited for a monopolist, by design -- see below), a "roll this change forward"
  option, and Previous/Next season navigation. See Section 6 for the full breakdown -- this
  screen grew substantially since the last version of this document.

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

**Also now built, since that update:** Endowment locks (disabled, visibly explained) for both
monopoly modules, not just Part I -- a monopolist is understood to be able to raise whatever
capital it takes to corner the market, so the field is meaningless there the same way
production/cost fields are meaningless for Part I, just for a different reason. Building this
surfaced a real, separate bug in the *existing* "apply to all seasons" bulk option: it
unconditionally included Endowment with no Part filter at all, meaning saving a Part III season
with that box checked would have silently overwritten every other season's Endowment, including
Part II seasons where it's a real, enforced credit limit. Fixed by excluding Endowment from the
bulk update specifically when the season being saved is Part III.

**Item 2 and item 3 from the standing to-do list are now resolved too, confirmed against the
real, live project, not just in theory:**
- `repair-equilibrium-history`'s shared-secret auth has been retrofitted to the same
  ownership-based pattern as the other three functions (Section 5), deployed, and actually run
  against `eco112-fall26` end to end. Result: `eco112-fall26` has no completed Part III seasons
  yet, so there was nothing for the Section 18.7-era corruption bug to have touched -- a clean,
  verified answer, not an assumption. Safe and cheap to rerun any time (idempotent, always
  recomputes from `season_parameters`); worth doing again once the class actually reaches a
  completed Part III season.
- `INSTRUCTOR_SECRET` is now genuinely, confirmedly safe to remove from the Supabase project --
  every function that ever depended on it has been retrofitted and proven working against the
  real deployment, not just tested locally.

Running this check also surfaced a real, reusable finding: `eco112-fall26` predates the
instructor-ownership system entirely, so its `instances.owner_id` was `null` until backfilled
by hand via the SQL editor. Any other pre-existing class connected to the instructor interface
later will likely need the same one-time fix -- see Section 8.

**Still not yet built:** the scoring-weights panel (explicitly deferred, not just delayed — see
Section 9), the results-rollup/download *view* itself (the RLS access it needs is done and
verified -- see Section 9), and Part III's own guardrail coverage — the preview currently uses
`Engine.ts`'s competitive-market functions only; `MonopolyEngine.ts`'s
`findMonopolyOptimum`/`findRegulatedOptimum` were handed off but aren't wired in yet.

**Verified by 312 automated checks across 20 suites** — the original 138 across 6 suites, 24
more across the two Run Season suites, 96 more across 8 further suites, and 54 more across 4
newest suites (the Part III endowment lock, the copy-token button, and the two auth/RLS suites
described in Sections 5 and 9). See Section 7.

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
   └─ Four Edge Functions (Deno/TypeScript, service-role, deployed to Supabase)
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
         — repair-equilibrium-history: also not owned by this track, same reasoning as above --
           the engine track's one-time data-repair function, now called via the operational
           script in Section 10 rather than the old shared secret. Retrofitted to the same
           ownership pattern, deployed, and confirmed working against the real project
           end-to-end (Section 9).
```

**Why service role for all four:** each needs to act across ownership boundaries that RLS's
per-row model can't express directly -- creating an `auth.users` row nobody has signed into yet,
reading a *source* instance's season data regardless of who owns it, scoring and advancing a
class's game state, or recomputing and overwriting historical `equilibrium_history` rows a
normal write policy was never meant to allow. All four verify the caller's own identity and
destination-instance ownership explicitly, in code, before doing anything privileged -- the
service role isn't a shortcut around authorization, it's what makes the actually-necessary
authorization check possible to enforce precisely (see each function's own file comments for
the specifics).

Everything else -- profile/class creation, season edits, roster reads -- is a direct client
query against RLS, no Edge Function needed, following the same "RLS where it's expressible,
Edge Function only where it isn't" principle as the student-side track.

---

## 4. Schema additions (this track)

Layered on top of the schema `Burgerville_Handoff_Document.md` already describes
(`instances`, `game_state`, `season_parameters`, `teams`, `participants`). Three migrations,
all already applied to the live project:

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

**`migration_results_instructor_select.sql`** (built for the results-download work, Section 9):
- Adds one instructor-owns-the-instance SELECT policy per `results_part_*` table (`results_part_i`,
  `results_part_ii`, `results_part_iii_profit`, `results_part_iii_regulation`) -- the same gap,
  and the same fix, as the participants addendum above. Checked directly against the real
  schema first rather than assumed: each table had exactly one existing SELECT policy (the
  student's own row), no instructor-facing read path at all. Verified against a real, freshly
  rebuilt Postgres instance with all three instructor migrations applied together (16 checks of
  its own, plus a full rerun of the original 27-check `test_rls.js` suite to confirm no
  regression -- see Section 7). Read-only by design: grants `SELECT` only, nothing else.

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

### `repair-equilibrium-history` (also not this track's own file)
The engine track's one-time data-repair function (Section 18.7 of the engine spec) was still on
the old shared-secret auth when this track went looking to actually run it -- confirmed by
grepping the deployed source directly rather than assumed, since `run-current-season`'s
retrofit didn't automatically imply every function got the same treatment. Retrofitted to the
identical ownership pattern, tested the same way (10 checks: missing header, invalid token,
wrong owner, missing instance, the success path), deployed, and then actually run end-to-end
against the real `eco112-fall26` -- not just verified locally. See Section 9 for what that run
found, and Section 8 for a real deployment gotcha it surfaced along the way (Supabase Edge
Functions require the entry file to be named exactly `index.ts`; a renamed download silently
either failed to deploy or left the old function running, which looked at first like the auth
patch itself hadn't worked).

Because this function is genuinely one-time/occasional rather than a regular part of an
instructor's workflow, it wasn't wired into a dashboard button the way `run-current-season`
was -- it stays a `curl`-based operational task, but now authenticated the same safe way as
everything else, via a reusable script (`repair_check.sh`, Section 10) rather than a
hand-typed, quote-fragile command.

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
   long-run adjustment applied -- or a clear visible error, directly on the dashboard row. Also
   has a "Copy access token" button, separate from the per-class controls -- copies the
   instructor's own current session token to the clipboard (with a visible fallback prompt if
   the Clipboard API is ever unavailable), so any operational `curl` command needing their own
   auth is one click away instead of a trip through the browser console.
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
   bulk option for the latter -- which itself excludes Endowment specifically when the season
   being edited is Part III, since bulk-propagating a meaningless value there would otherwise
   silently clobber real Endowment values on other seasons; Endowment additionally locks
   (independently of the Part-I lock, different reason, its own visible explanation) for both
   monopoly Parts, since a monopolist's capital is treated as unlimited; long-run/show-
   equilibrium checkboxes and, when a later season exists, a "roll this change forward"
   checkbox; a season-to-season comparison panel (previous vs. current vs. delta, unchanged
   fields shown muted, changed ones bold); season name (a brief label, with a note
   distinguishing it from the fuller text below) and shock description (with a note to make the
   message match what the comparison table above actually shows); and Save at the very bottom,
   which advances directly to the next season rather than returning to the list. Previous/Next
   season navigation buttons sit in the header, reusing rows already fetched for the comparison
   panel and equilibrium preview rather than adding new queries.

All screens share one `render()` helper that replaces a single `<div id="app">`'s contents --
no router, no framework, consistent with the student app's approach.

---

## 7. Testing infrastructure & how to rerun it

Three kinds of tests, all built and run inside this chat's own sandbox (Postgres installed
locally, Node available by default):

**RLS policy tests** (`test_rls.js`, 27 checks; `test_results_instructor_select.js`, 16 checks)
-- run against a real, freshly rebuilt Postgres 16 instance, not verified by reading the SQL.
Setup: `simulate_supabase_env.sql` (mocks `auth.users` + `auth.uid()`), the engine track's own
`schema.sql` (the real, canonical schema -- an earlier version of this document referenced a
`base_schema.sql` file that turned out not to actually exist, caught while rebuilding this exact
test environment for the newest suite), then all three migrations from Section 4, then the test
scripts. One genuine finding from this process worth remembering: Postgres RLS `UPDATE`/`DELETE`
policies filter out unauthorized rows *silently* (`rowCount: 0`), they don't throw -- a test (or
any code) checking "did this throw" instead of "how many rows changed" will pass even when
nothing was actually blocked.

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

Four more suites cover the newest work (54 checks total): `test_part_iii_unlimited_endowment.js`
(19) verifies Endowment locking for both monopoly Parts, live toggling, and -- the part that
actually matters most -- that the bulk apply-to-all correctly excludes Endowment when saving a
Part III season while still including it normally for Part II, confirmed both ways rather than
just the fixed case. `test_copy_token_button.js` (9) covers the dashboard's new token button:
copies the real, current session token (not a stale or fabricated one), reverts its label after
the confirmation, falls back to a visible prompt when the Clipboard API is unavailable, and
fails visibly rather than silently when there's no active session.
`test_repair_equilibrium_history_auth.js` (10) is the same "strip `Deno.serve`, test the logic"
treatment already given to `run-current-season`'s retrofit, applied to
`repair-equilibrium-history`'s (Section 5): missing header, invalid token, wrong owner, missing
instance, the success path, and a direct check that no trace of `X-Instructor-Secret` remains
anywhere in the source. `test_results_instructor_select.js` (16) is an RLS suite in the same
style as `test_rls.js`, run against a real, freshly rebuilt Postgres instance with all three
instructor migrations applied together -- per `results_part_*` table, confirms the owning
instructor can read, a non-owning instructor can't, the student's own pre-existing read stays
untouched, and instructors get read-only access, not write.

**312 checks across 20 suites in total** (138 original + 24 from the two Run Season suites + 96
from the eight-suite batch + 54 from the four newest suites above). None of this test
infrastructure is deployed anywhere -- it's a local verification harness, re-creatable in any
future session from the files listed in Section 10.

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
- **A bulk operation with no filter is a latent bug waiting for the right (wrong) combination
  of inputs.** The "apply to all seasons" checkbox unconditionally included Endowment with no
  Part-based exclusion at all, for as long as it existed -- harmless every time it was actually
  used, right up until an instructor might edit a Part III season (where Endowment is
  meaningless, per the notice right next to it) with that box checked, which would have silently
  overwritten every other season's real Endowment, including Part II's, where it's an enforced
  credit limit. Found while building the *unrelated* Part III locking feature, not while working
  on the bulk-apply code itself -- worth remembering that a feature touching one field can have
  side effects on a completely different, existing feature that also touches that field, and
  it's worth tracing all of them, not just the one currently being changed.
- **Retrofitting one function's auth doesn't mean every function got retrofitted.** Assumed,
  reasonably but wrongly for a while, that `run-current-season`'s auth patch meant the shared
  secret was fully retired. `repair-equilibrium-history` was a second, separate function that
  had always shared the same secret and had simply never come up again until this track went
  looking to actually use it -- confirmed by grepping the live source directly rather than
  trusting the earlier assumption. Worth explicitly checking every consumer of a piece of
  shared infrastructure before declaring it fully migrated, not just the one that prompted the
  original change.
- **Supabase Edge Functions require the entry file to be named exactly `index.ts`.** A patched
  function was downloaded and deployed without renaming it back to `index.ts` first; the deploy
  step didn't error, but the live function kept its old, unpatched behavior (returning "Not
  authorized," the pre-retrofit error text, instead of the new function's own messages) --
  which looked at first like the auth patch itself was wrong, not a filename issue. Cost a full
  extra round of debugging before the mismatch between old and new error text made the real
  cause obvious. Worth stating explicitly whenever handing off a patched Edge Function file:
  it must replace (or be renamed to) `index.ts` inside that function's own folder, not sit
  alongside it under its downloaded name.
- **A pre-existing row can be missing data a newer feature assumes every row has.**
  `eco112-fall26` was created before the instructor-ownership system existed at all, so its
  `owner_id` was `null` -- not corrupted, just never backfilled, since adding a column doesn't
  retroactively populate it for rows that predate the column. The ownership check correctly,
  safely failed closed ("You do not have permission...") rather than silently succeeding or
  crashing, which made this straightforward to diagnose and fix with one manual `update`
  keyed off the instructor's own email. Worth checking for on any other pre-existing instance
  that gets connected to the instructor interface later, rather than assuming every instance
  already has an owner.

---

## 9. Not yet built / next steps

**Resolved since the last version of this document, confirmed against the real, live project:**
- `run-current-season`'s shared-secret auth retired; the "Run Season" dashboard control is
  built and tested (Section 5).
- The pre-save equilibrium preview/guardrail, fully wired into the season edit form -- live
  price/quantity preview, degenerate-configuration confirmation, and the buyer/seller price
  distinction under tax or subsidy (Section 8). Along with it: the season-to-season comparison
  panel, Part-I field locking, the demand-shift field, Imports' billions-unit conversion,
  Previous/Next navigation, roll-forward, and the full layout restructuring described in
  Section 1.
- Endowment locking for both monopoly Parts, and the bulk apply-to-all bug it surfaced
  (Section 8).
- The dashboard's "Copy access token" button, removing the need to use the browser console for
  any operational task needing the instructor's own auth (Section 6).
- **`repair-equilibrium-history`'s shared-secret auth retired** (Section 5), deployed, and
  **actually run against `eco112-fall26`** -- not just tested locally. Result: no completed Part
  III seasons on that instance yet, so nothing for the Section 18.7-era corruption bug to have
  touched. A real, checked answer, not an assumption -- worth rerunning
  (`bash repair_check.sh`, Section 10) once the class reaches a completed Part III season, since
  this was a point-in-time check, not a standing guarantee.
- **`INSTRUCTOR_SECRET` is now safe to actually remove** from the Supabase project -- every
  function that ever depended on it (`run-current-season`, `repair-equilibrium-history`) has
  been retrofitted and proven working against the real deployment.
- **The results-download RLS gap, step 1 of 2.** A new SELECT policy per `results_part_*` table
  (instructor-owns-the-instance, identical pattern to `participants_select_instructor`) is now
  applied and verified against real Postgres (Section 7). An instructor can read their own
  students' scores through direct client queries now; step 2, the actual rollup/download view,
  is still open -- see below.

**The next real, open pieces of work, roughly independent of each other:**

1. **The results-download view itself.** The read access exists now; nothing built on top of it
   yet. The genuinely non-trivial part is the rollup: each of the four `results_part_*` tables
   has a different column set (each part's own scoring breakdown), so a class-wide "download
   results" view most likely means summing each table's own `total_points` column per student
   across however many of the four tables they have rows in, rather than trying to merge the
   detailed per-question breakdowns, which don't share a common shape across parts. "Available
   at any point" falls out naturally once the read access exists -- `results_part_*` rows only
   ever exist for seasons that have actually been scored, so a mid-semester download would
   correctly just reflect whatever's been graded so far, no separate "in progress" handling
   needed.

2. **Part III guardrail coverage.** The equilibrium preview currently only calls `Engine.ts`'s
   competitive-market functions (`solveEquilibrium`, `marketDemand`, `marketSupply`,
   `applyLongRunAdjustment`). `MonopolyEngine.ts`'s `findMonopolyOptimum` and
   `findRegulatedOptimum` were handed off in full alongside `Engine.ts` but were never wired in
   -- so a Part III (monopoly/regulation) season currently gets no live preview or
   degenerate-outcome guardrail at all, only the same basic client-side sanity checks every part
   gets. Worth deciding whether this is worth building given Part III seasons are edited less
   frequently than Part I/II, or whether the existing checks are sufficient there.

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
| `migration_results_instructor_select.sql` | Instructor results-read fix (Section 4) | Supabase SQL editor (already applied) |
| `supabase/functions/bulk-upload-roster/index.ts` | Roster pre-provisioning | `burgerville-edge` repo, `supabase functions deploy --use-api` |
| `supabase/functions/clone-season-template/index.ts` | Season setup via cloning (includes the perturbation fix, Section 5) | Same as above |
| `supabase/functions/repair-equilibrium-history/index.ts` | Section 5's retrofit -- **must be named exactly `index.ts`** inside that function's own folder when deployed, not left under a downloaded filename (Section 8) | Same as above |
| `repair_check.sh` | Reusable operational script for the repair check above -- prompts for a pasted access token rather than embedding one, sidesteps the same copy-paste quote-mangling issue `run_season.sh` (engine track) already solved the same way | Run locally, not deployed |
| `Burgerville_Instructor_Interface_Spec.md` | Original design spec for this track | Reference only |
| `test_rls.js`, `simulate_supabase_env.sql`, `schema.sql` | RLS verification harness | Not deployed -- local testing only. Note: an earlier version of this table referenced a `base_schema.sql` file that does not actually exist; the real, canonical schema (`schema.sql`, from the engine track's own `burgerville-supabase` folder) is what this harness is actually built against -- confirmed directly while rebuilding the test environment for `migration_results_instructor_select.sql`. |
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
| `test_part_iii_unlimited_endowment.js` | Endowment locking for Part III + the bulk apply-to-all exclusion fix (Section 8) | Not deployed -- local testing only |
| `test_copy_token_button.js` | Dashboard's "Copy access token" button (Section 6) | Not deployed -- local testing only |
| `test_repair_equilibrium_history_auth.js` | `repair-equilibrium-history`'s auth retrofit (Section 5) | Not deployed -- local testing only |
| `test_results_instructor_select.js` | `migration_results_instructor_select.sql`, run against real Postgres (Section 4) | Not deployed -- local testing only |

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

**`INSTRUCTOR_SECRET` is now fully retired, and confirmed clear to remove.** Both functions that
ever depended on it -- `run-current-season` and `repair-equilibrium-history` (Section 5) -- have
been retrofitted to the ownership-based pattern and proven working against the real, live
project, not just tested locally. This was previously written as "safe to remove once confirmed
nothing else still depends on it" -- that confirmation has now actually happened, so this
environment variable can be deleted from the Supabase project whenever convenient.
