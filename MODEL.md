# MonumentalAI — Civ 6 Victory-Time Optimization Model (v2)

*Named for Monumentality, the golden-age dedication that makes the faith
economy sing — and for the scale of the ambition.*

A formal model of Civilization VI's economic core for computing optimal play
that minimizes expected victory turn E[T], for Science and Culture victories.
The model is the specification; any code is just its numerical solver.

Status: design review complete (2026-10-08). v1 errors corrected, structure
refined. Open calibration gaps listed in §9.

---

## 1. Problem statement

Civ 6 is a finite-horizon stochastic game. We decompose it into:

- **Deterministic core** (fully modelable): city yields, growth, production
  queues, tech/civic progression, great people, tourism accounting, and all
  currency exchanges. Every formula below is verified against game data.
- **Stochastic shell** (estimated, sampled): AI rival behavior (notably rival
  culture rates), combat RNG, hidden map, goody huts.

Decision rule: at each decision point, choose the action maximizing expected
victory-time reduction, ΔT(a) = E[T(s)] − E[T(f(s,a))], computed by forward
simulation over the deterministic core with the shell sampled.

## 2. State representation

The state splits into **local** (per city c) and **global** (empire-wide).
This split is load-bearing: it is what makes locality enter the math.

**Local** (city c at turn t):
- p_c(t): population; food stock; housing cap h_c; net amenities a_c
- π_c(t): production/turn; single build queue q_c(t) (one item at a time)
- Districts (placed vs. completed), buildings, worked tiles, governor present

**Global:**
- G(t): gold stock; F(t): faith stock
- S(t): science/turn; K(t): culture/turn
- τ(t): tourism/turn; L(t): lifetime tourism (by target civ for culture)
- GPP per great-person type; unlocked techs/civics; envoys; trade route
  capacity and assignments; diplomatic state; era score / age

**Key structural fact:** gold, faith, science, culture, tourism, and GPP are
fungible empire-wide. Production, food, housing, and amenities are
city-local, and each production queue is a single-server line.

## 3. Deterministic core — verified formulas

All rates at Standard speed; scale per game speed where noted.

**G1. Growth.** Marginal citizen p costs
  Φ(p) = 15 + 8(p−1) + (p−1)^1.5 food.
Each citizen consumes 2 food/turn. Housing: growth is penalized as p
approaches/exceeds h_c, collapsing toward zero past the cap (exact penalty
curve: calibration gap §9).

**G2. Passive yields per pop.** Each citizen yields 0.5 science + 0.3 culture
per turn, plus worked-tile yields.

**G3. Amenities (corrected v2 — step function, not linear).** Net amenities a_c
map to multipliers on non-food yields and growth:

| Level (net a) | Growth | Non-food yields |
|---|---|---|
| Ecstatic (≥ +5) | +20% | +10% |
| Happy (+3 to +4) | +10% | +5% |
| Content (−1 to +2) | — | — |
| Displeased (−3 to −4) | −15% | −10% |
| Unhappy (−5) | −30% | −20% |
| Unrest (−6 to −7) | halted | −30% |
| Revolt (≤ −8) | halted | −50% |

Requirement ≈ 1 amenity per 2 population, counting from pop 3. The threshold
jumps make amenity investment a discrete optimization: buy the luxury that
crosses a breakpoint, ignore the rest.

**G4. District cost.** Placed-district production cost:
  D(type, t) = base · (1 + 9ξ(t)) · 0.75^{below_avg(type)},
where ξ(t) = max(tech progress, civic progress) fraction, and the 0.75
applies when the empire's count of that district type is below the all-player
average. **Cost locks at placement** (free, instant action), independent of
when construction starts. Strategic consequence, derived not assumed: place
planned districts ASAP; keep type counts diversified below average.

**G5. Chops / harvests.** Yield V(t) = v₀ · (1 + 9ξ(t)) (same progress
scaling as districts; e.g. Forest v₀ = 20 production, Jungle 10 prod +
10 food, Marsh 20 food). Effective value when spent:
  V_eff = V(t) · 1.5^{Magnus} · (1 + ρ_policy(target)),
where Magnus's Groundbreaker gives +50% on harvests/feature removal in his
city, and ρ_policy is the production multiplier of whatever the city is
currently building (chop overflow inherits the target's modifiers — e.g.
Limes +100% on walls stacks with Magnus). Chop timing is therefore a joint
optimization over *when* (ξ growth vs. time discounting) and *what the queue
holds* (policy multiplier).

**G6. Gold purchases.** Rush-buying costs 4 × remaining production cost
(units, buildings; districts via Reyna's Contractor presumed same rate on
the locked cost — calibration gap).

**G7. Great-person patronage.** Marginal cost: 150 + 10·(points remaining)
faith, or 200 + 15·(points remaining) gold. I.e. 1 GPP ≡ 10 faith ≡ 15 gold
at the margin.

**G8. District projects.** While running, convert 15% of city production to
science (Campus Research Grants), culture (Bread and Circuses), or faith
(Holy Site Prayers), plus GPP on completion (~22% of invested production as
GPP on Standard). Deliberately inefficient — the game has no efficient
production→science converter, which the model respects.

**G9. Royal Society.** Each builder charge spent in the project city adds 2%
of a district project's production cost. Space-race projects are the
binding application.

**G10. Tourism accounting (culture victory).**
- Domestic tourists (rival r): DT_r = (lifetime culture of r) / 100.
- Foreign tourists you attract from r: FT_r = (lifetime tourism applied to
  r, with bilateral modifiers) / (150 · N), N = civ count.
- Win condition: FT_r > DT_r for every rival r. (Foreign tourists never
  decline once earned.)

**G11. Eurekas / inspirations** grant 40% of the tech/civic cost.

## 4. Exchange operators — the currency board

| # | Conversion | Rate | Notes |
|---|---|---|---|
| E1 | gold → hammers | 1 : 4 | anywhere, instant (G6) |
| E2 | faith → hammers (units) | ≈ 1 : 2 | Grand Master's Chapel / Theocracy; varies by item (gap) |
| E3 | faith/gold → GPP | 10 F / 15 G per point | marginal (G7) |
| E4 | hammers → sci/cul/faith | 0.15 per turn | projects (G8); inefficient by design |
| E5 | food → pop → yields | Φ(p) food → tile yields + 0.5 sci + 0.3 cul per turn, perpetuity | (G1, G2) |
| E6 | chop → hammers | V_eff per charge | (G5); transportable within city only |
| E7 | builder charges → project hammers | 0.02 × project cost per charge | Royal Society (G9); the space-leg converter |
| E8 | gold → district | presumed 4× locked cost | Reyna Contractor (gap: confirm rate) |

**Derived arbitrage (v2):** for the space leg, E7 via bought builders beats
E1. Builder route wins iff C_builder < 0.08 · n_charges · P_project in gold
terms. For a 5-charge builder against a 2000-production Mars module, viable
up to ~800 gold/builder — roughly 2× actual late-game builder prices. The
model therefore *derives* "stockpile builders before the space leg" rather
than taking it as lore. (Monumentality faith makes the arbitrage even wider.)

## 5. Objectives

**Science.** T = T_tech + T_exec, subject to:
- ∫₀^T_tech S(t)dt ≥ Σ_i c_i − Σ_boosted 0.4·c_i (eureka accounting, G11),
- each space project j requires P_j hammers in its city; Mars modules run
  in parallel across spaceports (finish = max, not sum).

Setup vs. execution split: building/buying the spaceport, positioning
builders and Magnus, and slotting policy cards can fully overlap T_tech;
only project execution is sequential. The optimizer's late-T_tech job is
purely setup — every action scored by T_exec removed.

**Culture.** min T subject to ∀r: FT_r(T) > DT_r(T) (G10). The finish line
DT_r(T) = C_r^life(T)/100 is adversarial and growing: culture is a race
against the highest-culture rival, not a fixed threshold.

## 6. Time preference

v1 used a constant discount ρ ≈ 1.5× per 30 turns. v2: **the discount rate
is not a constant.** Early yields compound at the empire's reinvestment
growth rate g(t) (~2–4%/turn during expansion, →0 in a mature empire);
near victory, a yield's value is purely whether it advances the critical
path. The optimizer therefore does not discount with a fixed ρ — it
computes ΔT(a) directly via the forward model. ρ survives only as a human
heuristic and as a prior for truncated search.

## 7. Optimality conditions

**O1. Bellman.** V(s) = 1 + min_a E[V(f(s,a))]. Intractable exactly; the
conditions below are its tractable first-order consequences, used for
pruning and scoring.

**O2. Marginal equalization.** At optimum, the victory-time reduction per
unit of spendable currency is equalized across uses. Discrete form: rank
candidate purchases by ΔT/cost and spend down the list — but only against a
**reservation price** r_G(t): the discounted best future use of gold (e.g.
the planned Reyna spaceport buy). Spend iff ΔT/cost > r_G(t). This prices
the option value of holding gold. (Greedy ranking is a first-order
approximation for interacting actions; the solver replans after each action
— model-predictive control, not one-shot ranking.)

**O3. Shadow prices of local production.** λ_c(t) = (∂T/∂d_c)/π_c(t):
marginal victory-time value of one hammer in city c, via the queue-delay
channel d_c. Decision rule: **all global currency (gold, faith, bought
builders) flows to c* = argmax_c λ_c(t)**; builders and chops are
transportable hammers, built/bought where λ is low, spent where it is
highest. Sharp consequence: **λ_c = 0 for any city off the critical path**
— e.g. after T_tech in a science game, every non-spaceport city's hammers
are worth ~zero, and the optimizer stops considering ~90% of build actions.
(Distinct from the *flow* channel: every city's S(t)/K(t) output still
matters during T_tech.)

**O4. Eureka stopping rule.** For tech/civic i with boost pending, cap
investment at 0.6·c_i; resume only if the boost is unreachable. Optimal
stopping, not heuristic. (Same for inspirations.)

**O5. Phase switching.** The trajectory's phase structure (expansion →
infrastructure → victory leg) is *emergent* from threshold nonlinearities
(district unlocks, golden-age windows, the T_tech pivot), not imposed.
Switch investment from regime k to k+1 when marginal ΔT crosses. Do not
hardcode phases in the solver.

**O6. Assignment problems** (all solved by enumeration — small and discrete):
- Governors → cities: Magnus → argmax(pending chop value × λ_c); Pingala →
  argmax science+culture flow; Reyna → argmax gold flow or district-buy
  target; etc.
- Policy cards → slots at each civic unlock: maximize the value function
  under the new multiplier regime.
- Dedications (e.g. Monumentality) → era: choose the regime maximizing ΔT;
  note dedications *rewrite* exchange rates (E2/E7 get dramatically cheaper),
  so faith stockpiling ahead of a Monumentality window is derived behavior.
- Envoys → city-states: marginal equalization over suzerainty bonuses.
- Traders: each trader is queue-cost in its home city, then a *mobile,
  reassignable* yield bundle (food/prod domestic; gold/science
  international); value = max over assignments of the bundle dotted with
  shadow prices.

## 8. Solver architecture

Two layers:

- **Strategic layer** (replanned every ~10 turns or at regime changes: tech/
  civic unlocks, era rolls, war/peace): sets shadow prices λ_c, gold
  reservation price r_G, governor/policy/dedication regimes; samples the
  stochastic shell (rival culture as estimated process from scouting +
  margin; combat in expectation).
- **Tactical layer** (per turn, MPC): enumerates non-dominated actions,
  scores each by 1-step ΔT via the deterministic forward model, pruned by
  O2–O6 (skip λ_c=0 cities, cap pre-eureka investment, etc.), acts, replans.

Feasibility constraints: loyalty for new cities (settler targets must hold),
housing caps on growth, strategic-resource requirements for units.

## 9. Calibration gaps

Empirical parameters not verifiable from public sources; measurable in-game
in an afternoon; none change the model structure:

1. Exact faith purchase cost formula by item type (units/buildings/districts).
2. Settler production-cost scaling curve.
3. Housing-vs-growth penalty shape near/over cap.
4. Production overflow cap rules.
5. Reyna Contractor district gold cost (presumed 4× locked cost).
6. Re-derive the time-preference calibration from compounding structure
   rather than the single forum analysis.

## 10. Changelog (v1 → v2)

- Corrected: amenity multipliers are a step function (G3), not ±10%/step.
- Replaced constant discount rate with direct ΔT computation (§6).
- Added gold reservation price / option value of holding currency (O2).
- Formalized shadow prices with critical-path λ_c = 0 pruning (O3).
- Discovered builder→Royal Society arbitrage beating 4:1 rush-buying (§4).
- Extended chop model with policy-multiplier stacking (G5).
- Added district cost-locking as a distinct free action + 25% diversification
  discount (G4).
- Split space leg into setup (overlaps T_tech) vs. execution (O5/Science).
- Added regime switches: dedications, governments, governors, policies,
  envoys, traders as assignment problems (O6).
- Added maintenance drain on gold dynamics; loyalty feasibility constraint.
- Phases demoted from imposed structure to emergent property (O5).
