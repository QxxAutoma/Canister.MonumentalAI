# MonumentalAI — Enhanced AI for Civilization VI

*An AI player that actually plays the game, driven by a formal mathematical
model of its systems rather than prompts. Named for Monumentality, the
golden-age dedication that makes the faith economy sing — and for the scale
of the ambition.*

A formal model of Civilization VI's economic core for computing optimal play
that minimizes expected victory turn E[T], for Science and Culture victories.
The model is the specification; any code is just its numerical solver.

Status: v3 (2026-10-09). Full wiki validation audit complete — every numbered
claim checked against civilization.fandom.com. **Ruleset target: Gathering
Storm.** Equations encode GS values; vanilla/R&F divergences are tagged.
Open calibration gaps listed in §9.

---

## 1. Problem statement

Civ 6 is a finite-horizon stochastic game. We decompose it into:

- **Deterministic core** (fully modelable): city yields, growth, production
  queues, tech/civic progression, great people, tourism accounting, and all
  currency exchanges. Every formula below is verified against game data or
  the wiki, with the source noted.
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

## 3. Deterministic core — verified formulas (GS)

All rates at Standard speed; scale per game speed where noted.

**G1. Growth.** Marginal citizen p costs
  Φ(p) = 15 + 8(p−1) + (p−1)^1.5 food
(CivFanatics formula thread; not stated as text on Fandom — citation moved
accordingly). Each citizen consumes 2 food/turn (Food_(Civ6)). Housing and
amenity/loyalty modifiers scale the **surplus added per turn**, not Φ(p).
Housing table (Housing_(Civ6), exact):
  Housing − Pop ≥ +2 → 100% growth; +1 → 50%; 0 to −4 → 25%; ≤ −5 → 0%
  (hard cap at Housing+5). Loyalty multiplies growth: −25% at 51–75,
  −75% at 26–50, −100% at 0–25; starvation drains loyalty 4/turn (City_(Civ6)).

**G2. Passive yields per pop.** 0.5 science/citizen (Science_(Civ6),
"with or without the Campus"). Culture is **positional, not flat**
(Population_(Civ6)): citizens #1, 6, 11… → 0.2; #5, 10, 15… → 0.4; all
others 0.3. Flat 0.3 is the 5-citizen average (sums to 1.5) — use the cycle
in exact computation, 0.3 labeled as average in heuristics.

**G3. Amenities — GS ruleset** (Amenities_(Civ6); v2 encoded the vanilla
column — corrected). Net amenities a_c vs. required:

| Level (net a) | Effect (GS) |
|---|---|
| Ecstatic (≥ +5) | **all yields +20%, food included** (no separate growth bonus) |
| Happy (+3 to +4) | **all yields +10%, food included** |
| Content (0 to +2 above required) | — |
| Displeased (1–2 fewer) | −15% growth, −10% yields |
| Unhappy (3–4 fewer) | −30% growth, −20% yields |
| Unrest | growth halted, −30% yields |
| Revolt | −40% yields |

Requirement: 1 per 2 pop from pop 3 (vanilla rule confirmed); **GS
non-capitals get no free amenity** (capital excepted via Palace), so
requirement = floor(pop/2)+1 for non-capitals. Threshold jumps make amenity
spending discrete: buy the luxury that crosses a breakpoint. (Flag: Fandom
is internally inconsistent across pages on the negative-side GS values;
bands above are per the Amenities page GS column.)

**G4. District cost.** Placed-district production cost (District_(Civ6), exact):
  D(type, t) = base · (1 + 9·max(T, C)) · 0.60^{discount_eligible},
T/C = tech/civic completion fractions. **Cost locks at placement** (free,
instant action). Discount = **40% off** (not 25%; 25% is Government Plaza /
Diplomatic Quarter only), granted when the empire's completed count of that
type is below the **empire-internal** average across types, gated on total
completed specialty districts ≥ number of unlocked district types. Base
costs: most 54, Aqueduct 36, GP/DQ 30, Dam/Canal 81, **Spaceport 1800 with
no progress scaling**. Derived strategy: place planned districts ASAP to
lock cost; keep type counts diversified below the internal average.

**G5. Chops / harvests — GS model.** V(t) = v₀·(1 + 9ξ(t)), same ξ as
districts. Base: Forest 20 production; **Rainforest 10 production +
10 food** (game XML Features.xml; Fandom's Builder-page worked example
shows 61/61 after equal scaling — consistent; Fandom's Rainforest page
"50 base" flagged as likely editor error); Marsh 20 food. **Magnus
Groundbreaker +50% is a separate multiplicative layer** (GS; R&F was +100%).
**GS overflow rule** (Builder_(Civ6)): the absorbed portion of a chop gets
the target's building modifiers; the **overflow portion is stripped to base
production**; modifiers are fixed by the construction **at last turn's end**
(no within-turn swapping as in R&F). Effective model:
  V_eff = [min(V·1.5^Magnus, remaining)]·(1+ρ_policy)
        + max(0, V·1.5^Magnus − remaining)·(base only).
Planted (second-growth) woods yield **zero** on removal — no chop-planting
arbitrage. Placing a district/building destroys features with no yield:
always chop first.

**G6. Gold purchases.** Rush-buy = 4 × remaining production cost, rounded
down to nearest 5 (community formula thread; Fandom Gold_(Civ6) states the
equivalence "1 Gold = 0.25 Production"). Modifiers: Merchant Republic
legacy discounts gold purchases (15% + 1%/15 turns). **Repeat civilian-unit
purchases escalate in price** — the marginal rate is not constant across
repeated buys.

**G7. Great-person patronage** (Great_People_(Civ6), exact):
  Faith = 150 + 10·(total − available); Gold = 200 + 15·(total − available).
Fixed components (150/200) do **not** scale with game speed. GPP `total`
scales by (1 + 0.3·era gap)^era gap when the world era lags the GP's era.
**Oracle: −25% faith patronage. Democracy: −50% gold patronage.**

**G8. District projects** (Project_(Civ6); rates are **not uniform**):
- Campus Research Grants: 15% of production → science/turn; GPP ≈ 22% of
  invested production on Standard completion.
- Holy Site Prayers: 15% → faith/turn (+100% religious pressure while
  ongoing in GS).
- **Theater Square Festival: 15% → culture/turn** (GPP 11% each to
  Writer/Artist/Musician). **Bread and Circuses grants loyalty only — it is
  not a culture converter** (v2 misattribution corrected).
- Commercial Hub Investment: **30%** → gold/turn. Encampment/Harbor
  projects: 15% → gold. Dark Summoning: 30% → faith.
- GPP = D_TYPE·(1 + 7·DEVELOPMENT_RATIO), D_TYPE ∈ {10 most, 5 Festival/
  Carnival, 30 Occult Research}; GPP yield does **not** scale with game
  speed.

**G9. Royal Society.** 2% of project production cost per builder charge
(3%→2% Arctic Summer nerf); consumes **all** charges of the builder; once
per city per turn (List_of_buildings_in_Civ6).

**G10. Tourism accounting — GS** (Tourism_(Civ6)):
- Domestic tourists (rival r): DT_r = (lifetime culture + lifetime civic
  boosts)/100 — inspiration culture counts too.
- Foreign tourists from r: FT_r = (lifetime tourism applied to r, with
  bilateral modifiers) / (**200**·N), N = civ count (150 was vanilla;
  R&F raised to 200, GS inherits).
- Foreign (visiting) total is monotone non-decreasing; the rival's
  *domestic* count falls as you convert it — **the finish line is
  endogenous and decays as you approach victory**.
- Win iff FT_r > DT_r for every rival r. Tourism applies only to **met**
  civs (bilateral term needs a met-status gate). Modifiers are structured:
  national multiplicative × per-rival additive (trade route +25%, open
  borders +25%, government-difference penalty, …).

**G11. Eurekas / inspirations: 40%** of tech/civic cost in R&F/GS
(Boost_(Civ6); vanilla was 50%). Exceptions: China +10%, Free Inquiry /
Pen-Brush-Voice +10%, Babylon = full remaining cost, Near Future
Governance inspiration 90%.

## 4. Exchange operators — the currency board (GS)

| # | Conversion | Rate | Notes |
|---|---|---|---|
| E1 | gold → hammers | 1 : 4 | anywhere, instant (G6); Merchant Republic legacy improves it |
| E2 | faith → hammers (units) | ≈ 1 : 2 | Fandom: "1 Faith = 0.5 Production = 2 Gold"; Theocracy legacy −(15%+1%/15t); **exact formula unverifiable — keep empirical** |
| E3 | faith/gold → GPP | 10 F / 15 G per point | marginal (G7); Oracle/Democracy modify |
| E4 | hammers → sci/cul/faith/gold | **15% grants/prayers/festival; 30% investment/dark summoning** | (G8); Bread and Circuses excluded |
| E5 | food → pop → yields | Φ(p) food → tile yields + 0.5 sci + positional cul per turn, perpetuity | (G1, G2) |
| E6 | chop → hammers | V_eff per charge | (G5); Magnus multiplicative; GS overflow strip |
| E7 | builder charges → project hammers | 0.02 × project cost per charge | Royal Society (G9); the space-leg converter |
| E8 | gold → district | 4× the **locked** production cost | Reyna Contractor (Fandom: scales parallel to production cost + 1G=0.25P); **case split: first-copy buys don't get the 0.60× empire discount** |
| E9 | faith → districts / faith income | Moksha Divine Architect buys districts with faith; Citadel of God: +25% of a completed building's production cost as faith | new purchase path + income term |

**Derived arbitrage:** for the space leg, E7 via bought builders beats E1
iff C_builder < 0.08·n_charges·P_project in gold terms (≈2× headroom at
late-game prices). **Pingala Space Initiative: +30% production to all
space-program projects** stacks with Royal Society injections — include in
the space-leg production equation.

## 5. Objectives (GS)

**Science.** T = T_tech + T_exec + T_travel, subject to:
- Tech gate: only **Rocketry** (Spaceport), **Satellites** (Moon Landing),
  **Nanotechnology** (GS Mars Colony), **Smart Materials** (Exoplanet
  Expedition) gate the projects — the space leg's science requirement ends
  well before the full tree (Spaceport_(Civ6)).
- **GS has ONE Mars project** ("Launch Mars Colony") — the v2
  parallel-modules/max equation was vanilla/R&F-only and is deleted.
- Project completion ≠ victory: Exoplanet Expedition requires **50
  light-years of travel**; Lagrange + Terrestrial laser stations speed the
  spacecraft — model a travel-time term shrinking per completed station.

Setup vs. execution split (unchanged): building/buying the spaceport,
positioning builders and Magnus, slotting policy cards can fully overlap
T_tech; only project execution is sequential.

**Culture.** min T subject to ∀r: FT_r(T) > DT_r(T) (G10). The threshold is
**endogenous** — your tourism drains the leader's domestic tourists, so the
finish line lowers as you convert. Apply tourism only to met civs; modifiers
structured national-multiplicative × per-rival-additive.

## 6. Time preference

The discount rate is not a constant: early yields compound at the empire's
reinvestment growth rate g(t) (~2–4%/turn during expansion, →0 mature);
near victory, a yield's value is purely whether it advances the critical
path. The optimizer computes ΔT(a) directly via the forward model; ρ
survives only as a human heuristic and search prior.

## 7. Optimality conditions

**O1. Bellman.** V(s) = 1 + min_a E[V(f(s,a))]. Intractable exactly; the
conditions below are its tractable first-order consequences, used for
pruning and scoring.

**O2. Marginal equalization.** Rank candidate purchases by ΔT/cost, spend
down the list against a **reservation price** r_G(t) (discounted best future
use — e.g. the planned Reyna spaceport buy). Greedy ranking is first-order
only for interacting actions; the solver replans after each action (MPC).

**O3. Shadow prices of local production.** λ_c(t) = (∂T/∂d_c)/π_c(t).
**All global currency flows to c* = argmax_c λ_c(t)**; λ_c = 0 off the
critical path (post-T_tech science game: spaceport cities only). Space-leg
production equation includes Pingala Space Initiative +30%.

**O4. Eureka stopping rule.** Cap pre-boost investment at 0.6·c_i (40%
boosts, G11); resume only if the boost is unreachable.

**O5. Phase switching.** Phases (expansion → infrastructure → victory leg)
are emergent from threshold nonlinearities, not imposed. Switch when
marginal ΔT crosses.

**O6. Assignment problems** (enumerated — small, discrete):
- Governors → cities: Magnus → argmax(pending chop value × λ_c), with
  **Provision** (settlers don't consume pop) for the settler pump and
  **Black Marketeer** (80% strategic discount) as needed; Pingala →
  argmax science+culture flow (**Librarian** +15% sci/cul, **Grants**
  +100% GPP, **Space Initiative** +30% space production); Reyna → argmax
  gold flow (**Tax Collector** +2 gold/citizen, **Forestry Management**
  +2 gold/unimproved feature, **Harbormaster** doubles CH/Harbor
  adjacency) or the district-buy target (E8).
- **Specialists:** +2 sci / +2 cul / +4 gold / +2 faith / +1 prod per slot
  (≤3 per district, **no GPP**) — per-slot terms in city yield equations.
- Policy cards → slots at each civic unlock; dedications → era
  (Monumentality rewrites E2/E7 rates — stockpile faith ahead of the
  window).
- Envoys → city-states by marginal equalization over suzerainty bonuses.
- Traders: queue-cost in home city, then a **mobile, reassignable** yield
  bundle valued at shadow prices; bundles rewritten by policy cards
  (Triangular Trade +4G+1F, Caravansaries +2G, Collectivization +4F
  domestic, Ecommerce +5P+10G international, Market Economy
  +1G/improved luxury+strategic at destination, +2C+2S international).
- **Unused GPP → faith at 1:1** (e.g. Great Prophet points after all
  religions founded) — faith-income term.
- **Luxury amenities: unique copies only, up to 4 cities each** — caps the
  amenity-supply equation.
- **Placement constraints (hard):** pop gates district count (1 + 1 per
  3 pop); one specialty per type per city; placing destroys features with
  no yield → chop first (G5).

## 8. Solver architecture

Two layers:

- **Strategic layer** (replanned every ~10 turns or at regime changes):
  sets shadow prices λ_c, gold reservation price r_G, governor/policy/
  dedication regimes; samples the stochastic shell (rival culture as
  estimated process from scouting + margin; combat in expectation, see
  below).
- **Tactical layer** (per turn, MPC): enumerates non-dominated actions,
  scores each by 1-step ΔT via the deterministic forward model, pruned by
  O2–O6, acts, replans.

Feasibility: loyalty for new cities, housing caps, strategic-resource
requirements. **Combat (shell):** base formula confirmed
(Combat_(Civ6), community research) —
  Damage = 30 · e^(0.04·StrengthDifference) · U(0.8, 1.2), floored at 1 —
but the model's old multiplicative wall factor is **dropped**: walls act
through the strength terms (e.g. ranged −17 RS vs cities), not as a
post-multiplier. Strength degrades with HP loss (round(10 − HP/10));
melee takes retaliation, ranged does not; modifiers are flat CS — so
expectation needs per-round updating, not a single-shot formula.

**Version discipline:** Fandom mixes rulesets across pages (Amenities page
carries both columns; Population page reads vanilla). Every equation above
records its ruleset column; anything sourced from a mixed page is flagged.

## 9. Calibration gaps (v3)

1. Exact faith purchase cost formula by item type (Fandom silent; keep
   empirical ≈2:1 + Theocracy legacy).
2. Settler production-cost scaling curve.
3. Gold-rush rounding-to-5 detail.
4. Rainforest base split: game XML says 10 prod + 10 food; Fandom's
   Builder-page example consistent; Fandom's Rainforest page "50 base"
   contradicts — want a direct XML read to close.
5. GS negative-side amenity values (bands confirmed shifted; values carried
   from R&F — want confirmation).
6. Trader "Switch City" timing (Fandom pages contradict each other).
7. Combat damage floor/verification (community-research status).

## 10. Changelog

- **v1 → v2:** amenity linear→step (later corrected again for GS); constant
  ρ → direct ΔT; shadow prices formalized; builder→Royal Society arbitrage
  derived; chop policy-stacking; district cost-locking; space setup/exec
  split; regime switches; maintenance/loyalty terms.
- **v2 → v3 (wiki audit):** GS amenity table rewrite (all-yields incl. food,
  band edges, Revolt −40%, non-capital requirement); culture-per-pop
  positional cycle; district discount 0.60× + empire-internal test; GS chop
  overflow rewrite (Magnus multiplicative, overflow stripped, modifiers
  fixed at last turn's end); project table rewrite (Festival not Bread and
  Circuses; non-uniform 15%/30% rates); tourism divisor 200 + civic boosts +
  endogenous finish line + met-gate; GS single Mars project + travel-time
  term + tech-gate list; G7 Oracle/Democracy/era-scaling; E8 first-copy case
  split; E9 Moksha paths; Pingala Space Initiative; repeat-purchase
  escalation; combat wall-factor dropped; specialists/governor/trade-card/
  luxury-cap/placement-constraint terms; exact housing table; loyalty×growth.
