# Burgerville Engine Specification (Living Document)

**Status as of this pass:** Part I market equilibrium — validated against real historical data.
Part I scoring — derived from formulas, not yet run against real student submissions.
Part II / Part III — not yet started; architecture reserved for them below.

---

## 1. Core market model (validated)

### 1.1 Demand
```
P_buyer = P_seller + Tax - Subsidy
Qd(P_seller) = Q_0 - dQ_dP * P_buyer
```
`Q_0` and `dQ_dP` are season parameters (INPUT sheet rows 11–12).

### 1.2 Individual competitive firm supply
Cobb-Douglas production, `Q = A_tech * K^Alpha_land * L^Beta_cattle`, with land `K` fixed at
`Other_land` for every "other firm." Firm chooses cattle input `L` to set MC = P_seller
(profit-maximizing, price-taking). Solving that condition for output:

```
q_firm(P) = [ (A_tech * Other_land^Alpha_land)^(1/Beta_cattle) * P * Beta_cattle / calf_price ]
            ^ (1/Beta_cattle - 1)
```

**Validated case** (A_tech=800, Other_land=30, Alpha_land=Beta_cattle=0.5, calf_price=2500):
this collapses to `q_firm(P) = 3840 * P` exactly (because the exponent `1/Beta_cattle - 1 = 1`
when Beta_cattle = 0.5). Confirmed against the workbook's own "Baseline" and "Current" firm
values in `production & cost` (B20/C20) to 5+ significant figures.

*Note: if a future version of the game changes `Beta_cattle` away from 0.5, this stops being
linear in P and the general power-law form above must be used, solved iteratively if needed.*

### 1.3 Aggregate market supply
```
Qs(P_seller) = q_firm(P_seller) * NumOtherFirms + Q_class + Imports
```
`Q_class` = total output from student/team firm decisions (always 0 in Part I; nonzero from
Part II onward).

### 1.4 Short-run equilibrium
Solve `Qd(P) = Qs(P)` for P. Linear in this dataset (closed form). The original workbook
solves this via an iterative circular-reference trick (nudge P, recheck, repeat until
`|Qd - Qs| < Q_scale`) — the new engine can just solve it directly, which is simpler and exactly
equivalent for the linear case, and only needs true iteration if a future version introduces a
nonlinear supply curve.

---

## 2. CRITICAL FINDING: firm count is persistent state, not a constant

`Other_firms_BASE` (654,038) and `Other_firms_LR` (815,412.04) in the INPUT sheet are
**snapshots of an evolving state variable**, not fixed constants. Verified by backing out the
firm count actually implied by 15 real historical seasons of recorded equilibrium prices:

| Seasons | Long-run adjustment fired? | Implied firm count | Matches |
|---|---|---|---|
| 0–9 | No (steady from game start) | ≈600,000 | Not stored anywhere in current INPUT snapshot — this was the state *before* any adjustment |
| 10.5–13 | Yes, at season 10 | ≈654,000–654,050 | `Other_firms_BASE` (654,038) |
| 15 | Yes, at season 14 | ≈815,400 | `Other_firms_LR` (815,412.04) |

**Implication for the new system:** `NumOtherFirms` must be modeled as persistent game state
that gets updated whenever a "long-run adjustment" season fires, via a zero-economic-profit
entry/exit calculation (confirmed directly by the professor: "the long run number of firms is
determined by the level of demand at the minimum average total cost divided by the output of
the firms if they operate at minimum cost").

### 2.1 Long-run adjustment formula — VALIDATED

Fires whenever a season is flagged `long_run_on = 1`, using *that season's own* current cost
parameters:

```
q_min = Other_land * A_tech * sqrt(land_rent / calf_price)        # firm output at minimum ATC
LRMC  = 2 * Other_land * land_rent / q_min                        # minimum ATC = long-run price
Q_LR  = Q_0 - dQ_dP * LRMC - Imports                              # long-run market quantity
NumOtherFirms_new = Q_LR / q_min
```

(`q_min` and `LRMC` derive from minimizing `ATC(q) = Other_land*land_rent/q + calf_price*q /
(A_tech^2*Other_land)`, which holds for `Alpha_land = Beta_cattle = 0.5`; re-derive for other
exponents if the game's technology parameters ever change.)

**Verified against every long-run transition in the 15 real historical seasons:**

| Adjustment fires at | land_rent | calf_price | Predicted firm count | Actual (stored or implied) |
|---|---|---|---|---|
| Season 10 | 3,850 | 2,500 | 646,134 | 646,153 |
| Season 10.5 | 6,000 | 2,500 | 654,038 | 654,038 (= `Other_firms_BASE`) |
| Season 14 | 6,000 | 5,000 | 1,000,618 | 1,000,618 |
| Season 15 | 6,000 | 2,500 | 815,412 | 815,412.04 (= `Other_firms_LR`) |

The firm count persists as state from the season it's computed until the next long-run-flagged
season recomputes it. This closes out Part I's market engine — it is now fully validated
end-to-end (short-run equilibrium + long-run adjustment) against all 15 real historical seasons.

---

## 3. Part I scoring rubric — VALIDATED against real student submissions

Ran the rubric below against all 103 real student submissions from a completed season
(season 5, "a border issue cuts imports"), using the archived `hard paste 1` sheet as ground
truth:

- **Categorical predictions** (Demand, Supply, Shortage/Surplus, Price/Qd/Qs direction):
  618 checks across 103 students — **100% exact match.**
- **"Ballpark" price and quantity estimates:** 206 checks — **100% exact match.**
- **Rank bonus and total score:** 92 of 103 (89%) exact match. The remaining 11 were off by
  0.02–0.06 points, and every one of them shared an *identical* accuracy score with several
  classmates — a tie-breaking difference between Excel/Sheets' `PERCENTRANK` and the
  straightforward average-rank percentile used in the first-pass validation script, not an
  error in the underlying model. Worth tightening before production if exact tie-breaking
  matters for grading, but doesn't affect confidence in the rubric.

**Part I is now fully verified end-to-end: market engine (short-run + long-run) and scoring,
both checked against real historical data rather than just read off formulas.**

Rubric, from `results I` / `feedback`:

| Component | Basis | Points |
|---|---|---|
| Q1 Demand direction | Matches `feedback` sheet's answer key for that season | 0.5 |
| Q1 Supply direction | Matches answer key | 0.5 |
| Q2 Shortage/Surplus/Neither | Matches answer key | 0.5 |
| Q3 Price direction | Matches answer key | 0.5 |
| Q4 Qty demanded direction | Matches answer key | 0.5 |
| Q5 Qty supplied direction | Matches answer key | 0.5 |
| Q6 Price estimate | Within tolerance band (below) | 0.75 |
| Q7 Quantity estimate | Within tolerance band (below) | 0.75 |
| Accuracy rank bonus | Percentile rank vs. class on combined price+quantity relative error | up to 1.0 |

**Tolerance band construction:** built from whichever curve did *not* shift that season (the
"known" curve). Round the true equilibrium quantity to the nearest billion-unit grid point,
step one grid point in each direction, and read the corresponding price off the stable curve —
plus a fixed margin of error (`Margin_of_error_P` = 0.1). Named ranges: `Pc_lower_dot`,
`Pc_upper_dot`, `Qc_lower_dot`, `Qc_upper_dot` (feedback sheet rows 63–80).

**Rank bonus formula:**
```
error = |relative Q error| + |relative P error|
rank_bonus = (1 - percentrank(error among class, this student's error)) * 100 * PTS_rank (0.01)
```

Named point weights: `PTS_basic` = 0.5, `PTS_advanced` = 0.75, `PTS_rank` = 0.01
(feedback!B48:B50).

---

## 4. Proposed architecture (built to extend into Part II / Part III)

One core module, not three separate engines:

- **`MarketState`**: season parameters (Q_0, dQ_dP, calf_price, land_rent, Imports, Tax,
  Subsidy, Floor, Ceiling) + persistent `NumOtherFirms`.
- **`solve_equilibrium(state, supply_side)`**: takes a pluggable supply-side function.
  - Part I: supply_side = many small price-taking firms only (`Q_class = 0`).
  - Part II: supply_side = many small firms **+** student-chosen firm output
    (`Q_class` = sum of team decisions).
  - Part III: supply_side = single monopolist solving MR = MC instead of price-taking —
    same demand curve, same firm cost machinery, different market-clearing rule.
- **`long_run_adjust(state)`**: fires on designated seasons, updates `NumOtherFirms` via the
  zero-profit entry/exit condition (Section 5 pending verification).
- **Scoring module**: per-part rubric (categorical + ballpark + rank bonus for Part I; will
  extend for Part II/III's added firm-decision scoring once we get there).

---

## 5. Next steps

1. ~~Verify the long-run adjustment formula reproduces the observed firm-count transitions.~~
   **Done — validated in Section 2.1.**
2. ~~Run the Part I scoring rubric against real student submission rows.~~ **Done.**
3. ~~Part II specified and validated.~~ **Done — Sections 6–7.**
4. ~~Part III, profit-maximizing monopoly, specified and validated.~~ **Done — Section 8.**
5. ~~Part III, regulation (price-at-ATC) variant, specified and validated.~~ **Done — Section 9.**

**The entire game engine — market equilibrium, long-run adjustment, and scoring for all three
parts and both Part III variants — is now fully specified and validated against real historical
data.** Natural next step: move from spec to implementation — start building Phase 1 (the
single-instance Apps Script port) from the plan discussed earlier, using this document as the
ground truth for every calculation.

---

## 6. Part II: firm decisions (validated)

### 6.1 What students submit (`submit Part II`)
Same 5 categorical market predictions as Part I, plus Market Price and Market Quantity
estimates, plus three new firm-decision fields:
- **Land Investment** — dollar amount committed to land (student's budget choice).
- **Cattle (calves)** — physical quantity of the variable input chosen.
- **Quantity of burger produced** — the output the student *claims* their choices will yield.
- **Marginal cost** — the student's *stated* estimate of their own marginal cost.

### 6.2 "Actual" firm outcome (computed from the student's own inputs — validated)
```
ActualLand   = LandInvestment / land_rent                     # dollars -> physical units
ActualOutput = A_tech * ActualLand^Alpha_land * Cattle^Beta_cattle   # Cobb-Douglas, their own inputs
ActualTC     = ActualLand * land_rent * (1+interest) + calf_price * Cattle
ActualMC     = (calf_price/Beta_cattle) * (1/(A_tech*ActualLand^Alpha_land))^(1/Beta_cattle)
               * ActualOutput^(1/Beta_cattle - 1)
ActualTR     = ActualOutput * P_current
ActualProfit = ActualTR - ActualTC
```
This recomputes what the student's *actual* land/cattle choices would really produce and cost —
independent of what they claimed on the form — which is what the scoring below checks them
against.

### 6.3 Endowment-constrained optimal benchmark ("Profit_max_player")
The maximum profit achievable by a rational firm spending its *entire* endowment on land at
the going rent, choosing cattle optimally at the current price. Same profit-maximization
formula as the market-level firm (Section 1.2), just with `Land = endowment/land_rent`
substituted for `Other_land`. Used as the 100%-of-max benchmark for profit scoring.

### 6.4 Part II scoring rubric — VALIDATED against real student submissions
Ran against all 78 real student submissions from a completed season (season 10, "Long-run
adjustments"), using the archived `hard paste II` sheet as ground truth:

| Component | Basis | Points | Validation result |
|---|---|---|---|
| 5 categorical predictions | Match answer key | 0.5 each | 390 checks — 100% match |
| Quantity estimate | Within tolerance band | 0.75 | 78 checks — 100% match |
| Price estimate | Within tolerance band | 0.75 | **See known bug, 6.5** |
| Production accuracy | \|ActualOutput − stated output\| < 5% of ActualOutput | 0.25 | 78 checks — 100% match |
| Cost accuracy | \|stated MC − ActualMC\| ≤ 2 | 0.25 | 78 checks — 100% match |
| Firm theory | \|stated price − stated MC\| ≤ 1 (tests whether student set their own numbers to P=MC) | 0.5 | 78 checks — 100% match |
| Profit points | Bracketed by % of `Profit_max_player` achieved (100%→1.0, within 1%→0.75, within 5%→0.5, within 10%→0.25, within 50%→0.1, else 0) | up to 1.0 | 78 checks — 100% match |
| Profit rank bonus | Percentile rank vs. class on % of max profit achieved | up to 1.0 | (not independently isolated; folds into total) |
Maximum ≈ 7.0 points per season (higher ceiling than Part I's 5.5, reflecting the added
firm-decision complexity).

### 6.5 Known bug in the original system — NOT to be replicated
The "Total Market Theory points" formula (`results II` column AS) checks
`ISBETWEEN(L3, Pc_lower_dot, Pc_upper_dot, ...)` — a **fixed single-cell reference** (row 3
only) instead of the range `L3:L80` every other check in that formula correctly uses. In an
array formula, this means the whole column silently inherited **one student's** price result
(row 3's) and broadcast it to every other student that season. Confirmed empirically: in the
season 10 data, every one of the 78 students received price-ballpark credit regardless of
their own estimate (values ranging from 2.0 to 120,000 all "passed") — because row 3's actual
student happened to have a price estimate inside that season's tolerance band.

**Decision (confirmed by professor):** this is an unintended bug, not an intentional design
choice, and should **not** be carried into the new system. The new engine scores every
student's price estimate against their own value, correctly, every season — including
long-run-adjustment seasons.

## 7. Architecture note: Part II confirms the modular design from Section 4
Everything new in Part II — `Q_class`, the firm-decision scoring — plugs into the same
`MarketState` / `solve_equilibrium()` core from Part I without changing it. Part III follows
the same pattern: same demand curve and cost machinery, different market-clearing rule
(monopolist's MR=MC) and different scoring components layered on top.

---

## 8. Part III, variant A: profit-maximizing monopoly (validated)

There are **two variants** of Part III in the original workbook: this profit-maximization
scenario (`submit Part III profit` / ` Results III profit`), and a separate "regulation"
scenario forcing the monopoly to price at average total cost (`submit part III regulation` /
`results III regulation`) — not yet explored (see Section 5).

### 8.1 Key structural discovery: constant marginal cost, by construction
Unlike Parts I/II — where "other firms" or the student's own firm had *fixed* land and only
adjusted cattle — the monopoly scenario lets the monopolist optimize **both** land and cattle
freely (a true long-run cost-minimization, not a short-run one). Because `Alpha_land +
Beta_cattle = 1` (constant returns to scale), this makes marginal cost **constant regardless of
output level** — and that constant equals exactly the same `LRMC` from Section 2.1
(`LRMC = 2 * Other_land * land_rent / q_min`). Verified numerically for season 13 ("calf prices
double"): the monopolist's marginal cost, computed at the true optimum, matched `LRMC` to
10+ significant figures.

This collapses monopoly profit-maximization to a clean, closed-form rule: since MC is constant,
MR = MC becomes a simple linear equation.

### 8.2 True monopoly benchmark (validated)
```
Q_monopoly = (Q_0 - dQ_dP * LRMC) / 2 - Imports
P_monopoly = Q_0/dQ_dP - (1/dQ_dP) * (Q_monopoly + Imports)        # inverse demand at Q_monopoly
Land_monopoly = (Q_monopoly / A_tech) * (calf_price/land_rent)^Alpha_land   # cost-minimizing land
Cattle_monopoly = (Q_monopoly / (A_tech * Land_monopoly^Alpha_land))^(1/Beta_cattle)
P_max = Q_0 / dQ_dP                                                # demand curve's price intercept
CS_monopoly = 0.5 * (P_max - P_monopoly) * Q_monopoly              # consumer surplus triangle
Profit_max_monopoly = P_monopoly*Q_monopoly - (Land_monopoly*land_rent*(1+interest) + Cattle_monopoly*calf_price)
```
Verified against the workbook's own "calculated" columns for real student data (season 13):
matched to full precision.

### 8.3 What students submit (`submit Part III profit`)
- **Farms** — number of standard 30-acre land units committed (monopoly-scale, so this and
  Cattle are large numbers — a monopolist supplying the whole market needs land/cattle on the
  same order of magnitude as all "other firms" combined in Parts I/II).
- **Cattle** — physical quantity of the variable input.
- **Monopoly Price**, **Marginal cost**, **Marginal revenue**, **Expected Monopoly Quantity** —
  the student's own stated predictions for their chosen farms/cattle combination.
- **Competitive Market Price** / **Competitive Market Quantity** — the student's guess at what
  the *competitive* (non-monopoly) equilibrium would be, graded with the same tolerance-band
  mechanism as Parts I/II's market-prediction ballpark checks.
- Consumer-surplus estimates at both the monopoly and competitive price/quantity.

### 8.4 "Actual" outcome from the student's own farms/cattle choice (validated)
Same style as Part II — recompute what their actual inputs produce, independent of their
stated claims:
```
ActualLand = Farms * Other_land
ActualQm   = A_tech * ActualLand^Alpha_land * Cattle^Beta_cattle
ActualPm   = Q_0/dQ_dP - (1/dQ_dP)*(ActualQm + Imports)              # price from inverse demand
ActualMC, ActualMR computed the same way as Sections 1–2, at ActualLand/ActualQm
```
Verified exactly against real student data (season 13): reproduced the workbook's own
"calculated Land sections / Qm / Pm" columns to full precision.

### 8.5 Scoring rubric — validated core, three bugs found and corrected

| Component | Intended basis | Status |
|---|---|---|
| Quantity-level ("MC=MR?") | \|(ActualQm+Imports)\| within ±10% of `Q_long_run/2` (i.e. of true `Q_monopoly + Imports`) | Validated |
| Price ballpark | Stated price within tolerance band centered on the *true* `Q_monopoly` (not the current equilibrium — this scenario's "correct" price is the monopoly optimum) | Validated |
| MC accuracy | \|stated MC − ActualMC\| within 10% | Validated |
| MR accuracy | \|stated MR − ActualMR\| within 10% | Validated |
| Competitive-price ballpark | Stated competitive price within the standard Pc tolerance band (same mechanism as Parts I/II) | Validated |
| CS-monopoly / CS-competitive accuracy | Student's stated CS within 10% of the true computed value | Validated |
| Profit points + rank bonus | Same bracket table and percentile-rank mechanism as Part II | Validated |

**Three confirmed bugs — to be corrected in the new system (confirmed unintentional):**

1. **"Price calculation points" checks the wrong field.** Original formula compares the
   calculated true monopoly price to the student's *stated marginal revenue*, not their
   *stated price* (which is what the adjacent feedback text actually checks against).
   Confirmed with real data: a student whose stated price (27.89) matched the true value
   (27.886) almost exactly scored zero, because the comparison used their unrelated MR entry
   (15.77) instead. **Fix: compare calculated true price to the student's stated Monopoly
   Price field, not their stated Marginal Revenue field.**
2. **A dead formula reference.** The "Total operating points" formula includes a `#REF!` term
   (silently absorbed by an error-handler) — a leftover link to a deleted column, most likely an
   original farms/land-accuracy scoring component that no longer does anything.
   **Fix: design this component fresh** — a farms/land accuracy check (comparing the student's
   chosen Farms count to the true optimal count within some tolerance) is the natural
   replacement, analogous to Part II's production-accuracy check.
3. **A units mismatch in the competitive-quantity ballpark check.** The original formula divides
   the student's raw quantity guess by 1,000,000,000 before comparing it to a threshold that is
   *also* in raw units (~25–30 billion) — so the comparison essentially never succeeds.
   Confirmed empirically: **0 of 68 real students** received this credit all season.
   **Fix: compare the student's raw competitive-quantity guess directly to `Qc_lower_dot` /
   `Qc_upper_dot` without the spurious division**, exactly as done for the market-quantity
   ballpark checks in Parts I/II.

---

## 9. Part III, variant B: regulation (price at ATC) — validated, zero bugs found

The second Part III scenario (`submit part III regulation` / `results III regulation`) forces
the monopoly to price at average total cost — a regulator-imposed, zero-economic-profit rule —
and grades students on how close they get to maximizing **total welfare** (consumers +
producers combined), not profit.

### 9.1 Key insight: this scenario's optimum IS the long-run competitive outcome
Pricing at ATC (= LRMC, since MC is constant under CRS — Section 8.1) and letting quantity fall
out from the demand curve reproduces **exactly** the long-run competitive equilibrium already
derived and validated in Section 2.1. Nothing new to derive — this variant reuses:
- `LRMC` and `Q_long_run = Q_0 - dQ_dP*LRMC` (Section 2.1)
- `firms_long_run = (Q_long_run - Imports) / q_min` (the *same* long-run firm-count formula,
  just interpreted here as the optimal number of farms for a single regulated monopolist)
- `CS_Long_run_competition = 0.5 * (P_max - LRMC) * Q_long_run` — and since profit is zero at
  the true optimum, this also equals **maximum total welfare**.

### 9.2 What students submit (`submit part III regulation`)
Farms, Cattle, Expected Quantity, stated Marginal Cost, stated Marginal Revenue, stated Price,
stated Consumer Surplus. (No competitive-price/quantity guess in this variant — simpler form
than the profit variant.)

### 9.3 "Actual" outcome from the student's own farms/cattle choice — validated
```
ActualLand  = Farms * Other_land
ActualQ     = A_tech * ActualLand^Alpha_land * Cattle^Beta_cattle
ActualPrice = max(0, Q_0/dQ_dP - (1/dQ_dP)*(ActualQ + Imports))     # read off demand curve
ActualMC    = same Cobb-Douglas MC formula, at ActualLand/ActualQ
TFC = ActualLand * land_rent * (1+interest);  TVC = Cattle * calf_price
Profit  = ActualQ*ActualPrice - (TFC+TVC)
ActualCS = 0.5 * ActualQ * (P_max - ActualPrice)
Welfare  = Profit + ActualCS                                        # total surplus
```

### 9.4 Scoring rubric — VALIDATED against all 80 real students, zero mismatches
Ran against every real student submission from season 15 ("All Monopolies Price at ATC"),
using the archived `Copy of results III regulation` sheet as ground truth. **Every component —
ActualQ, ActualPrice, Profit, ActualCS, Welfare, and all four scoring components below,
including the grand total — matched exactly for all 80 students. No bugs found in this
variant.**

| Component | Basis | Points |
|---|---|---|
| Quantity level | `ActualQ + Imports` within ±1 grid unit of `Q_long_run` (grid = 1 billion) | 0.75 |
| Price level | `ActualPrice` within tolerance band centered on `Q_long_run`'s grid point, ±0.1 | 0.75 |
| Consumer surplus accuracy | `ActualCS` within 10% of `CS_Long_run_competition` | 0.75 |
| Welfare points | Bracketed by % of max welfare achieved (`Welfare / CS_Long_run_competition`) — same bracket table as Part II's profit points | up to 1.0 |
| Rank bonus | Percentile rank vs. class on total welfare achieved | up to 1.0 |

Maximum ≈ 4.25 points per season. Note: farms accuracy (X/Y columns) is displayed as feedback
but is **not** a separately scored component in this variant — confirmed this is simply how it
was designed (unlike the profit variant's dead `#REF!`, there's no evidence of a deleted
column here), so no fix needed.

---

## 10. Summary: engine specification complete
All three parts of Burgerville, and both Part III variants, are now fully specified and
validated against real historical student data:
- Part I: market engine (short-run + long-run) and scoring — validated.
- Part II: firm-decision engine and scoring — validated, one bug found and corrected (6.5).
- Part III (profit): monopoly engine and scoring — validated, three bugs found and corrected (8.5).
- Part III (regulation): welfare-maximization engine and scoring — validated, zero bugs found (9.4).

This document is the ground truth for implementation. Next step: begin Phase 1 (single-instance
Apps Script port) from the earlier discussion, porting each validated formula in this document
directly into code.

---

## 11. Instructor configuration (guided form) — design direction

Per-instance parameters are organized in three tiers, exposed through a guided form rather than
a raw parameter table:

1. **Game-level setup** (once, at instance creation, locked once the first season starts):
   `A_tech`, `Alpha_land`, `Beta_cattle`, `Other_land`, `endowment`, `interest`. Defaults to the
   validated historical values from this document.
2. **Season-by-season shocks** (the main guided flow): instructor picks a shock type first
   (demand shift / supply shift / cost change / trade shock / government intervention /
   long-run adjustment / monopoly round), and only the relevant fields for that type are shown.
   This selection is what populates the "which curve shifted" flag driving the tolerance-band
   logic (Section 3) — the instructor never needs to know that mechanism exists.
3. **Scoring tuning** (optional, collapsed by default): point weights, tolerance percentages,
   rank-bonus weight.

**Guardrail:** before a season can be activated, it's run through the engine (`solveEquilibrium`
+ a "preview" pass) and blocked if the result is degenerate (negative price, non-convergent
long-run adjustment, etc.), with a plain-language explanation rather than a formula error.

**Important consequence for implementation:** because instructors can change `Alpha_land` /
`Beta_cattle` away from 0.5, the *simplified* formulas validated in Sections 1.2 and 8.1 (which
collapse nicely only at Beta_cattle=0.5, and only reach constant marginal cost when
Alpha_land+Beta_cattle=1) must NOT be ported as-is. Implement the general power-law forms
(solved numerically where needed) and confirm they reduce to the validated historical results
as a special case at Alpha_land=Beta_cattle=0.5.

## 12. Deferred: price floors and price ceilings — NOT built in the original system
Your original structure doc lists "Market interference (Taxes/Subsidies, price Ceilings and
Floors)" as intended Part I content, but **this was never actually implemented** in the
workbook — Floor/Ceiling exist as input columns but are unused throughout all 15 real historical
seasons. There is no historical data to validate against, and the mechanic requires genuinely
new logic (detecting whether an imposed price binds, and if so switching from "solve for
clearing price" to "quantity = min(Qd,Qs) at the fixed price"), plus a new scoring rubric for
shortage/surplus-under-a-binding-control predictions.

**Decision: deferred.** Phase 1 ships the validated curriculum as actually run (no floors/
ceilings). This is a candidate follow-on feature after the multi-instance system is working,
not part of the initial port.

---

## 13. Phase 1 progress: implementation underway

### 13.1 Correction to Section 3's rank-bonus validation
The original validation (Section 3) found 89% exact match on the rank bonus, attributed to
"tie-breaking." Building the real implementation surfaced the actual cause: the validation
script approximated Google Sheets' `PERCENTRANK` using average-rank-among-ties, which is not
what `PERCENTRANK` actually does. The correct formula is:

```
percentRank(values, x) = (count of values strictly less than x) / (n - 1)
```

Implemented this way in `ScoringPartI.gs`, all 103 real students match exactly, including
every student who was tied with classmates on accuracy score. This is a strict improvement
over the 89% originally reported.

### 13.2 Files built so far

- **`Engine.gs`** — core market engine (Sections 1–2), implemented with the *general*
  power-law formulas rather than the Beta_cattle=0.5 shortcuts, so it stays correct if an
  instructor changes the technology parameters (per Section 11). Validated against all 11
  real non-long-run historical seasons and all 4 real long-run adjustment transitions.
- **`ScoringPartI.gs`** — Part I scoring (Section 3), with one genuine architectural
  improvement over the original workbook: every categorical answer (demand/supply shift,
  shortage/surplus, price/quantity direction) is **derived directly from the engine's own
  before/after equilibria**, not read from a hand-written answer-key table. This means the
  system grades correctly for *any* instructor-configured scenario automatically, with
  nothing to keep in sync by hand. Validated against all 103 real students from season 5.
- **`SheetSetup.gs`** — creates the full table schema (GameState, SeasonParameters, Teams,
  Participants, and per-part Submissions/Results tables) as Sheet tabs, with `instance_id` on
  every table from day one (the seam Phase 2 will use). One function to run, no code editing
  required for the instructor.
- **`Burgerville_Scenario_Tester.html`** — a no-code interactive tool (separate from the
  deployed app) for exploring what the engine produces under different market conditions,
  with real historical seasons loadable as presets for self-verification.

### 13.3 Still to build
- Scoring for Part II, Part III (profit), Part III (regulation) — porting Sections 6, 8, 9
  the same way, each independently validated against real historical data before being
  considered done.
- The instructor configuration screen (Section 11).
- The "run the season" orchestration function (close submissions → score → apply long-run
  adjustment if flagged → advance season → log results).
- Market-info, input-form, and results/leaderboard screens for each part.

### 13.4 Part II implementation — validated

**Files added:** `MarketPrediction.gs` (the shared answer-key-derivation and tolerance-band
logic factored out of `ScoringPartI.gs` so Part I and Part II use exactly one copy — no risk
of the two drifting apart), `ScoringPartII.gs`.

**Validated against all 78 real students from season 10** ("Long-run adjustments"): every
categorical prediction, both ballpark estimates, production accuracy, cost accuracy, firm
theory, and profit points matched exactly (66/78 identical to the historical record to the
last cent; the other 12 differ only by a few dollars of floating-point noise in the profit
calculation itself — never enough to cross a scoring-bracket threshold, so every student's
actual awarded points still match).

**Confirmed fixed:** the price-ballpark bug (Section 6.5) is structurally impossible now —
credit is based on each student's own row, and in this test run only 52 of 78 students
received it (versus the original bug giving it to all 78 regardless of their answer).

**A second lesson learned, worth knowing for Part III:** Part I's form phrases "unchanged" as
"no change" and price direction as "rise/fall." Part II's form phrases the same underlying
concepts as "not change" and "increase/decrease." Rather than hardcoding either part's exact
wording, `normalizeAnswer()` in `MarketPrediction.gs` treats these as synonyms, so scoring
doesn't depend on which convention a given part's form (or a future instructor-customized
form) happens to use.

**Handling a season where the long-run adjustment itself is part of the shock:** season 10
is both a market-prediction season *and* a long-run-adjustment season, meaning the "correct"
answer for whether supply increased depends on comparing the *old* firm count to the *new,
post-adjustment* firm count. `deriveMarketAnswerKey()` was generalized to accept separate
old/new firm counts to handle this correctly — validated: the derived answer key (`supply:
increase`) matches the real historical answer key exactly.

### 13.5 Part III implementation — both variants validated

**Files added:** `MonopolyEngine.gs` (shared by both variants), `ScoringPartIIIProfit.gs`,
`ScoringPartIIIRegulation.gs`.

**A generalization was needed here, not just a port.** The spec's closed-form monopoly
formulas (Sections 8.2, 9.1) only work because Alpha_land + Beta_cattle = 1 in the historical
data — constant returns to scale is what makes minimum average cost independent of firm size,
letting a small fixed-land firm's cost curve stand in for a monopolist choosing land and
cattle freely at whatever scale it wants. `MonopolyEngine.gs` replaces both closed forms with
general numeric solvers:
- `findMonopolyOptimum` — maximizes profit over quantity, with land and cattle both chosen to
  minimize cost at every candidate quantity (ternary search, same technique as `findMinATC`).
- `findRegulatedOptimum` — finds the quantity where price equals average cost (the regulator's
  break-even rule), again with land and cattle both free.

Both were checked against the validated historical numbers (season 13 for profit-mode, season
15 for regulation) and matched to within numerical precision, then used for the real
class-of-real-students validation below — confirming the generalization is correct, not just
theoretically appealing.

**Profit-mode validated against all 68 real students from season 13** ("calf prices double"):
every non-buggy component (calculated quantity/price/MC/MR, quantity-level points, MC/MR
points, competitive-price points, both CS-accuracy points, profit points, rank points) matched
exactly. All three confirmed bugs (Section 8.5) verified fixed:
1. Student #1013061's price-calculation points, previously 0 due to comparing against their
   stated marginal revenue, now correctly scores 0.5 against their stated price.
2. Farms/land accuracy (previously a dead `#REF!`, contributing nothing) now scores 13 of 68
   students within the tolerance band.
3. Competitive-quantity ballpark (previously 0 of 68 all season due to a spurious division)
   now correctly credits 2 of 68 students based on their own guess.

**Regulation-mode validated against all 80 real students from season 15**: every component —
quantity, price, consumer surplus, welfare, and all four scoring components including the
grand total — matched exactly, confirming the general solver is a fully faithful (not just
approximately equivalent) replacement for the CRS-specific formulas.

### 13.6 Engine implementation: complete
Every part of Burgerville — market engine, long-run adjustment, and scoring for Parts I, II,
and both Part III variants — is now built as real, tested Apps Script code, not just specified
on paper. Next: the "run the season" orchestration, and the market-info/input-form/results
screens for each part.

### 13.7 Season orchestration — validated end to end

**File added:** `RunSeason.gs` — the single function (`runCurrentSeason`) that replaces
"the professor manually runs the simulation and pastes results." It reads the current
season's submissions, figures out which part applies, scores the whole class with the right
engine, applies a long-run adjustment if this season is flagged for one, writes every result
to the permanent log, and advances `GameState` to the next season.

**Tested two ways, both against real historical data, using an in-memory mock of the Sheet
backend** (so the test doesn't require a live Google Sheet, but exercises the exact same
`readTable`/`appendRow` interface the real deployment will use):

1. **Simple case** — season 5 (Part I), real submissions from all 103 students. Result: all
   103 total-points values matched the historical record exactly, and `GameState` correctly
   advanced to season 6 with the right price, quantity, and firm count.
2. **Harder case** — season 10 (Part II), where a long-run adjustment fires *in the same
   season* being scored. Result: the firm count updated to ~646,134 (matching Section 2.1's
   validated transition), all 78 students were scored, and `GameState` advanced correctly.

This closes out the orchestration layer. What's left for Phase 1: the market-info,
input-form, and results/leaderboard screens for each part (the actual instructor is currently
still using a script-run rather than a Sheet-driven UI). Everything underneath those screens —
engine, scoring, orchestration — is now built and validated.

### 13.8 Market info display — ported to real deployment

**Files added/replaced:** `MarketInfoDisplay.gs` (rewritten), `MarketInfoDisplay.html` (new —
the real Apps Script client, replacing the standalone preview).

Everything validated interactively in the preview (season browsing with server-enforced
locking, the interactive prediction sliders, the equilibrium-anchored axis scheme, the
historical replay with shortage/surplus arrows, the config-driven input form, the
`show_predicted_equilibrium` difficulty toggle) is now wired to real `google.script.run` calls
instead of hardcoded demo data.

**Two things worth flagging about this port specifically:**

1. **Season locking is enforced server-side, not just hidden in the UI.** `getMarketInfoData`
   checks the requested season against `GameState.current_season` and throws if it's in the
   future. This matters because Apps Script functions can be called directly from browser dev
   tools regardless of what buttons are rendered — client-side hiding alone would not have
   been a real security boundary.
2. **The pending season's response never includes its own shock parameters.** For the
   currently-active season, the server returns only the *current, already-observable* baseline
   (safe — it describes the market as it stands, not the hidden shock) plus a magnitude-hint
   *string*, computed server-side from the real upcoming parameters but never sending those
   parameters themselves to the browser. Completed seasons are fully revealed, parameters
   included, since that's already-graded history.

**Validated with a full integration test** — the real server file and the real client file,
loaded together and wired through a mocked `google.script.run` that calls the actual
sandboxed server functions (not a stand-in): confirmed the pending season correctly hides
the shock's numbers while surfacing an accurate hint, navigating to a completed season
correctly reveals both endpoints, and requesting a locked future season is correctly rejected
by the server.

**Still simplified, flagged for later:** participant identity currently just uses
`Session.getActiveUser().getEmail()` matched against the `Participants` table. This works for
a Google Workspace deployment where students are logged in, but a proper
login/enrollment flow hasn't been designed yet — worth revisiting once the results/leaderboard
screen (which also needs to know "who is asking") gets built.

### 13.9 Part-grouped navigation, and input form backend confirmed for all parts

**Part-grouped season display**, implemented in `MarketInfoDisplay.gs`/`.html`:
- `getPartsOverview()` derives part boundaries directly from `SeasonParameters` — however many
  consecutive rows share a part code (`I`, `II`, `IIIProfit`, `IIIRegulation`, grouped under one
  "Part III" heading for navigation since they're one part of the curriculum) is that part's
  length. An instructor changing a part's season count is just adding or removing rows; nothing
  else needs to change. This directly satisfies "professors should add or subtract seasons from
  each part" from the discussion that prompted this.
- No half-integer "transition" seasons (the original workbook's 5.5/10.5) are needed — the part
  boundary is just wherever the `part` code changes between consecutive season numbers.
- The season selector now shows one "pill" per part (completed / active / future), each
  expanding to that part's season buttons. A completed part shows its *entire* range,
  regardless of current season — satisfying "navigate backward to completed parts." A future
  part's pill has no click handler at all — satisfying "can't move forward to parts that are
  inactive." Validated with `test_parts_integration.js`: confirmed pill states, confirmed
  clicking a completed part reveals its full range, confirmed a future part is a genuine no-op.

**Input form backend confirmed working for all four parts**, not just Part I. The client's
`partFormConfigs` (one config per part, field names matching the real `Submissions_*` columns
exactly) and the server's `submitPrediction` (generic across parts, keyed by `formData.part`)
were both already in place. Validated with a real end-to-end test
(`test_parts_form_submission.js`) that loads a pending season for each of Part II, Part III
(profit), and Part III (regulation), fires the actual submit button's click handler, and
confirms the submission lands correctly in `Submissions_PartII`, `Submissions_PartIII_Profit`,
and `Submissions_PartIII_Regulation` respectively.

### 13.10 Results & leaderboard screen — built and validated

**Files added:** `ResultsDisplay.gs`, `ResultsDisplay.html` — a separate screen from the market
info display, per the professor's preference.

**Design confirmed with the professor before building:**
- Separate screen (not a section on the market info page).
- Team leaderboard shows both this season's standing and a cumulative (season-to-date) total.
- Individual scores stay private — a student sees only their own breakdown and their own
  cumulative total, never another individual's score or rank by name. Team-level standings are
  the only inter-student comparison shown, and those are visible to everyone.

**How the numbers are computed:**
- A team's score for a given season = the average `total_points` among that team's members who
  submitted that season (matches the original game's "team's average points are totaled"
  design). A team with zero submissions in a season is excluded from that season's leaderboard
  entirely, not penalized with a zero — a design choice worth revisiting if it ever seems to
  reward non-participation.
- A team's cumulative score = the sum of its per-season averages across every graded season so
  far (again matching "totaled" from the original design).
- An individual's cumulative score = the sum of their own `total_points` across every season
  they've submitted, read across whichever Results table each season's part maps to (a student
  who has played through Part I and into Part II has their total pulled from both
  `Results_PartI` and `Results_PartII` automatically).
- Score-component labels are part-specific (e.g., Part I shows "Shortage / surplus," Part II
  swaps that for "Firm theory (set price = marginal cost)," etc.) and reuse the exact column
  names validated throughout Sections 3/6/8/9 — nothing new to keep in sync.

**Validated two ways:** a direct server-function test (`test_results_display.js`) covering the
season/part grouping, one student's full component breakdown, the season and cumulative team
leaderboards, and a correctly-rejected request for an ungraded season; and a full integration
test (`test_results_integration.js`) loading the real client script against the real server
functions through a mocked `google.script.run`, confirming the page renders correctly on load
and after navigating to an earlier season.

### 13.11 Phase 1 status: all four screens built
Market info display, input forms (all four parts), season orchestration, and now results/
leaderboard are all built, wired to real Apps Script functions, and validated against realistic
multi-season, multi-part data. What remains before a live pilot: a full dry run through actual
Google Sheets (this conversation's validation has used an in-memory mock of the Sheets backend
throughout — functionally equivalent, but an end-to-end run against the real Sheet API is the
natural final check), and the instructor configuration screen discussed in Section 11, which
hasn't been built yet.

## 14. Supabase migration (moving off Google Apps Script)

Following a discussion about wanting a standalone system not tied to Google Sheets/Workspace
login, and a comparison against Convex (Convex is TypeScript-native with genuine real-time
reactive sync, but is a document database, not relational — a poor fit for this project's
aggregation-heavy, tabular data model, and offers no real advantage since the app has no
actual requirement for live cross-client push updates), the migration target is
**Supabase (Postgres) + Edge Functions + static hosting**.

### 14.1 Database schema — built and genuinely tested, not just written

**File: `schema.sql`.** Every table from `SheetSetup.gs` ported to proper Postgres types, with
foreign keys, unique constraints, and Row Level Security (RLS) policies. `instance_id` is on
every table, same as always — but now with Postgres RLS actually available to enforce
per-instance and per-student isolation properly, this is the seam that makes genuine
multi-instance support (the original goal of this whole project) realistic to build next,
rather than a column that's present but unenforced.

**Design principle: RLS is defense in depth, not the only line of defense.** The primary
business logic (season locking, secrecy of upcoming shock parameters, scoring) still belongs
in server-side functions (Edge Functions, replacing the Apps Script `.gs` files) — same
separation of concerns as `MarketInfoDisplay.gs`/`ResultsDisplay.gs`/`RunSeason.gs` always had.
RLS backstops that: even if a client queried the database directly, bypassing the Edge
Function, it still could not see another student's scores or a future season's parameters.

**Validated by actually running it**, not by inspecting the SQL: installed Postgres locally,
built a small simulation of Supabase's environment (`simulate_supabase_env.sql` — an `auth`
schema, `auth.uid()`, the `anon`/`authenticated` roles), seeded realistic test data, and wrote
`test_rls.js`, which connects as the unprivileged `authenticated` role (not the superuser or
table owner, which would bypass RLS entirely and make the test meaningless) and simulates two
different students' JWTs.

**This caught three real bugs before they would have shipped**: `instances`, `game_state`, and
`teams` all had RLS *enabled* but no policy defined, which defaults to denying ALL access —
including the subqueries other policies depend on (e.g., "is this the current season?"). The
season-locking and pending-season-secrecy policies were silently failing closed (denying
everything) because of this, not because their own logic was wrong. Fixed by adding explicit
`for select using (true)` policies to all three (safe, since none of that data is sensitive on
its own), then rebuilt the entire database from scratch from the corrected file and re-ran all
10 tests clean.

**All 10 tests passing, covering:**
- A student can see their own results; querying a classmate's results by ID returns zero rows.
- Season parameters: graded seasons' full parameters are visible; the current (pending)
  season's are not, via direct table access.
- The team leaderboard view exposes only aggregates (team averages) for every team, never an
  individual's row or `participant_id`.
- A student can submit a prediction for the current season.
- A student CANNOT submit for an already-closed season (RLS rejects the insert).
- A student CANNOT submit while impersonating another participant's ID (RLS rejects the insert).

### 14.2 What's next
- Port `Engine.gs`, `MarketPrediction.gs`, `MonopolyEngine.gs`, and the four `ScoringPart*.gs`
  files — these have zero Google-specific dependencies already (validated throughout this
  project by testing them in plain Node.js), so this is a rename, not a rewrite.
- Port `RunSeason.gs`, `MarketInfoDisplay.gs`, `ResultsDisplay.gs` to Supabase Edge Functions —
  the business logic carries over directly; only the `readTable`/`appendRow` data-access calls
  need replacing with Supabase client calls.
- Replace `google.script.run` calls in the two HTML files with `fetch()`/`supabase-js` calls.
- Set up Supabase Auth (magic link, to support classes not on Google Workspace).

### 14.3 Edge Functions — ported and validated

**Files added** (under `supabase/functions/`):
- `_shared/*.ts` — the engine and all four scoring modules, mechanically converted to ES
  modules (adding `export`/`import` statements) with **zero logic changes**. Confirmed by
  re-running the exact historical validations from Phase 1 against the converted files: all 4
  long-run adjustment transitions, all 103 real Part I students, all 78 real Part II students,
  plus smoke tests confirming Part III's more complex import chains resolve correctly.
- `get-parts-overview/` — replaces `getPartsOverview()`. Needs the service-role key specifically
  because RLS hides *future* seasons' rows entirely, but this function must reveal that a
  future part exists and how many seasons it spans (safe metadata) so the season selector can
  render a locked "Part III" pill before Part III has started.
- `get-market-info/` — replaces `getMarketInfoData()`. Same secrecy design as before, now
  proven at the level that actually matters: a direct test confirms the pending season's JSON
  response **never contains its own secret parameter value anywhere in the payload**, even
  though the function correctly used that value internally to compute the magnitude hint.
- `run-current-season/` — replaces `runCurrentSeason()`. Validated against the same two real
  scenarios from Phase 1's `RunSeason.gs` testing: season 5 (103 real Part I students, exact
  historical match) and season 10 (78 real Part II students, with a long-run adjustment firing
  in the same season being scored, firm count landing within 0.1% of the validated ~646,134
  transition).

**Testing methodology, and an honest limitation:** Deno itself isn't reachable from this
environment (network restrictions), so the actual Deno runtime and the real `Deno.serve` HTTP
layer haven't been executed here. What *has* been tested is the actual business logic inside
each function — deliberately factored out into plain, dependency-free functions
(`buildPartsOverview`, `buildMarketInfoResponse`, `runSeasonLogic`) that were extracted and run
directly in Node, the same rigor applied to every other piece of this project. The thin
Deno-specific wrapper around each (`Deno.serve`, `createClient`, service-role vs. anon-key
usage) is written to standard, documented Supabase conventions, but is mechanical enough that
the real risk was always in the logic, not the plumbing — which is exactly what got tested.

**A architecture simplification worth noting:** not everything needs an Edge Function. Reads
that RLS can fully express as a row filter — a student's own results, the team leaderboard
views, submitting a prediction — can go directly from the client to Postgres via `supabase-js`,
with RLS as the enforcement, no server function needed at all. Only the three genuinely
special cases above (locked-part metadata, upcoming-season secrecy, and season orchestration)
need real server-side logic. This is a smaller Edge Function surface than a literal 1:1 port of
every Apps Script `.gs` file would have needed.

### 14.4 What's left in the Supabase migration
- Update the two HTML clients (`MarketInfoDisplay.html`, `ResultsDisplay.html`) to call these
  Edge Functions and direct Supabase queries instead of `google.script.run`.
- Set up Supabase Auth (magic link) and wire `getCurrentParticipant`-equivalent logic (now just
  a direct RLS-protected query against `participants`, no Edge Function needed).
- Replace the placeholder `INSTRUCTOR_SECRET` header check in `run-current-season` with a real
  instructor-role system before handling more than one instructor.
- An actual deployment: `supabase init`, `supabase functions deploy`, and a real dry run against
  live Supabase (this environment could validate the schema and logic thoroughly, but a real
  Supabase project is the natural final check, the same way a live Google Sheet was flagged as
  the final check for the Apps Script version).

### 14.5 HTML clients ported to Supabase — the migration is now functionally complete

**Files:** `MarketInfoDisplay.html`, `ResultsDisplay.html` (in `burgerville-supabase-clients/`),
every `google.script.run` call replaced.

**Auth added** (genuinely new, not present in the Apps Script version): both pages now gate on
Supabase magic-link sign-in — an email field, a "send sign-in link" button, and
`onAuthStateChange` showing/hiding the app accordingly. This is what actually delivers the
"share with classes not on Google Workspace" goal from earlier in this conversation; any email
address works, no institutional login required.

**What talks to what, concretely:**
- `MarketInfoDisplay.html` calls the `get-parts-overview` and `get-market-info` Edge Functions
  (need server-side logic: revealing locked-part metadata, preserving upcoming-season secrecy).
  Submitting a prediction and reading your own roster row are **direct Supabase queries** — no
  Edge Function needed, since RLS alone already enforces "own row only" and "current season
  only" (validated in `test_rls.js`).
- `ResultsDisplay.html` uses **zero Edge Functions** — every read (results overview, a season's
  breakdown, both leaderboards, cumulative standings) is a direct query, relying entirely on
  season_parameters' graded-only RLS policy and the team-leaderboard views' safe aggregation.

**Validated functionally**, the same way as everything else in this project — not just visually
inspected:
- `test_supabase_client.js`: confirms `get-market-info` calls are shaped correctly (right URL,
  JWT passed as a Bearer token, right body), and — importantly — that the submit handler's
  insert uses `participant_id` from the authenticated session (`auth.getUser()`), never from
  client-supplied form data, so a student can't spoof who a submission is attributed to even
  before RLS gets a chance to check.
- `test_resultsdisplay_supabase.js`: re-derives the exact same cumulative numbers validated
  against the original Apps Script version (ANT: 41.0, BEE: 14.5) using a mock that faithfully
  simulates the season_parameters RLS filtering already proven correct in `test_rls.js`, so this
  test is checking the client logic against a realistic simulation of the real security
  boundary, not a simplified stand-in.

### 14.6 Supabase migration: what's left
- Fill in `SUPABASE_URL` / `SUPABASE_ANON_KEY` in both HTML files once a real Supabase project
  exists (currently placeholder values).
- Deploy: `supabase init`, `supabase db push` (the schema), `supabase functions deploy` (all
  three functions), host the two HTML files anywhere static (Vercel, Netlify, GitHub Pages).
- Replace the placeholder `INSTRUCTOR_SECRET` check in `run-current-season` before more than
  one instructor uses this.
- A real end-to-end dry run against live Supabase — everything so far has been validated
  against faithful local simulations (a real local Postgres with RLS for the schema, mocked
  Supabase clients built to match validated real behavior for the app logic), which is strong
  evidence the pieces are individually correct, but seeing the whole system work together for
  real is the natural final check, same as flagged for the Apps Script version's Google Sheets
  dependency.

## 15. Live deployment — the system is running, in a real classroom, for real

Following the schema/Edge Function/client migration in Section 14, the system was actually
deployed to real infrastructure (not just tested against local simulations) and Season 1 was
run against real student submissions. **This section documents the deployment itself and
everything that surfaced along the way** — several real bugs were caught here that no amount
of local testing against simulated environments would have found, since they only manifest
when actual third-party infrastructure (email delivery, institutional spam scanners, browser
CORS enforcement) gets involved.

**Live infrastructure, as of this writing:**
- Supabase project: `oxhojojzwvnfzsfpybyf` (organization: Tomecon-wcu)
- Instance ID: `eco112-fall26`
- Static hosting: GitHub Pages, `tomecon-wcu.github.io/burgerville-app/`
- Custom domain for transactional email: `funconomics.org`, verified with Resend

### 15.1 Bugs caught only by real-world deployment, not local testing

**CORS was missing entirely from all three Edge Functions.** Every function returned responses
with no `Access-Control-Allow-Origin` header and no handling of the browser's preflight
`OPTIONS` request. This is invisible in Node-based logic testing (there's no browser enforcing
CORS in a test harness) and only surfaced as an opaque "Failed to fetch" in the actual browser.
Fixed by adding shared CORS headers and an `OPTIONS` short-circuit to all three functions.

**The OTP code length was hardcoded to 6 digits in the login UI** (`maxlength="6"`), but
Supabase's actual OTP length is a per-project setting (`GOTRUE_MAILER_OTP_LENGTH`) that isn't
always 6 — this project's happened to be 8, silently truncating every code the user typed and
making login impossible. Fixed by removing the hardcoded assumption entirely (both the
`maxlength` and the "6-digit" wording) rather than trying to match whatever the current project
setting happens to be.

**Supabase's default email sender has a very low quota** (2 emails/hour on the free default,
raised to 30/hour with any custom SMTP configured) — intended only for early testing, and
immediately exhausted by real multi-student sign-in activity. This is also where the
institutional email link-scanning problem (documented back in Section 13, "Auth" discussion)
would have recurred at classroom scale: university security scanners pre-fetching magic links
and burning the one-time token before the real user clicks it. **Fixed on two fronts**: (a)
switched login from a clickable magic link to a typed one-time code (`verifyOtp`), which
scanners can't pre-consume since there's no link to click; (b) configured custom SMTP through
Resend, using the professor's own verified domain (`funconomics.org`), removing the low default
quota entirely. Notably, sending to arbitrary student recipients requires a *verified domain*
with a transactional provider — services that only let you send to your own account's address
without domain verification (several were considered) would not have worked for this use case.

**A subtle, security-relevant RLS bug in the resubmission feature.** When "allow resubmission
until the deadline" was added, the natural implementation — a `season_is_open()` function
checking the deadline — silently failed open: the same RLS policy that hides the *current*
season's parameters from students (to keep shock values secret) also hid the deadline from this
check when it ran with the student's own privileges, so "can't see the deadline" was
misread as "no deadline, always open." Caught by direct testing against a real deadline
value, not by reading the SQL. Fixed by making that one narrow check `SECURITY DEFINER` (it
only ever returns a yes/no boolean, never leaking any of the actual secret parameters it reads
internally to compute that answer).

**Two pieces of CSS were silently dropped when the two HTML pages were merged into one** during
the "reduce to a single sign-in" request — `.grid` (the side-by-side card layout) and the whole
Results-screen rule set (`.component-row`, `.total-row`, `table.leaderboard`, etc.). The page
still rendered and functioned, just with unstyled, run-together text on the Results tab. A
reminder that a functional/logic test passing (which the merge's own test suite did) doesn't
guarantee visual completeness — CSS carries no equivalent automated check in this project.

### 15.2 Resubmission + confirmation email (built after initial deployment)

Two follow-up features, requested once the system was live:
1. **Students can resubmit (overwrite) their prediction any time before the season's actual
   deadline**, not just while the season happens to still be "current." Implemented via
   `season_is_open()` (see the SECURITY DEFINER note above) plus a matching `UPDATE` RLS policy
   alongside the existing `INSERT` one, and a client-side upsert instead of a plain insert.
   Migration file: `migration_allow_resubmission.sql` (safe to run against a live database with
   existing data — adds policies and a function, touches no rows).
2. **A confirmation email is sent after every successful submission**, listing exactly what was
   recorded, via a new Edge Function (`submit-prediction`) that upserts the row and then calls
   Resend's API directly (not through Supabase Auth's email system, which only handles auth
   events). Email failures are logged but never fail the submission itself — the database write
   is what matters; a missing confirmation email is a lesser problem than losing a valid
   submission over a mail provider hiccup.

Both validated with the same rigor as everything else in this project: a genuine RLS test suite
(rebuilding the local Postgres database from the schema file and testing as an unprivileged
role, not the superuser) confirming resubmission actually overwrites rather than erroring, and
that both insert and update are correctly rejected once the deadline passes; and a client-side
test confirming the submit handler calls the new Edge Function with the right payload and that
the participant ID comes from the authenticated session, never from client-supplied data.

### 15.3 Current status
Season 1 has been run end-to-end against real student submissions (`run-current-season`,
triggered via the documented `curl` command), and the Results screen correctly displays that
season's breakdown, team leaderboard, and cumulative standings. The system is live and usable
for a real class today. **What's still needed to run the full semester smoothly**: adding each
subsequent season's `season_parameters` row before its deadline (a manual SQL step — this is
exactly the gap the instructor-configuration screen, discussed next, would close), and
eventually replacing the placeholder `X-Instructor-Secret` header check with a real
per-instructor authentication system if this is ever used by more than one instructor.

## 16. Next planned work: instructor configuration screen
Not yet started. The professor's stated intent is to build a guided interface for setting up
seasons (currently done by hand in Supabase's SQL Editor) — this is the same "Section 11:
Instructor configuration (guided form)" design direction sketched early in this document,
now to be built against the live Supabase schema rather than the original Apps Script/Sheets
design. See the Hand-off Document for a concrete starting point.

## 17. Full 16-season dataset loaded, a real scoring bug found and fixed, and the Results page overhauled

This section covers everything since Section 16 — the instructor configuration screen itself
still hasn't been started, but a lot of groundwork (a real historical dataset, a genuine
correctness bug, and a substantially richer Results screen) happened first.

### 17.1 The real VF26.1 dataset, extracted and loaded

The professor uploaded the original engine spreadsheet (`VF26_1_Engine_100__1_.xlsx`) — the
actual, complete input sheet used to run a prior semester. It was read precisely with
`openpyxl` (not just a text-dump tool), which caught a real row-offset bug in the first
extraction pass (a mislabeled "interest rate" row had shifted every subsequent row's data by
one, including the long-run-adjustment flag). Corrected extraction confirmed the spreadsheet's
19 markers break down as: 15 real seasonal events (1–15), season 0 (the baseline reference
point, never itself shown to students), and three pure placeholders (5.5, 10.5, and an
end-of-game marker) that this project already deliberately excludes.

**One deliberate deviation from the raw sheet:** the sheet's own `long_run_on` flag is set for
seasons 10, 10.5, 11, 14, *and* 15 — workbook carryover from how the original spreadsheet
tracked the adjustment across a multi-season transition window, not five independent triggers.
The validated ground truth from this project's own Phase 0 work (cross-checked against real
historical student score data) confirms the adjustment should fire exactly twice — season 10
and season 15 — so that's what got loaded, not a mechanical copy of the flag column.

**Loaded into a dedicated test instance (`test-run-vf26`), not the live class.** This was a
deliberate, explicit choice — the point was to let the professor run through all 15 real
seasons and check results without touching real student data in `eco112-fall26`. Files:
`load_full_test_run.sql` (creates the instance, 18 real team brands, and all 16 season rows in
one script, validated by actually loading it into a fresh local Postgres database before
handing it over) and `Burgerville_TestRun.html` (identical to the live client except
`INSTANCE_ID` points at the test instance, with a distinct browser-tab title so the two are
never confused).

### 17.2 A real scoring bug, found while investigating the results-comparison feature, with real consequences

While tracing through how to show a "correct answer" column, it became clear that
`quantity_estimate` (and its Part III equivalents) had a **units mismatch between what the form
told students to type and what the scoring engine actually compared against** — the form
labeled the field "billion lb." (implying e.g. `20.9`), but the tolerance-band comparison in
`ScoringPartI.ts` compares directly against raw-unit bounds (tens of billions). A `scale: 1e9`
property already sat in the field config, clearly intended to convert one to the other, but was
never actually applied anywhere, client or server — so a mathematically correct guess like
`20.9` (meant as billions) would always score zero on that component, regardless of accuracy.

**The professor's resolution wasn't to apply the missing scale factor — it was to remove the
mismatch entirely, on pedagogical grounds:** students should type the full raw number
(`20900000000`, not `20.9`), specifically *because* confronting the actual magnitude matters —
professor's stated experience is that students who are never forced to write out real
magnitudes make errors like believing Africa has ten billion farmers. This is a good example of
a "bug fix" whose correct resolution came from a design principle already established for this
project, not from picking whichever fix was mechanically easiest. Concretely: the `scale`
property and "(billion lb.)" labels were removed from all five affected fields (across all four
parts), replaced with plain "(lb.)" labels and a live, non-editable helper readout beneath the
field (e.g., "= 20.9 billion lb.") that helps catch an obvious typo without doing the
conversion *for* the student. No scoring-engine change was needed at all — it already expected
raw units; the bug was purely in what the form told students to type.

**Real consequence, found and corrected carefully.** The professor had already run Season 1 for
the live class before this was caught. Rather than assume every affected submission needed
correcting, a read-only diagnostic was run first (three students' actual submitted values),
which showed only **one** of three students was genuinely affected — the other two had guesses
far enough off that the units bug didn't change their outcome either way. The fix itself was
computed by re-running the *actual* scoring engine locally (not a hand-derivation) with that
one value correctly rescaled, confirming the correction was isolated to exactly one student's
`pts_quantity_estimate` and `total_points` (their rank bonus, and everyone else's numbers,
were confirmed unchanged) before writing a single, minimal, reviewed `UPDATE` statement against
the live database.

### 17.3 Results page: market chart, correct-answer comparison, and color coding

Three related additions to the Results screen:
1. **A market chart**, showing the resolved shift for whichever season is being viewed — same
   visual grammar as the Market Info tab's historical view (old/new curves, the shortage/surplus
   arrow, and both quantity-demanded/quantity-supplied movement arrows). Implemented as a
   self-contained function (`drawResultsChart`) with its own local canvas context and drawing
   helpers, deliberately *not* reusing the Market Info chart's `drawLine`/`drawArrow`/
   `labelCurve` (which are hard-wired to that chart's own canvas via module-level constants) —
   duplicating a modest amount of drawing code was judged lower-risk than refactoring
   extensively-validated, already-shipped code to be canvas-agnostic.
2. **A "Your Answer / Correct Answer" column**, for Parts I and II. The correct answer is
   derived client-side (`deriveAnswerKeyClient`), reusing the `marketDemand`/`marketSupply`
   primitives already present for chart-drawing. Deliberately does *not* re-derive price/
   quantity-demanded/quantity-supplied direction from raw parameters (which would need to
   account for `Qclass` — the sum of every student's own firm output in Part II — to be exactly
   right); instead it compares the *actual* historical equilibrium prices already returned by
   `get-market-info`, which are correct by construction since the real scoring engine computed
   them. Demand/supply direction genuinely are pure functions of the season's parameters
   (`Qclass`-independent) and are computed directly.
3. **Green/red row coloring**, driven directly by the points already earned on that component
   (not a separate correctness re-derivation) — this can never drift out of sync with the
   actual score, by construction.

**Deliberately scoped, not silently under-built:** Part III isn't covered yet — its correct
answer is a fundamentally different computation (an optimal monopoly/regulated outcome, not a
direction prediction) that hasn't been ported client-side. The rank bonus and Part II's own
firm-decision fields (land investment, cattle, etc.) intentionally show "—" for correct answer
rather than fabricating a comparison, since there genuinely isn't a single right value for a
choice a student made, only for a prediction.

Validated against the exact real, historically-verified season 1→2 demand-increase data used
throughout this whole project, plus a full DOM-level test confirming the rendered HTML
correctly marks a right answer green and a wrong one red.

### 17.4 Live comma-formatting for large-quantity inputs
A direct follow-on to 17.2: since students now type the full raw number, the five affected
fields format with thousands separators live as they type (`21130000000` →
`21,130,000,000`), switched from `type="number"` (which rejects commas outright) to `type="text"`
with numeric input mode. Commas are stripped back out immediately before submission, so the
stored value is unaffected — purely a readability aid, matching the same formatting already
used in the Results page's "Correct Answer" column.

### 17.5 Current status
Seasons 1–5 have been run through in full on `test-run-vf26` and confirmed working, including
the new Results page features. Seasons 6–15 (Part II and Part III) remain to be tested — Part
III in particular will need the "correct answer" feature extended once real testing surfaces
whether it's worth the additional client-side porting effort. The live class (`eco112-fall26`)
has its Season 1 quantity-estimate scoring corrected and is otherwise proceeding normally.

## 18. Part II and Part III fully brought up to the same standard as Part I, one real data-corruption bug found mid-way, and a genuine terminal-quoting detour

This section covers the long stretch of work between Section 17 and full end-to-end testing of
Seasons 6–15 — both firm-level dashboards, Part III's display layer for *both* of its modules,
a serious data-pipeline bug (not just a display bug) that required a one-time repair function,
several rounds of chart-design iteration driven directly by the professor's own worked examples,
and a readability pass across the whole app.

### 18.1 The Firm Dashboard: an in-browser spreadsheet, not a form

Part II and Part III both ask students to make firm-level decisions (land/cattle, or farms/
cattle) informed by cost curves they need to actually work out — not just fill in numbers. The
professor's existing Excel prototype was ported to an in-browser spreadsheet using
**HyperFormula** (an open-source formula-evaluation engine), rather than reimplementing formula
parsing from scratch. Two versions exist, sharing the same underlying mechanics but different
column layouts: one for Part II's small competitive firm, one for Part III's monopoly (both
Profit and Regulation share this second one — see 18.2).

**Persistence:** autosaves to a new `firm_dashboard_state` table (`instance_id, season_number,
participant_id, cells` as JSONB), keyed so Part II and Part III never collide even though they
reuse the same season-number space differently.

**A genuine, non-obvious bug found here:** the grid's own CSS class was originally just `.grid`
— which collided with a *pre-existing*, unrelated `.grid { max-width:1040px; margin:0 auto; }`
rule used for the app's overall card layout. The spreadsheet silently inherited a max-width and
centering rule meant for something else entirely, which looked like "the table is shifted right
and too narrow" — a symptom that pointed toward the column-width logic being wrong, when the
real cause was a naming collision several layers away. Renamed to `.sheet-grid` (and
`.sheet-grid .cell-inner`, which had the same problem) — a reminder that a plausible-looking
symptom doesn't guarantee the bug is where the symptom appears.

### 18.2 The Monopoly Dashboard: shared between Part III's two modules by design

Part III has two distinct modules — **Profit** (students maximize their own profit, 3 seasons)
and **Regulation** (students operate the monopoly to maximize *welfare* instead, price set at
average total cost, 2 seasons) — each with its own submission form and scoring method, but
**one shared dashboard**, per the professor's explicit direction: the underlying cost/production
mechanics are identical regardless of which objective the student is optimizing for, so
duplicating the tool would add maintenance cost without adding anything students actually need.
Bootstrapped with `m`-prefixed functions/state throughout (`mBuildGridDom`, `mHF`, `mSaveState`,
`mDrawChart`) specifically to avoid any collision with the small-firm dashboard's `d`-prefixed
equivalents, since both are compiled into the same HTML file.

One deliberate asymmetry, confirmed with the professor rather than assumed: Profit's
submission form includes a competitive-price/quantity *estimate* (comparing monopoly to what
competition would have produced) that Regulation's form doesn't need, since Regulation isn't
benchmarking against competition — it's targeting Price = ATC directly. The dashboard's
Pc/Qc ("competitive estimate") cells were kept as-is for Regulation seasons too, rather than
hidden or relabeled — students can simply ignore or repurpose them — since building a second,
slightly-different dashboard variant wasn't judged worth the added surface area for two cells.

### 18.3 Market Info redesign: split cost details, and a unified two-column outcomes table

Two related changes, both aimed at fitting substantially more information (firm-level cost
parameters, now needed for Part II onward) into the same screen without it feeling cluttered:

1. **Market Details split into two sub-columns** for Part II and later — "Market Details"
   (tax/subsidy/imports/producer count) alongside "Production & Cost Details" (calf cost, land
   rent, tech factor, α, β) — rather than one long stacked list.
2. **Market Outcomes condensed into a single two-column table**, reused identically across
   every part rather than having Part-specific formats: "Original" vs. "Current" for a normal
   competitive season, "Original/Competitive" vs. a separate Monopoly row for Part III. This
   replaced an earlier, more stacked design (separate "Original Price / Original Qty / Shortage
   at $X / Current Price / Current Qty" rows) that the professor flagged as both visually heavier
   than necessary and, worse, actively misleading for Part III (see 18.7's narrative-text fix
   for the related problem this format created).

### 18.4 Part II scoring refinements

Several rounds of real-testing feedback, each addressing a genuine gap rather than a cosmetic
preference:
- **Point value rebalanced.** Part II's direction-prediction questions were dropped from
  `PTS_BASIC` (0.5, matching Part I) to a new `PTS_BASIC_PART_II` (0.25) — the same six
  categorical questions repeat from Part I, so full weight a second time over-rewarded
  repetition rather than new understanding.
- **A missing question added.** "Shortage or surplus?" existed conceptually in the original
  design but had no submission field, scoring column, or results-page row — added end-to-end
  (`shortage_or_surplus_answer` on the submission, `pts_shortage_or_surplus` on the results,
  migration for the live database, and a new results-page row), not just patched at whichever
  single layer first surfaced the gap.
- **Firm-level correct-answer display**, extending the "Your Answer / Correct Answer" comparison
  (built in 17.3 for Parts I/II's market-direction questions only) down to the firm-decision
  fields themselves — actual output, actual marginal cost, price-minus-marginal-cost, and
  profit-vs-maximum-possible, each derived client-side from the *student's own* land/cattle
  inputs using the exact same formulas as the server-side scoring engine (cross-validated
  numerically, not just visually inspected).
- **Credit-limit validation on the submission form** — land investment is now checked live
  against the season's endowment as the student types, with a clear warning and a submit block
  if exceeded, rather than only being caught after the fact by the scoring engine.

### 18.5 Two orphaned Part III Profit scoring components found and fixed

While building the Part III Profit equivalent of 18.4's correct-answer display, two scoring
components — `priceCalculation` and `farmsAccuracy` — turned out to be **computed by
`ScoringPartIIIProfit.ts` but never persisted anywhere**: no column in `results_part_iii_profit`,
and no mapping in `run-current-season`'s result-row builder. Structurally the same class of bug
as Section 17.2 (a real computation whose result silently never reaches the database), just
caught by inspection this time rather than a live-classroom incident. Fixed with the same
pattern used throughout this project: schema columns added via migration, the result-row mapping
corrected, and the fix verified by running the real scoring function against realistic
monopoly-season data and confirming both values now compute and would persist correctly.

### 18.6 Part III Regulation brought up to the same display standard as Profit

Everything built for Part III Profit's display layer (dashboard part-gating, the Market Outcomes
table, the Results page's correct-answer comparison, the completed-season chart) was hard-coded
to `IIIProfit` specifically and did nothing for `IIIRegulation` — confirmed by checking each
integration point directly rather than assuming symmetry. Extended throughout, with one
structural difference worth understanding: **Regulation's scoring engine only ever grades the
*outcome* of a student's farms/cattle choice** (quantity, price, consumer surplus, and welfare,
each compared against the true price=ATC optimum) — none of the five typed fields (stated price,
marginal cost, marginal revenue, quantity, consumer surplus) are individually scored, by the
original design. The Results page reflects this honestly: those five fields show as plain
reference values, uncolored, while four *derived* rows (not tied to any single submitted field)
show what the student's actual choice produced against the true regulated target, colored
correctly. This mirrors the "no fabricated correctness" principle from 17.3 rather than
inventing per-field comparisons the scoring engine doesn't actually make.

**A genuine bug found and fixed along the way:** the dashboard's "load my own previous
submission" query was hard-coded to Part III Profit's table and field name
(`submissions_part_iii_profit`, `stated_monopoly_quantity`), so a Regulation season would either
silently fail to prefill or pull nothing at all. Made table/field selection dynamic based on
which module is actually active.

Per the professor's explicit choice: outcome labeling distinguishes the two modules ("Regulated
Monopoly Outcome" vs. plain "Monopoly Outcome") throughout the chart and outcomes table, rather
than using one generic label for both.

### 18.7 A real data-corruption bug: `equilibrium_history` stored the wrong equilibrium for every Part III season

The most serious bug found in this stretch of work — not a display glitch, but corrupted data
written to the database by every Part III season run before the fix.

**The bug.** `run-current-season`'s logic for what to write to `equilibrium_history` (the table
that becomes next season's "baseline" reference, both for the chart and for continuity) was:

```
const trueEq = output.answerKey ? output.answerKey.newEquilibrium
  : (output.monopolyOptimum || output.regulatedOptimum);
```

Parts I/II have an `answerKey` with the real competitive equilibrium baked in, so this worked
correctly for them. Part III has no `answerKey` at all, so the fallback kicked in — storing that
season's *monopoly* (or regulated) outcome as if it were the market's competitive equilibrium.
The two are, by design, very different numbers (a monopoly restricts output and raises price
well above the competitive level) — so every season following a Part III season inherited a
badly wrong "what the market looked like before this shift" reference point.

**How it surfaced.** The professor, working through real test-run seasons and independently
sanity-checking the numbers by hand, caught it from the *chart* — arrows pointing the wrong
direction, an "Original" reference price that didn't match the true prior competitive price, and
(a second, related symptom) the "Competitive" column of the outcomes table snapping to that
season's own monopoly values once the transition animation reached 100%. Both symptoms traced
back to the exact same single line: `get-market-info`'s `oldState`/`newState` price and quantity
are read directly from `equilibrium_history`, so once that table held the wrong number for a
season, everything downstream that trusted it — the chart's starting point, the outcomes table,
the next pending season's baseline — inherited the same error.

**The fix.** `equilibrium_history` now always stores the true competitive equilibrium, computed
directly via `solveEquilibrium(newParams, newFirmCount, 0)`, for every part uniformly, rather
than branching on whether an `answerKey` happens to exist. Verified directly against both Part
III modules with realistic season data (previously, Regulation's own scoring function didn't
even expose a competitive-equilibrium value at all — computing it independently, rather than
relying on each scoring module's differing output shape, closes that gap for good).

**Confirmed this never affected any student's actual score.** Part III's scoring functions take
only that season's own `newParams` (from `season_parameters`, never corrupted) — never
`equilibrium_history`. This was purely a display/continuity bug, with real consequences for
what the chart showed, not for what anyone was graded on.

**The repair problem: fixing the code doesn't fix data already written.** Seasons already run
before the fix have the wrong values sitting in the database permanently — no amount of
redeploying corrects a row that already exists. Built a small, one-time-use Edge Function,
`repair-equilibrium-history`, that recomputes the correct competitive equilibrium directly from
each already-completed Part III season's `season_parameters` (never corrupted) and updates both
`equilibrium_history` and, if the most recent completed season was affected, `game_state`'s own
baseline fields. Idempotent by design (always recomputes from source parameters, so re-running
is harmless) and returns each season's before/after values directly in its response — which is
how the fix was actually confirmed on `test-run-vf26`: Season 11's stored price corrected from
$21.84 (its own monopoly price) to $9.77 (its true competitive price), Season 12 from $24.84 to
$11.49, Season 13 from $26.85 to $17.85 — each "before" value an exact match for that season's
*monopoly* price, confirming the bug's signature precisely, and Season 13's corrected value
landing almost exactly on the professor's own independent hand-calculation ($17.80), a good
independent check that the fix is mathematically sound and not just structurally different.

### 18.8 Chart redesign, driven directly by the professor's own worked example

The Part III chart went through real iteration, each round driven by a specific, concrete
correction from the professor rather than a general "make it look better" request:

1. **First pass:** the monopoly outcome shown as a separate green dot, with the two black
   "movement to new equilibrium" arrows *removed* entirely for any Part III season, on the
   reasoning that the market doesn't actually move there under monopoly.
2. **Correction, backed by the professor's own Season 7 screenshot as a concrete reference:**
   this went too far. The competitive-shift visualization (the same black/orange arrows shown
   for a normal season) needed to stay *exactly as-is*, for every part — it shows what
   competition alone would have done, which is the actual pedagogical point of the comparison
   ("the key to monopoly is being able to compare the competitive outcome to what a monopoly
   would do"). The monopoly outcome is an *addition* to that picture, not a replacement for it.
   Restored the standard arrows unconditionally, and — since hiding the "Watch the Adjustment"
   slider for Part III had been part of the same over-correction — restored that too.
3. **A connecting green arrow, added then removed one round later:** briefly added a second
   arrow from the competitive point to the monopoly point, to visually tie the two together. The
   professor's own framing made clear this was wrong: "the process is not about change from one
   to the other (that almost never happens) — it's about imagining the alternative." A monopoly
   doesn't arrive at its outcome *from* the competitive one; it's a separate, deliberate choice
   being compared against a hypothetical. Removed.
4. **Narrative text rewrite.** The same underlying misconception showed up in the plain-English
   description below the chart — a single sentence describing a shortage "pushing price up" to
   the *monopoly* price, which isn't what a shortage does; a shortage pushes toward the
   *competitive* price, and the monopoly price is a firm's deliberate choice, unrelated to the
   shortage/surplus mechanism entirely. Split into two explicitly separate sentences for any
   Part III season: what competition alone would produce (conditional — "if this market were
   competitive..."), followed by a clearly separate statement of the actual monopoly (or
   regulated) outcome and why it's not driven by the same mechanism.

Both of `draw()` (Market Info tab) and `drawResultsChart()` (Results page) needed each of these
changes applied in parallel — they are separate, independently-written functions (a deliberate
choice from Section 17.3, to avoid risking already-shipped code by making the Market Info chart
canvas-agnostic), which meant the *same* mistake had to be independently caught and fixed twice
each round. Worth remembering if the chart needs touching again.

### 18.9 `submit-prediction` hardened against a stale season number

While investigating a confusing "Part mismatch" submission error on `test-run-vf26` (a student —
actually the professor's own test submission — had a season's form open in the browser, but the
game had already advanced past it by the time they clicked submit), found that
`submit-prediction` never checked what season the client's form was actually built for. It
always independently looked up `game_state.current_season` and validated against *that* season's
part — correct in isolation, but blind to whether the browser's view was still current, and (a
more serious latent risk than this one incident) capable of silently recording a submission
under the wrong season number if the mismatch went undetected. The client now sends the season
number its form was rendered for; the server checks that against `game_state.current_season`
*before* checking part, and returns a clear "the game has moved on since you loaded this page,
please refresh" message on mismatch, rather than either the confusing part-mismatch text or a
silent wrong-season write.

### 18.10 A genuinely time-consuming terminal-quoting detour, worth documenting for next time

Running the new repair function's `curl` command repeatedly got stuck on a `dquote>` prompt —
zsh's way of saying it found an opening quote with no matching close, waiting indefinitely for
more input. The command looked correctly balanced every time it was retried. Root cause:
**macOS's "smart quotes" silently converting straight quote characters into curly typographic
ones** during copy-paste — visually near-identical in a terminal font, but not recognized as
quote characters by the shell at all, so an odd number of "real" quotes gets left after even one
silent substitution. Resolved by moving the request body into a file first (`cat > file <<
'EOF' ... EOF` — the quoted heredoc delimiter means the shell treats everything between the two
`EOF` markers as pure literal text, sidestepping quote-interpretation entirely), then having curl
read from that file rather than embedding a long, quote-heavy JSON body directly in the command
line. Worth reaching for this pattern immediately, rather than as a last resort, for any future
`curl` command with a nontrivial `-d` payload run from a terminal that might have smart quotes
enabled.

### 18.11 Readability pass across the whole app

Separately from any of the above: the professor reported the app's standard text was hard to
read. Confirmed this was baked into the code, not a browser/OS zoom setting — 66 separate
`font-size` declarations exist across the CSS, the login screen's inline styles, and JS-generated
form templates, almost all in fixed pixel values rather than relative units, so no single root
font-size change could have reached most of them. Scaled every declaration up systematically
(~25% throughout the app generally; a more conservative ~15% specifically for the Firm/Monopoly
Dashboard's spreadsheet grid, whose column widths were already tuned tightly in 18.1's fixes),
plus increased the root font-size for the handful of places using relative units. Confirmed via
the full regression suite that nothing structurally broke, though — worth flagging honestly —
that suite doesn't verify visual rendering, only logic and structure, so this was a first pass
confirmed correct by the professor's own follow-up ("much improved") rather than something fully
self-verified.

### 18.12 Current status

All of Parts I, II, and III (both Profit and Regulation modules) now share the same display
standard: a firm-level dashboard where relevant, a unified Market Outcomes table, a Results page
with a real correct-answer comparison and coloring, and a chart whose competitive-shift
visualization is identical across every part, with Part III's outcome shown as an addition to
that picture rather than a replacement for it. The `equilibrium_history` bug is fixed going
forward and repaired on `test-run-vf26`'s already-run seasons (Seasons 11–14 confirmed via the
repair function's own before/after output); the live class (`eco112-fall26`) has not yet had the
same repair run and should not be assumed clean until it has, if any Part III seasons have been
run there. The professor is actively testing `test-run-vf26` season-by-season with concrete,
specific feedback each round — the working pattern for the remainder of this testing phase looks
to be: professor finds something wrong on a real screen, screenshot in hand, root cause traced
to its actual source (not just the layer where the symptom appeared) before any fix is written.
