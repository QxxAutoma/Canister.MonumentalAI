# MonumentalAI

*Enhanced AI for Civilization VI, model-driven. Named for Monumentality, the
golden-age dedication that makes the faith economy sing — and for the scale
of the ambition.*

A formal model of Civilization VI's economic core for computing optimal play
that minimizes expected victory turn E[T], for Science and Culture victories.
The model is the specification; any code is just its numerical solver.

Status: v4 (2026-11-10). Second full validation audit complete: every
numbered claim in G1–G14 and E1–E9 re-checked against
civilization.fandom.com, game XML mirrors, and official patch notes.
**Ruleset target: Gathering Storm.** Equations encode GS values;
vanilla/R&F divergences are tagged. Open calibration gaps listed in §9.

---

## 1. Problem statement

Civ 6 is a finite-horizon stochastic game. We decompose it into:

- **Deterministic core** (fully modelable): city yields, growth, production
  queues, tech/civic progression, great people, tourism accounting, and all
  currency exchanges. Formulas below are verified against game data or
  the wiki where a source is noted; unverified items are flagged inline
  and in §9.
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
  Φ(p) = 15 + 8(p−1) + (p−1)^1.5 food, values floored to integer (the
floored values reproduce the Population_(Civ6) required-food table
exactly; CivFanatics formula thread for the formula itself). Each citizen consumes 2 food/turn (Food_(Civ6)). Housing and
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
| Content (0 to +2) | — |
| Displeased (−1 to −2) | −15% growth, −10% non-food yields |
| Unhappy (−3 to −4) | −30% growth, −20% non-food yields |
| Unrest (−5 to −6) | growth halted*, −30% non-food yields |
| Revolt (≤ −7) | growth halted*, −40% non-food yields |

Requirement (GS): **required = ceil(pop/2) for every city**. Supply:
**Palace +2 in the capital (GS; +1 in vanilla/R&F)**, +1 per unique luxury
(up to 4 cities each; 6 in special cases such as Aztec/Zanzibar),
+entertainment etc. Net a_c = supply − required.
(Vanilla/R&F: required = ceil(pop/2 − 1); the GS change raises non-capital
requirement by 1 and raises Palace supply by 1, leaving the capital's net
unchanged. The v3 formula floor(pop/2)+1 overstated the requirement by 1 at
even pops; corrected 2026-10-14 against the Amenities page formulas and
worked examples: pop 5–6 requires 2 vanilla / 3 GS, pop 11–12 requires
5 vanilla / 6 GS.) Threshold jumps make amenity spending discrete: buy the
spending discrete: buy the luxury that crosses a breakpoint. (Provenance,
corrected 2026-11-10: the August 2020 patch notes set the non-food
percentages (−10/−20/−30/−40); the December 2020 update restored the
thresholds above and raised Happy/Ecstatic to +10%/+20%; the combination
matches the current Amenities page GS column exactly. Growth penalties
−15%/−30% are from that column. *Growth halted at ≥5 fewer rests on
Population_(Civ6) alone: single-source, flagged in §9.)

**G4. District cost.** Placed-district production cost (District_(Civ6), exact):
  D(type, t) = base · (1 + 9·max(T, C)) · f, where f = 0.60 for a
  discount-eligible specialty type, 0.75 for Government Plaza / Diplomatic
  Quarter, else 1. **Spaceport is the exception: fixed 1800, no progress
  scaling, never discounted.**
T/C = tech/civic completion fractions. **Cost locks at placement** (free,
instant action). Discount = **40% off** for specialty types (25% for
Government Plaza / Diplomatic Quarter), granted when the empire's count
of that type **placed or completed**, C(T), is below the empire-internal
average B/A (B = specialty districts completed, A = specialty district
types unlocked), gated on B ≥ A. B refreshes only when a district
finishes and then a tech or civic completes; C(T) updates on placement.
Non-specialty districts (Aqueduct, Dam, Canal, Neighborhood, Spaceport)
never receive the discount. Base costs: most 54, Aqueduct 36, GP/DQ 30,
Dam/Canal 81, Spaceport 1800. Derived strategy: place planned districts
ASAP to lock cost; keep type counts diversified below the internal
average.

**G5. Chops / harvests — GS model.** V(t) = v₀·(1 + 9ξ(t)), same ξ as
districts. Base: Forest 20 production; **Rainforest 10 production +
10 food** (direct XML read 2026-10-11: Feature_Removes table gives
FEATURE_JUNGLE 10 production + 10 food; Fandom Builder worked example
(61/61 unmodified, 122/122 with R&F Magnus, 91/91 with GS Magnus)
corroborates equal 10/10 bases; the Fandom Rainforest page's "50 base"
harvest claim is a confirmed editor error); Marsh 20 food. **Magnus
Groundbreaker +50% is a separate multiplicative layer** (GS; R&F was +100%).
**GS overflow rule** (Builder_(Civ6), Production_(Civ6)): the absorbed
portion of a chop gets the target's building modifiers; the **overflow
portion is stripped of policy/building modifiers** (Magnus's separate
multiplicative layer is part of the base yield); modifiers are fixed by
the construction **at last turn's end** (no within-turn swapping as in
R&F). **The exact carry amount is unresolved (reopened 2026-11-10):**
Production_(Civ6)'s worked table implies carry = max(0, V_base −
remaining), and 0 when only the modified yield completes the item, while
Builder_(Civ6)'s worked example implies carry = (V_modified −
remaining)/(1+ρ_policy): 66 vs 78.5 on identical structure. Needs in-game
measurement; use neither figure as exact (see §9.8).
Planted (second-growth) woods yield **zero** on removal — no chop-planting
arbitrage. Placing a district/building destroys features with no yield:
always chop first. **Overflow carry-over (no cap, confirmed):**
Civ VI has **no overflow cap** (caps are Civ IV/V mechanics); the wiki's
worked example shows a 100-production chop against a nearly finished item
leaving 99 base production as carry-over. **Zero-decay switching:**
per-item invested production persists indefinitely across queue
switches/reorders — treat per-item invested amounts as persistent state.
**Re-boost routing:** base overflow is enhanced by the *next* item's policy
bonus (chop a nearly-finished unit, then start Walls with Limes slotted →
carried base overflow doubles); sequence overflow into bonus-eligible
items. Chop with an empty queue: **unverified** — Production_(Civ6) is
silent and a CivFanatics report says the yield is wasted (§9.8 reopened).
Corner: Shift+Enter forced end-turn reportedly stores that turn's
production for exactly 1 turn, then wastes it, with policy-bonus portions
not carrying — single Steam-thread source, not asserted. Net:
chop-into-cheap-item banks the base layer without a cap or switching
decay; whether it is fully lossless depends on the unresolved carry
amount in §9.8. If the receiving item is a conversion project, overflow
behavior is untested — flag before routing overflow there.

**G6. Gold purchases.** Rush-buy = 4 × remaining production cost. (A
community formula thread reports rounding down to the nearest 5; no wiki
page states a rounding rule, so the rounding is unverified — §9.3.) The
4× multiplier is
corroborated twice: Fandom Gold_(Civ6) ("1 Gold is equivalent to 0.25
Production") and Fandom City_(Civ6) purchasing section ("the Gold cost of
an item is 4x its current Production cost", read 2026-10-19); the Faith
rate is 2× production the same way (City_(Civ6)) — the base rate is
corroborated, while the exact per-item faith formula stays empirical
(§9.1). Purchase prices are computed from
the item's production cost — discounts to production costs from policies
and other gameplay elements do **not** lower the purchase price (Fandom
City_(Civ6), read 2026-10-19). Modifiers: Merchant Republic (GS, while
adopted) discounts gold purchases by a flat 15%. (The accruing legacy
bonus, +1% per 15 turns retained after switching, is vanilla-only;
legacy bonuses were removed in Rise and Fall.) **Repeat civilian-unit
purchases escalate in price**, following the same scaling rules as
producing them (G12 for Settlers/Builders) — the marginal rate is not
constant across repeated buys.

**G7. Great-person patronage** (Great_People_(Civ6), exact):
  Faith = 150 + 10·(total − available); Gold = 200 + 15·(total − available)
  (fractional points rounded down).
Fixed components (150/200) reportedly do **not** scale with game speed
[single-source: CivFanatics formula thread, vanilla-era — provisional].
GPP `total` scales by (1 + 0.3·gap)^gap, gap = GP era − World Era when
the GP enters the queue (gap ≥ 0), **for all classes except art-related
People (Writer/Artist/Musician) and the Great Prophet**, which do not
scale this way.
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
  projects: 15% → gold. Dark Summoning (Secret Societies mode):
  30% → faith.
- GPP = D_TYPE·(1 + 7·DEVELOPMENT_RATIO), where DEVELOPMENT_RATIO is
  the higher of the unlocked-tech and unlocked-civic fractions (the same
  max(T, C) as G4); D_TYPE ∈ {10 most, 5 Festival/Carnival, 30 Occult
  Research (Secret Societies mode)}; GPP yield does **not** scale with
  game speed.

**G9. Royal Society.** 2% of project production cost per builder charge
(reduced from 3% in the Antarctic Late Summer Update); consumes **all** charges of the builder; once
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
- Win iff Σ_r FT_r > DT_r for every rival r (total visiting tourists
  exceed each rival's domestic count; per-rival dominance is not
  required). Tourism applies only to **met**
  civs (bilateral term needs a met-status gate). Modifiers are structured:
  national multiplicative × per-rival additive (trade route +25%, open
  borders +25%, government-difference penalty, …).

**G11. Eurekas / inspirations: 40%** of tech/civic cost in R&F/GS
(Boost_(Civ6); vanilla was 50%). Exceptions: China +10%, Free Inquiry /
Pen-Brush-Voice +10%, Babylon = full remaining cost, Near Future
Governance inspiration 90%.

**G12. Settler and Builder scaling (GS, Standard speed).**
Settler: C(n) = 80 + 30n. Builder: C(n) = 50 + 4n. Here n counts
previous copies of that unit, with n = 0 for the first trainable copy
(observed sequence 80, 110, 140…). Basis: the Units.xml rows give
Settler Cost 80 with COST_PROGRESSION_PREVIOUS_COPIES Param1 30, and
Builder Cost 50 with Param1 4; the Fandom Settler formula 30*(x+1)+50
reduces to the same 80+30n. Builder_(Civ6) states its count is copies
trained or purchased; the Settler page's text says trained, and its
variable definition (including the initial Settler) conflicts with its
own formula and the observed sequence — purchase-counting for Settlers
and the counting of starting/captured/free-granted copies stay
single-source (Fandom alone) and unasserted. The 20 increment was the
launch value; 30 applies from the Summer 2017 patch onward, across
rulesets. Game-speed scaling of Param1 is unverified, so this rule is
scoped to Standard.

**G13. Government Plaza and Ancestral Hall.** The Government Plaza
district is unlocked by **State Workforce** (not Political Philosophy);
base cost 30 (G4); +1 Governor Title on completion, +1 per building.
Ancestral Hall is the Tier 1 building (requires a Tier 1 government,
i.e. Political Philosophy): **+50% production toward Settlers in the
Plaza city only** (not empire-wide), and **every city founded after its
completion starts with a free Builder** — the free-builder effect is bound
to the building, not to the settler, so a settler trained before
completion still grants the builder if it founds after. Stacks with
Colonization and Magnus Provision (Ancestral_Hall_(Civ6),
List_of_districts_in_Civ6). Sequencing consequence: the Plaza can be
built and finished before Political Philosophy, so Ancestral Hall can
start the turn PP lands.

**G14. Era score and ages (standard GS ruleset; Dramatic Ages mode
excluded).** Dark/Normal threshold: base 12 earned in the era, +5 for
each prior Golden Age, −5 for each prior Dark Age, +1 for each city held
at the start of the age (Age_(Civ6)). Golden threshold: the Normal
threshold + 12 under the commonly reported formula (base 24 at game
start) — [single-source: CivFanatics R&F-era reports and derivatives;
Age_(Civ6) does not state a fixed 24 and describes the Golden threshold
differently, including an unexplained difficulty-based increase, so the
Golden base and any difficulty term are provisional]. The ±6-per-prior-age
variant in older forum reports conflicts with Age_(Civ6)'s ±5 and is
dropped. Era Score does not carry over an era change: thresholds for the
next era are computed at the change and overshoot above the Golden
threshold is wasted. First-era calibration from the user's Deity games
(2026-10-15, user-reported): goal 25, usually achieves 25–35; the golden
age is a managed target with fallbacks (buy a ship for exploration meets,
levy a city-state military), not a given.

## 4. Exchange operators — the currency board (GS)

| # | Conversion | Rate | Notes |
|---|---|---|---|
| E1 | gold → hammers | 1 : 4 | anywhere, instant (G6); Merchant Republic legacy improves it |
| E2 | faith → hammers (units) | ≈ 1 : 2 | Fandom: "1 Faith = 0.5 Production = 2 Gold"; Theocracy (GS, while adopted): 15% discount on faith purchases (the accruing legacy form is vanilla-only); **exact formula unverifiable — keep empirical**; Monumentality: settler/builder purchases 30% cheaper (§9.1, O6) |
| E3 | faith/gold → GPP | 10 F / 15 G per point | marginal (G7); Oracle/Democracy modify |
| E4 | hammers → sci/cul/faith/gold | **15% grants/prayers/festival; 30% investment/dark summoning** | (G8); Bread and Circuses excluded |
| E5 | food → pop → yields | Φ(p) food → tile yields + 0.5 sci + positional cul per turn, perpetuity | (G1, G2) |
| E6 | chop → hammers | V_eff per charge | (G5); Magnus multiplicative; GS overflow strip |
| E7 | builder charges → project hammers | 0.02 × project cost per charge | Royal Society (G9); the space-leg converter |
| E8 | gold → district | 4× the district's current production cost at purchase | Reyna Contractor (Fandom: scales parallel to production cost + 1G=0.25P; a purchase is the placement, priced at the current scaling cost); **case split: first-copy buys don't get the empire discount** (District_(Civ6)); non-first-copy factor is 0.60×, or 0.75× for Government Plaza / Diplomatic Quarter |
| E9 | faith → districts / faith income | Moksha Divine Architect (GS, Tier III) buys districts with faith; Citadel of God (Moksha Tier II title; faith term GS-only): gain faith equal to 25% of the construction cost when finishing buildings **in Moksha's city** (buildings only) | new purchase path + income term; the faith price of a Moksha district buy is derived from E2, not separately sourced (§9.1) |

**Derived arbitrage:** for the space leg, E7 via bought builders beats E1
iff C_builder < 0.08·n_charges·P_project in gold terms (≈2× headroom at
late-game prices). **Pingala Space Initiative: +30% production to all
space-program projects** applies alongside Royal Society injections in
the space-leg production equation; whether it multiplies the injection
amount itself is unverified.

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
  Accelerating that term requires the **Offworld Mission** technology,
  which unlocks both laser-station projects (+1 light-year/turn per
  completed station; Terrestrial stations count only while their city
  is Powered).

Setup vs. execution split (unchanged): building/buying the spaceport,
positioning builders and Magnus, slotting policy cards can fully overlap
T_tech; only project execution is sequential.

**Culture.** min T subject to ∀r: Σ_r FT_r(T) > DT_r(T) (G10). The threshold is
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
  +2 gold/unimproved feature (+1 Appeal to tiles adjacent to unimproved
  features), **Harbormaster** doubles CH/Harbor
  adjacency) or the district-buy target (E8).
- **Specialists:** +2 sci / +2 cul / +4 gold / +2 faith / +2 prod per slot
  (≤3 per district, **no GPP**) — per-slot terms in city yield equations.
  (Simplification: Encampment slots give +1 cul +1 prod and Harbor slots
  +2 gold +1 food; use those values for those two districts.)
- Policy cards → slots at each civic unlock; dedications → era
  (Monumentality: civilian units purchasable with faith, settler/builder
  purchases 30% cheaper — stockpile faith ahead of the window).
- Envoys → city-states by marginal equalization over suzerainty bonuses.
- Traders: queue-cost in home city, then a **mobile, reassignable** yield
  bundle valued at shadow prices; bundles rewritten by policy cards
  (Triangular Trade +4G+1F, Caravansaries +2G, Collectivization +2P+4F
  domestic in GS (Communism only; +4F alone pre-GS), Ecommerce +2P+5G
  from all routes in GS (+5P+10G international pre-GS), Market Economy
  +1G/improved luxury+strategic at destination, +2C+2S international).
- **Early-game gold calibration (user-reported, Deity, 2026-10-15):**
  luxury sale 13–16 GPT or ~200–280 lump; relic sale up to 43 GPT or
  lump equivalent (community sources on relic pricing are old and
  contradictory — keep as user-reported); first trader bought with gold,
  transferred to a satellite city (1 turn), domestic route for
  food/production to origin; pantheon by situation: Fertility Rites
  (free Builder + 10% growth) or Religious Settlements (free Settler +
  15% border growth) when available, else City Patron Goddess (+25%
  production toward districts in cities without a specialty district),
  God of the Open Sky (+1 culture per pasture), Goddess of the Hunt
  (+1 food +1 production per camp, GS), Lady of the Reeds and Marshes
  (+2 production per marsh/oasis/desert floodplain, GS; +1 pre-GS),
  Goddess of Festivals (+1 culture per plantation, GS) — the top two are contested on Deity since AI takes
  them first; first governor title
  to Amani for early suzerainty (era score + suzerain bonuses).
- **Unused GPP → faith at 1:1 (GS):** only for classes the civ can no
  longer attract (e.g. Great Prophet points after founding, or after the
  Prophet pool is exhausted); ordinary excess toward a still-recruitable
  class carries forward and is not converted — faith-income term.
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
requirements. **Combat (shell):** base formula confirmed (Combat_(Civ6)) —
  Damage = 30 · e^(0.04·StrengthDifference) · U(0.8, 1.2), with a floor
  of 1 that the wiki itself flags "verification needed" (§9.7).
The model's old multiplicative wall factor stays **dropped**, but walls
are not only a strength term: **+3 City CS per wall level (max +9)**
inside the strength terms, plus a separate outer-defense damage-reduction
layer (−85% melee, −50% ranged; bombard-strength units do full damage)
and a wall-HP shield on city HP (undamaged walls limit city damage to
1 HP per attack). The −17 ranged penalty applies against city/district
defenses generally, walled or not — it is not a wall effect. Strength
degrades with HP loss: CS penalty = −10·(100 − HP)/100 (max −10;
Samurai / Nihang with Sanjo exempt); melee takes retaliation, ranged
does not; modifiers are flat CS — so expectation needs per-round
updating, not a single-shot formula.

**Version discipline:** Fandom mixes rulesets across pages (Amenities page
carries both columns; Population page reads vanilla). Every equation above
records its ruleset column; anything sourced from a mixed page is flagged.

## 9. Calibration gaps and resolutions

1. Exact faith purchase cost formula by item type (keep empirical ≈2:1
   + Theocracy 15% while adopted; the base 2× rate is corroborated by
   City_(Civ6), the exact per-item application is not sourced).
   Verified around it: Monumentality
   (Golden Age dedication) makes civilian units purchasable with faith at
   30% cheaper settler/builder prices (Fandom Age page); builder faith
   cost observed 80–100 in the user's Deity games (2026-10-15,
   user-reported), consistent with ≈2× production × 0.7 for later builders.
2. **RESOLVED 2026-10-14** — Settler scaling (GS, Standard): C(n) =
   80 + 30n, n = previous copies trained or purchased; Builder 50 + 4n
   by the same XML pattern. Starting settler does not increment n.
   Vanilla Param1 was 20, which explains the old 80+20n formula. Open:
   whether captured or free-granted settlers increment n (single-source),
   game-speed scaling of Param1, and purchase-counting for Settlers (the
   Settler page's text says trained only).
3. Gold-rush rounding direction (floor-to-5 vs round-to-5) — narrowed
   2026-10-19: the 4×/2× multiplier, the remaining-cost basis, and the
   policy-production-discount exclusion are now corroborated (Fandom
   Gold_(Civ6) + City_(Civ6) purchasing section, direct reads). The
   rounding direction itself remains single-source (community formula
   thread says "rounded down to nearest 5"; Fandom's purchasing section
   states no rounding), so it is not asserted as fact in G6.
4. **RESOLVED 2026-10-11** — Rainforest base split: direct XML read
   (community mirror of the game files,
   Base/Assets/Gameplay/Data/Features.xml, Feature_Removes table) confirms
   FEATURE_JUNGLE harvest = 10 production + 10 food (same table: Woods =
   20 production only). The Fandom Rainforest page's full text was read;
   its "50 base" is a harvest-yield claim, an editor error contradicted by
   both the game data and the Fandom Builder page's own worked example
   (61/61 unmodified, 122/122 with R&F Magnus, 91/91 with GS Magnus —
   equal splits at every step, consistent only with equal 10/10 bases).
   Likely confused with the Builder unit's own 50-production cost. The
   model's 10/10 base stands.
5. **RESOLVED 2026-10-12, provenance corrected 2026-11-10** — GS
   negative-side amenity values: the August 2020 patch notes set the
   non-food percentages (Displeased −10%, was −5% R&F; Unhappy −20%, was
   −10%; Unrest −30%, unchanged; Revolt −40%, was −60%); the December
   2020 update restored the thresholds (Displeased −1..−2, Unhappy
   −3..−4, Unrest −5..−6, Revolt ≤ −7, Content 0..+2) and raised
   Happy/Ecstatic to +10%/+20%. The combination matches the Fandom
   Amenities page GS column exactly. Growth penalties −15% Displeased /
   −30% Unhappy are from that column (patch notes do not mention
   growth). Still single-source: growth halted at ≥5 fewer amenities
   (Population_(Civ6) only).
6. **RESOLVED 2026-10-15** — Trader "Transfer to another city" takes 1
   turn; route yields apply to the origin city (Fandom Trade Route page).
   Domestic routes yield food + production to origin. Encoded in O6 (no
   separate G-formula); route yield amounts stay open — item 10.
7. Combat damage floor: the floor of 1 is stated on Combat_(Civ6) with
   the wiki's own "verification needed" flag — not asserted (§8 carries
   the flag). The base damage formula itself is confirmed.
8. **PARTIALLY REOPENED 2026-11-10** — Production overflow carry-over.
   Confirmed and kept: no cap in Civ VI; zero-decay queue switching;
   re-boost routing. Reopened: (a) the exact carry amount —
   Production_(Civ6) and Builder_(Civ6) worked examples conflict (66 vs
   78.5 on identical structure), needs in-game measurement; (b)
   empty-queue chop — Production_(Civ6) is silent and a CivFanatics
   report says the yield is wasted, contradicting the 2026-10-10
   closure; (c) the Shift+Enter 1-turn storage corner is single-source
   (one Steam thread). Overflow behavior when the receiving item is a
   conversion project remains untested by the cited experiment thread;
   the wiki states no exception — flag before routing overflow there.
9. Era-score Golden threshold (G14): the base 24 and any difficulty term
   are single-source and conflict with Age_(Civ6)'s own Golden section;
   the Normal threshold terms (+5/−5 per prior age, +1 per city) are
   sourced. G7's game-speed note for patronage fixed components is
   likewise single-source (CivFanatics formula thread, vanilla-era).
10. Unmodeled systems the formulas depend on: loyalty supply/pressure
    (G1 uses loyalty bands but no loyalty formula exists in the doc);
    trade-route yield amounts (timing/origin resolved in item 6, amounts
    by distance and destination are not modeled). User-reported
    calibrations (luxury/relic sale prices in O6) stay user-reported
    only and are tracked here as such.

## 10. Changelog

- **v1 → v2:** amenity linear→step (later corrected again for GS); constant
  ρ → direct ΔT; shadow prices formalized; builder→Royal Society arbitrage
  derived; chop policy-stacking; district cost-locking; space setup/exec
  split; regime switches; maintenance/loyalty terms.
- **v2 → v3 (wiki audit):** GS amenity table rewrite (all-yields incl. food,
  band edges, Revolt −40%, non-capital requirement — superseded by the
  2026-10-14 G3 correction, below); culture-per-pop
  positional cycle; district discount 0.60× + empire-internal test; GS chop
  overflow rewrite (Magnus multiplicative, overflow stripped, modifiers
  fixed at last turn's end); project table rewrite (Festival not Bread and
  Circuses; non-uniform 15%/30% rates); tourism divisor 200 + civic boosts +
  endogenous finish line + met-gate; GS single Mars project + travel-time
  term + tech-gate list; G7 Oracle/Democracy/era-scaling; E8 first-copy case
  split; E9 Moksha paths; Pingala Space Initiative; repeat-purchase
  escalation; combat wall-factor dropped; specialists/governor/trade-card/
  luxury-cap/placement-constraint terms; exact housing table; loyalty×growth.
- **§9.8 close (2026-10-10):** production overflow carry-over verified — no
  cap in Civ VI, zero-decay queue switching, re-boost routing (next item's
  policy bonus applies to carried base overflow), empty-queue chop stored;
  G5 extended, §9 item 8 closed.
- **§9.4 close (2026-10-11):** rainforest harvest base confirmed by direct
  XML read (Feature_Removes: FEATURE_JUNGLE 10 production + 10 food; Woods
  20 production only); Fandom Rainforest page's "50 base" harvest claim is
  a confirmed editor error; Builder worked example (61/61, 122/122,
  91/91) corroborates equal 10/10 bases; G5 updated, §9 item 4 closed.
- **§9.5 close (2026-10-12):** GS negative-side amenity values confirmed
  by the official August 2020 patch notes (GS ruleset), which give
  non-food yield penalties Displeased −10%, Unhappy −20%, Unrest −30%,
  Revolt −40%, matching the Fandom Amenities page GS column exactly
  (growth penalties −15%/−30% for Displeased/Unhappy); G3 table stands
  with the non-food qualifier added to negative-side yield penalties,
  §9 item 5 closed.
- **§9.2 close (2026-10-14):** settler scaling confirmed — GS Units.xml
  gives Settler Cost 80 with COST_PROGRESSION_PREVIOUS_COPIES Param1 30
  (Builder 50 / Param1 4); Fandom formula 30*(x+1)+50 reduces to the same
  80+30n; starting settler does not increment n; vanilla Param1 20
  explains the old 80+20n figure; captured/free counting and
  game-speed scaling of Param1 stay open as single-source/unverified.
  G12 added, §9 item 2 closed.
- **G3 correction (2026-10-14):** amenity requirement corrected to
  ceil(pop/2) for every city with Palace +2 supply in the capital (GS); the
  v3 floor(pop/2)+1 formula overstated requirement by 1 at even pops, which
  had applied a phantom Displeased penalty to the capital at pop 2 in the
  opening sim. G3 updated.
- **§9.6 close + early-game engine (2026-10-15):** Trader transfer timing
  resolved (1 turn, yields to origin) — §9 item 6 closed. Government Plaza
  unlock corrected to State Workforce (was misstated as Political
  Philosophy); Ancestral Hall: +50% settler production in Plaza city only
  + free Builder in cities founded after completion (G13). Monumentality:
  faith purchase of civilian units, 30% cheaper settler/builder purchases;
  builder faith cost 80–100 (user-reported). Era-score base thresholds
  12/24 with per-city and prior-age modifiers (forum-sourced, GS modifiers
  unconfirmed); first-era target 25, achieved 25–35 (user-reported Deity
  calibration) (G14). Early-game gold calibration (user-reported): luxury
  13–16 GPT / 200–280 lump; relic up to 43 GPT / lump equivalent.
- **§9.3 narrowed (2026-10-19):** gold/faith purchase multipliers (4×/2×)
  corroborated by two independent wiki pages (Fandom Gold_(Civ6) and
  City_(Civ6) purchasing section, direct reads); added the
  policy-production-discount exclusion to G6. Rounding direction
  (floor-to-5 vs round-to-5) stays open as the single-source residual.
- **v4 second audit (2026-11-10):** culture win condition corrected to
  Σ_r FT_r > DT_r (G10, §5); G14 rewritten (Normal threshold sourced,
  Golden base 24 provisional, ±6 dropped, Dramatic Ages scoped out);
  Merchant Republic and Theocracy legacy accrual removed (vanilla-only;
  GS is a flat 15% while adopted); G4 discount test corrected to
  placed-or-completed counts, with the GP/DQ 0.75 factor and the
  B-refresh timing in the formula; G3 Unrest/Revolt bands filled
  (−5..−6 / ≤−7) and §9.5 provenance corrected (Aug 2020 percentages +
  Dec 2020 thresholds); G7 era-gap scaling scoped (art-related People
  and the Great Prophet excluded) and its game-speed note marked
  single-source; specialists +2 production (was +1); Collectivization
  and Ecommerce GS values corrected; Lady of the Reeds and Marshes +2
  production in GS; City Patron Goddess wording corrected; unused-GPP
  conversion scoped to classes the civ can no longer attract; combat
  strength-degradation formula corrected and walls rebuilt as +CS plus
  a damage-reduction layer, damage floor flagged unverified; G5
  overflow amount reopened (conflicting wiki worked examples),
  empty-queue chop and the Shift+Enter corner de-asserted; E8/E9
  wording, Secret Societies mode tags, the Antarctic Late Summer Update
  name, the Offworld Mission laser-station gate, and the
  DEVELOPMENT_RATIO definition added. Sim fixes in the same pass:
  eureka boost is an increment, and Revolt growth no longer exceeds
  Unrest.
