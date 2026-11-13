# MonumentalAI

*Enhanced AI for Civilization VI, model-driven. Named for Monumentality, the
golden-age dedication that makes the faith economy sing — and for the scale
of the ambition.*

A model and solver specification for Civilization VI's economic core,
targeting Science and Culture victories. The objective prioritizes winning
within a declared horizon, then minimizes conditional victory time (§1).

Status: **v5 (2026-11-13)** — documentation correctness and specification
revision. This repository contains the design, not an implemented solver
or a certified game-engine adapter. **Ruleset target: Gathering Storm,
Standard speed** unless a rule states otherwise; optional-mode rules apply
only when that mode is enabled. Vanilla/R&F divergences are tagged.
Every run must pin its exact build/data and scenario configuration (§2).
Evidence and unresolved calibration are recorded in [SOURCES.md](SOURCES.md)
and §9; analytical cases and pending engine probes are in
[VALIDATION.md](VALIDATION.md). Earlier audit summaries are historical,
and do not certify a complete executable forward model.

---

## 1. Problem statement

Civ 6 is a finite-horizon stochastic game. We decompose it into:

- **Deterministic core** (modelable once its inputs and rules are known):
  city yields, growth, production queues, tech/civic progression, great
  people, tourism accounting, and currency exchanges. Unresolved rules
  and omitted systems are explicit model limitations (§9), not exact
  transitions.
- **Stochastic shell** (estimated, sampled): AI rival behavior (notably
  rival culture rates), combat RNG, hidden map, and goody huts. Hidden
  information is represented by a belief over possible states (§2).

Each scenario fixes an initial turn t0, a horizon H, its ruleset and game
speed, and the permitted victory type(s): Science, Culture, or either.
Let W be the event that a permitted victory occurs by H, and let T_win be
its turn when W occurs. Any terminal result without a permitted victory
(including defeat or an opponent winning first), or reaching H without
a permitted victory, is failure. A permitted victory recognized
on turn H counts as success; within-turn ordering follows the engine.

The objective is to **lexicographically minimize**

  J(policy) = (Pr(not W), E[1_W · (T_win − t0)]).

The duration contribution is defined as zero on failure.
Thus the solver first maximizes the probability of winning within the
fixed horizon, then minimizes victory time among policies with the same
win probability. If Pr(W) > 0, the second component equals
Pr(W)·E[T_win − t0 | W], so its ordering at fixed Pr(W) is exactly the
ordering of conditional victory times. Failure has no fabricated
victory turn, and successful runs are not compared by timing alone when
their failure probabilities differ. H is a declared scenario parameter;
changing it defines a different objective.

Actions use the game clock: legal purchases, placements, and other
intra-turn decisions have duration zero; advancing a turn has duration
one. The transition advances both the game state and clock, including
the sampled shell. Search over intra-turn actions must use finite,
noncycling action bundles (§7.O1).

Victory-time reduction ΔT is a search heuristic only after candidates
have matching primary failure value. It compares their conditional
successful finish times under a specified continuation policy; it does
not replace J or assign a finite victory time to losing outcomes.

## 2. State representation

The state separates **local** city production and consumption from
**global** accounting, while retaining the spatial constraints that connect
them. A list of current yields is not sufficient state: the transition must
also retain invested stocks, counters, timers, and pending events.

Let x be the complete simulation state. The planner sees observations o,
not hidden rival or map data; its planning state s contains the observed
snapshot and a belief over unknown components of x. Shell samples must be
conditioned on those observations. The fields below are a required schema
for the supported actions, not a claim that a standalone simulator is
already implemented. An exact game-engine adapter retains the complete
engine snapshot, including additional engine fields, as authoritative;
a standalone calibrated mode must declare its supported systems (§8).

**Configuration and clock:** ruleset/build and enabled content/modes;
game speed, map/player configuration, civilization/leader abilities,
victory settings; evaluation start t₀ and fixed horizon H (§1); current
turn t, active player/turn phase, pending events, and per-turn action-use
flags. A repeatable rollout also records its shell model and random seed.

**Local** (city c):

- Population p_c, food stock, gross food and consumption; housing,
  amenity supply/requirement/allocation, loyalty, power and their sources.
- Owned/workable tiles and assignments; tile ownership, terrain,
  features (including planted woods), improvements, resources and
  adjacency inputs; districts placed versus completed, placement-locked
  costs, buildings, damage and other active modifiers.
- Selected production item and queue; invested production **per item**,
  unspent overflow and its base/modifier bookkeeping, and the construction
  and modifier snapshot relevant at the previous turn's end (G5).
- Gross yield components and modifier inputs for food, production, gold,
  faith, science, culture and tourism; specialists, great works/slots and
  other tourism sources. π_c(t) is a derived production rate, not a
  substitute for these inputs.
- Governor presence and effective promotions, linked to the empire-wide
  assignment/establishment state; local per-turn use flags such as the
  Royal Society injection limit.

**Global** (empire/world):

- Gold G(t), faith F(t), strategic-resource stocks, income, maintenance,
  commitments and scheduled changes; government, policy slots/cards,
  religion/beliefs, dedications and other active empire modifiers.
- Current research/civic selections, invested progress **per tech/civic**,
  pending science/culture overflow, unlocks and boost status/amounts.
  S(t) and K(t) are derived rates; they do not encode progress. Completion
  counts/denominators and any era-dependent costs remain explicit.
- District counts by type (placed and completed), unlocked-type counts,
  and the cached discount eligibility/counts plus refresh state needed
  by G4. Retain purchase/production counters for escalating units (G12),
  with acquisition provenance where the counting rule is unresolved.
- Units with identity/type, owner, position, remaining movement/actions,
  charges, health and relevant promotions/status. Builder charges alone
  do not establish that a chop or Royal Society injection is reachable.
- Governor titles/promotions, assignments and establishment/reassignment
  timers; envoys, suzerainties, luxury ownership/allocation, diplomatic
  contacts, agreements and expiry times; trade capacity, traders, routes,
  locations, paths and remaining route/transfer commitments.
- GPP stocks by class, available candidates, costs and competition state;
  era score, current age, era timing and prior-age/city-count inputs.
- For **each civilization**, lifetime culture including civic-boost
  contributions, bilateral lifetime tourism/visiting-tourist accounting,
  met status and bilateral/source-specific tourism modifiers (G10).
  Hidden quantities remain beliefs rather than privileged observations.
- Science-victory project prerequisites/completions, current spacecraft
  distance and speed, completed laser stations and their power-dependent
  activity; rival victory progress and all terminal outcomes.

**Accounting versus eligibility.** Gold and faith are empire stocks;
science/culture progress, GPP by class, and tourism by source/target have
their own accounts and cannot be exchanged freely. Applying or spending an account
requires an eligible item, city, unit, unlock and timing. Production, food,
housing and amenity effects remain city-local; each city processes one
production item at a time. Neither global accounting nor a high local
shadow price makes an otherwise illegal or unreachable action feasible.

## 3. Economic rules and evidence status (GS)

All rates are at Standard speed unless noted. Apply speed scaling only
where the rules provider has a supported formula. Inherited evidence
and open items retain their status in §9 and SOURCES.md.

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
5 vanilla / 6 GS.) Threshold jumps make amenity spending discrete: evaluate the luxury
that crosses a breakpoint. (Provenance,
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
Dam/Canal 81, Spaceport 1800. Early placement can lock a lower cost; compare that saving with
tile/feature loss, capacity use and changed discount opportunities. Type
diversification can help satisfy the discount test but is not mandatory.

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
arbitrage. Placement on a feature can destroy it with no harvest yield; compare
chopping first with charge/movement costs, delay and earlier cost-locking. **Overflow carry-over (no cap, confirmed):**
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

**G6. Gold purchases.** An ordinary purchase buys a complete eligible unit
or building; it does not pay off the remaining production in the queue.
The nominal undiscounted price is approximately 4 gold per production in
the item's **full current cost**, before production-only modifiers.
Production already invested does not reduce that price. For units, the
purchased copy is separate from the copy under construction: the official
manual's example buys an Archer immediately while the queued Archer still
has four turns left ([official manual, printed p. 96](https://steamcdn-a.akamaihd.net/steam/apps/289070/manuals/CIV_VI_25TH_ONLINE_MANUAL_ENG.pdf)).
The 4× gold and approximately 2× faith baselines are the purchase rates
reported by City_(Civ6); exact rounding and item-specific exceptions remain
calibration questions (§9.1, §9.3). Use the engine's quoted price for an
actual action, including unit-cost escalation (G12) and purchase discounts;
do not substitute remaining production or a production-policy multiplier.

**Government modifiers (GS):** Democracy gives a 15% discount on ordinary
gold purchases while adopted ([official April 2019 GS patch notes](https://store.steampowered.com/news/posts/?appids=289070%2C353370&enddate=1560877460&feed=steam_community_announcements)).
Merchant Republic gives **+10% gold in cities with an established governor
and +15% production toward districts**, not a gold-purchase discount.
The GS data deletes its vanilla gold-purchase modifier and Democracy's
vanilla patronage modifier, then assigns these replacement effects
([removed modifiers](https://github.com/AutoDevJapan/civ6-banseiki/blob/f0e993f8dd486620d9b41471456fd106b12c700b/tools/civ6-data-rs/fixtures/game/DLC/Expansion2/Data/Expansion2_RemoveData.xml#L337-L345),
[replacement government modifiers](https://github.com/AutoDevJapan/civ6-banseiki/blob/f0e993f8dd486620d9b41471456fd106b12c700b/tools/civ6-data-rs/fixtures/game/DLC/Expansion2/Data/Expansion1_Governments.xml#L277-L307)).

Purchasing requires the appropriate unlocks, prerequisites, currency,
placement and purchase permissions. Ordinary purchases do not inject
production into arbitrary wonders or projects; district purchases require
the separate governor abilities in E8/E9. The item appears immediately,
but movement and activation are separate legal actions: check their
availability rather than assuming a purchased Builder can contribute to a
project during the same turn. Building completion and queue cleanup must
follow the purchase transition; do not refund or reuse invested production
without a verified rule.

**G7. Great-person patronage.** For an eligible offered Great Person with
available points below the required total, the commonly documented
unmodified Standard-speed baseline is:
  Faith = 150 + 10·(total − available); Gold = 200 + 15·(total − available).
Patronage acquires that candidate; it is not an unrestricted purchase of
fungible GPP. Preserve the fixed fee when comparing it with earning points.
The reported fractional-point rounding and fixed components that do not
scale with game speed originate in a vanilla-era community formula thread;
use the engine's patronage quote for exact prices and other speeds (§9.9).

GPP total scales by (1 + 0.3·gap)^gap, gap = GP era − World Era when
the GP enters the queue (gap ≥ 0), **for all classes except art-related
People (Writer/Artist/Musician) and the Great Prophet**, which do not
scale this way. **Oracle discounts faith patronage by 25%. Democracy's
vanilla −50% gold-patronage bonus does not apply in GS:** the GS data
explicitly deletes DEMOCRACY_PATRONAGE_GOLD_DISCOUNT
([game XML](https://github.com/AutoDevJapan/civ6-banseiki/blob/f0e993f8dd486620d9b41471456fd106b12c700b/tools/civ6-data-rs/fixtures/game/DLC/Expansion2/Data/Expansion2_RemoveData.xml#L337-L345)).
Its ordinary gold-purchase discount (G6) is a separate effect; do not apply
that effect to patronage without a matching engine quote. Recheck candidate
availability and the quoted price immediately before recruitment because
rival recruitment changes the offered person and required total.

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

**G10. Tourism accounting — GS.** Keep the cumulative number of tourists a
civilization has **generated** separate from its current **domestic** stock.
Foreign tourism transfers tourists out of that domestic stock, so lifetime
culture alone cannot specify the culture-victory threshold.

**Authoritative game adapter.** At the same turn boundary for every player,
read DT_r from player r's Culture:GetStaycationers(), bilateral visiting
tourists FT_(i←r) from player i's Culture:GetTouristsFrom(r), and total
visiting tourists V_i from player i's Culture:GetTouristsTo(). These are
integer stocks, not fractional tourism progress. The game UI reads these
distinct values and sets the required visiting total to one more than the
largest domestic stock among alive major rivals outside the player's team
([pinned WorldRankings.lua, lines 1667–1690](https://github.com/mrobaczyk/civ6/blob/310319f5a7f42ac65b0979a3e7572129fb17cc69/Base/Assets/UI/PartialScreens/WorldRankings.lua#L1667-L1690)).
Thus the culture terminal test is V_i > max_r DT_r over those rivals.
GetStaycationers() already supplies the current domestic count: **do not
subtract foreign transfers from that engine result a second time**.

**Standalone forecasting approximation, pending engine fixtures.** For
Standard-speed GS, use separate ledgers:

  Generated_r ≈ floor((C_life,r + I_life,r) / 100)
  FT_(i←r) ≈ floor(L_(i→r) / (200·N_scale))
  DT_r ≈ max(0, Generated_r − Σ_(j≠r) FT_(j←r))
  V_i = Σ_(r≠i) FT_(i←r).

C_life excludes inspiration increments and I_life records them once;
avoid double-counting if an engine lifetime-culture query already includes
boosts. L_(i→r) is cumulative tourism actually applied to r, after that
turn's modifiers and met-status gate. Sum integer tourists **after** each
bilateral conversion, not fractional progress across targets. Track transfers
to every foreign destination, including rival-to-rival flows, to forecast a
rival's remaining domestic stock. A rival that generated 100 tourists and
lost 20 to foreign destinations has approximately 80 domestic tourists,
not 100.

The 100-culture and 200-tourism baseline constants are the documented GS
accounting values; the exact fixed-point/integer order, divisor population
N_scale, source-pool limits and elimination/liberation behavior are **not
established by the UI source above**. Calibrate these against adapter
snapshots before treating the standalone ledger as exact. In particular,
do not silently change N_scale to the number of currently alive rivals or
assume visiting totals remain monotone through eliminations. Forecasts
that depend on these unverified cases must report that uncertainty (§9).

Tourism applies only to met civilizations. Preserve source classes where
modifiers differ (for example, religious versus general tourism), apply
national modifiers to the relevant sources, and then apply the bilateral
modifiers for that target (trade routes, open borders, government difference,
etc.). Recompute the victory threshold after both culture generation and
tourist transfers: it can rise or fall.

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

These are discrete, legally constrained actions, not unrestricted exchanges
between currency balances. P denotes production, G gold, and F faith.
Unmodified purchase rates are reference costs; actual actions use the
engine's price and availability at execution, including the full item cost
(G6), modifiers, unlocks and placement. Purchases create items immediately;
movement, establishment, travel and activation can delay their use.

| # | Operator | Reference rate | Eligibility and timing |
|---|---|---|---|
| E1 | Gold → eligible unit/building | **4 G → 1 P-equivalent** (0.25 P/G), before discounts/rounding | Buy a full item in a city that can purchase it; prior production is not a discount (G6). This is not a project/wonder rush or transferable production stock. Democracy's GS gold-purchase discount applies through the quoted price; Merchant Republic has no such GS discount. |
| E2 | Faith → eligible item | **≈2 F → 1 P-equivalent** (≈0.5 P/F) | Requires an ability permitting that item/currency purchase. Monumentality permits faith purchases of civilian units and reduces Settler/Builder purchase prices by 30%; Theocracy's GS faith-purchase discount is 15% while adopted. Exact per-item prices and stacking need engine quotes (§9.1). |
| E3 | Faith/gold + available GPP → offered Great Person | Baseline **150 F + 10 F per missing point**, or **200 G + 15 G per missing point** | Acquires the current eligible candidate (G7), consuming the applicable points; not an arbitrary GPP balance top-up. Oracle affects faith patronage. Democracy's vanilla 50% gold-patronage discount is excluded in GS. |
| E4 | Local project production → per-turn science/culture/faith/gold | **15%** for Grants/Prayers/Festival; **30%** for Investment/Dark Summoning | Requires the corresponding district/project and occupies the city's production queue. Yield arrives while the project runs; its GPP award arrives on completion (G8). Bread and Circuses is not a culture converter. Dark Summoning requires Secret Societies mode. |
| E5 | Local food surplus → population → worked/passive yields | Φ(p) food toward the next population step (G1); +0.5 science and positional culture per citizen (G2) | Growth occurs at the city's food threshold, subject to housing, amenities and loyalty. The new citizen consumes 2 food/turn; additional worked yields require an available tile/specialist slot. Evaluate yields through the planning horizon, with the changed amenity requirement. |
| E6 | Builder removal/harvest → local production/food | V_eff per consumed charge (G5) | Requires a legal removal/harvest action and Builder at the target. Yields are immediate and city-local; production target, modifiers and unresolved overflow rules constrain their use (§9.8). |
| E7 | Builder charges → eligible project production | **0.02·P_project per charge** (G9), before any unverified interaction | Requires Royal Society, an eligible ongoing project and a legal Builder contribution action. Consumes all remaining charges; at most one Builder contribution per city per turn. Buying, moving and positioning the Builder are separate actions. |
| E8 | Gold → district | Nominal **4 G per P of district cost**, subject to the actual quote | Requires Reyna with Contractor established in the city and a legal district placement, unlock and capacity. District-specific scaling/discount eligibility must follow G4 and the purchase quote; a non-first copy does not automatically qualify for the 40% discount. Do not treat a placed district's remaining construction as an ordinary rush-buy. |
| E9 | Faith → district; local building completion → faith | Moksha district price from the engine; Citadel of God grants **25% of building construction cost as faith** | Divine Architect (GS, Tier III) requires Moksha established in the purchase city and a legal district placement; ≈2 F/P is a baseline, not a separately verified district formula (§9.1). Citadel of God (Tier II; faith term GS-only) pays when a building finishes in Moksha's city, not empire-wide. |

**Royal Society cost comparison.** Let Q_G be the full quoted gold price of
one Builder, n its remaining charges when consumed, and P_project the
project's full production cost. Under G9's baseline, with all contributed
production useful and no additional modifier, H_RS = 0.02·n·P_project.
Its effective cost is Q_G/H_RS gold per production. This is below the
**undiscounted reference** 4 G/P iff:

  Q_G < 0.08·n·P_project.

This compares costs; it is not an executable gold-to-project alternative
or a sufficient condition for a faster victory. Evaluate the Builder's
purchase permissions, spawn/use timing, travel, other uses, city contribution
limit and project completion schedule. Cap useful production at the amount
actually accepted if contributions would overshoot. If comparing against a
discounted eligible purchase, replace 4 G/P with that purchase's effective
rate instead of keeping 0.08. Pingala Space Initiative supplies +30%
production to space projects; whether it multiplies Royal Society injections
is unresolved, so do not silently apply it to H_RS (§9).

## 5. Victory conditions and schedules (GS)

Both victory types use the success/failure objective in §1. Their
completion times come from the joint turn simulation, with research,
city production, resource deliveries, and opponent progress advancing
together.

**Science.** Use a precedence schedule, rather than adding all research,
project execution, and travel durations. The mandatory project chain is
Earth Satellite → Moon Landing → Mars Colony → Exoplanet Expedition
([official Gathering Storm manual, p. 11](https://cdn.2kgames.com/manuals/civilization-vi/gathering_storm/2KGMKT_CIV6_Gathering_Storm_PC_Online_Manual_V7_US.pdf)).
GS has one Mars project; the parallel Mars modules belong to vanilla/R&F.

For remaining mandatory milestone k in a proposed schedule, let r_k be its
technology-unlock turn, e_k its earliest feasible city/queue setup turn,
b_k its start turn, and C_k its completion turn. Omit already completed
milestones and use remaining work for projects already in progress. With
C_0 = t0:

  b_k = max(r_k, e_k, C_(k−1));   C_k = b_k + d_k.

Here d_k is the duration produced by the turn simulation for that start
and action schedule, including changing production and legal injections;
it is not a constant independent of future decisions. The setup term
includes the selected city's Spaceport, queue availability, and other
legal prerequisites. Research gates and setup are simulated concurrently
with earlier projects: an early project need not wait for every later
technology. Project rewards and yield changes feed back into subsequent
unlock times, so r_k is recomputed with the schedule.

The gates are Rocketry (Spaceport), Satellites (Moon Landing),
Nanotechnology (GS Mars Colony), and Smart Materials (Exoplanet
Expedition). Acceleration additionally requires Offworld Mission, which
unlocks both laser-station projects. Research is therefore valued
through its remaining unlocks, rather than set to zero after one blanket
T_tech cutoff.

After Exoplanet Expedition completes, accumulate travel distance D in
the engine's event order:

  D(t+1) = D(t) + v(t),   T_science = first victory check with D ≥ 50.

Initialize D and the expedition's baseline speed from the game state.
Completed Lagrange and Terrestrial stations each add +1 light-year/turn;
Terrestrial contributions count only while the relevant city is Powered.
Compute v(t) from the active effects each turn, not from a final station
count applied retroactively. Laser construction uses legal per-city
queues and does not inherit the mandatory chain's sequential ordering.
Research, station construction, builder deliveries, and expedition travel
can overlap when their prerequisites allow it. The simulator must also
check opponent victory and defeat during travel.

**Culture.** At each victory check, success requires total visiting
tourists to exceed every eligible rival's domestic tourists (G10):

  Σ_r FT_r(T) > DT_j(T) for every rival j.

Compute integer tourists, transfers, rival culture growth, and bilateral
modifiers using G10. Tourism applies only to met civs. The finish line
is endogenous: transfers can lower a rival's domestic tourist count,
while new culture can raise it. Track every rival rather than assuming
the currently leading rival remains the leader. Evaluate this condition
alongside competing victory and failure events under §1.

## 6. Time preference

The objective has no separate fixed discount rate. Early yields can
compound through reinvestment, while late yields matter through their
effect on success probability and the remaining victory schedule.
Those effects are evaluated by the forward model.

For candidates with matching primary failure value, ΔT may summarize
the change in conditional successful finish time under the chosen
continuation policy. A heuristic discount rate ρ may help order search,
but it is not an additional objective, a proof of dominance, or a reason
to prune actions that improve the primary win-probability objective.

## 7. Decision rules and search heuristics

**O1. Time-aware Bellman objective.** Let s include the current turn t,
and let F(s,a,ω) produce the successor state with
t' = t + δ(s,a), where ω is a sampled shell outcome. For the fixed
planning state, F returns the next observation-conditioned belief; it does
not reveal sampled hidden world facts to the next action choice. For the
scenario t0 and H, define the terminal cost vector:

  V(success at t) = (0, t − t0)
  V(any terminal result or timeout without permitted success) = (1, 0).

At a nonterminal state:

  V(s) = lexmin_(a ∈ A(s)) E_ω[V(F(s,a,ω))].

Expectation is componentwise; lexicographic comparison first minimizes
the failure component and then the successful-duration component. Time
is charged through the terminal timestamp, so there is no extra +1 for
each action. Intra-turn actions have δ = 0 and an end-turn transition
has δ = 1. To keep the finite-horizon recurrence well founded, enumerate
finite action bundles ending in an end-turn or terminal transition;
eliminate reversible zero-time cycles and repeated equivalent states.
Legality and the engine's event order determine which bundles exist.

Exact solution is intractable at full scale. Approximate search uses
declared rollout policies and horizon handling to estimate both
components of V. The heuristics below order candidates; they do not
establish exact optimality or justify pruning without a valid dominance
bound that respects the objective.

**O2. Purchases, bundles, and reserves.** Rank purchases by their estimated
effect on V. Among candidates with matching failure value, ΔT/cost is a
useful ordering heuristic against a gold reservation price r_G(t), such
as gold reserved for a planned Reyna Spaceport purchase. Evaluate
affordable bundles and future commitments before spending: even
independent indivisible purchases form a budgeted selection problem, so
greedy ratio order can miss the best bundle. Complementary actions also
need joint evaluation. Replanning after each action updates estimates
but does not recover already-spent currency.

**O3. Shadow prices and support flows.** For a fixed continuation with
positive success probability and matching failure value, let
M(s) = E[T_win − t0 | success]. Define a
local production shadow price by an explicit feasible perturbation:

  λ_c(t; ε) = (M(s) − M(s ⊕ ε production in city c)) / ε.

Its units are turns saved per production. The perturbation adds
production through a specified legal route to a specified item at the
current time; it is not freely transferable stock. Use finite
differences because completions, unlocks, and other thresholds make
derivatives discontinuous or zero between breakpoints. If the
perturbation changes failure probability, compare V directly instead.

Follow the entire conversion and delivery path when valuing a city.
A city without a Spaceport can have positive value by supplying
builders for Royal Society, earning gold for a purchase, or supporting
another prerequisite. Currency allocation therefore depends on the
purchase menu, discounts, item limits, timing, and remaining budget;
there is no rule sending all global currency to one city. Recompute
values as bottlenecks change. Include Pingala Space Initiative's +30%
space-project production where applicable; its interaction with Royal
Society injections remains a calibration gap (§4).

**O4. Eureka and inspiration timing.** If a boost supplies fraction b_i
of a technology/civic's cost c_i, (1−b_i)c_i is a candidate pre-boost
pause point, not a hard cap; use the applicable G11 fraction. Compare
continuing now with waiting for the boost, including its arrival-time
distribution, probability, required actions, alternative research, and
the unlock's effect on the victory schedule. Resume when waiting has
worse objective value even if the boost remains reachable. For example,
a 100-science technology with 60 invested and 10 science/turn finishes
in four turns without its 40% eureka; a eureka arriving in twenty turns
does not justify a sixteen-turn delay to a critical unlock. Add the boost
amount once to existing progress; do not reset accumulated investment.
Handle completion and overflow in the engine's event order.

**O5. Phase switching.** Expansion, infrastructure, and the victory leg
are descriptions of the chosen plan, not imposed phases. Re-evaluate
candidate bundles when success probability, critical prerequisites, or
marginal successful-duration values change. Thresholds can make a
multi-action investment useful even when its first action alone has no
immediate ΔT benefit.

**O6. Assignment problems** (enumerated where tractable):
- Governors → cities: evaluate complete assignments and the timing of
  legal reassignment/establishment. Magnus's pending chop value × λ_c
  is a candidate-ordering heuristic, with **Provision** (settlers do not
  consume population) for a settler pump and **Black Marketeer** (80%
  strategic discount) as needed. Pingala's **Librarian** (+15% science
  and culture), **Grants** (+100% GPP), and **Space Initiative** (+30%
  space production) are valued through the current objective, rather
  than maximizing unweighted science+culture. Reyna's **Tax Collector**
  (+2 gold/citizen), **Forestry Management** (+2 gold/unimproved feature,
  +1 Appeal to tiles adjacent to unimproved features), **Harbormaster**
  (double Commercial Hub/Harbor adjacency), and district purchasing (E8)
  similarly depend on timing and planned uses of gold.
- **Specialists:** +2 science / +2 culture / +4 gold / +2 faith /
  +2 production per slot (≤3 per district, **no GPP**), subject to the
  district's actual slots. Encampment slots instead give +1 culture
  +1 production; Harbor slots give +2 gold +1 food. Compare these yields
  against the citizen's alternative tile assignment and growth effects.
- Policy cards → slots at legal policy-change events; dedications → era.
  Monumentality makes civilian units purchasable with faith and makes
  settler/builder purchases 30% cheaper. Compare stockpiling for that
  window with current uses of faith and the chance of reaching the age.
- Envoys → city-states: evaluate yield and suzerainty thresholds, rival
  responses, and complementary envoy bundles.
- Traders: queue-cost in the home city, then a mobile yield bundle
  valued through its origin and route commitments. Policies modify the
  bundle: Triangular Trade +4G+1F; Caravansaries +2G; Collectivization
  +2P+4F domestic in GS (Communism only; +4F alone pre-GS); Ecommerce
  +2P+5G from all routes in GS (+5P+10G international pre-GS); Market
  Economy +1G/improved luxury or strategic resource at the destination
  and +2C+2S international. Route durations, reassignment timing, and
  yield amounts must be supplied by the transition model (§2, §9).
- **Early-game gold calibration (user-reported, Deity, 2026-10-15):**
  luxury sale 13–16 GPT or approximately 200–280 lump; relic sale up to
  43 GPT or lump equivalent (community sources are old and
  contradictory). A reported opening buys the first trader with gold,
  transfers it to a satellite city (one turn), and uses a domestic route
  for food/production at the origin. These are scenario observations,
  not fixed exchange rates or mandatory opening actions.
- Pantheon candidates include Fertility Rites (free Builder +10%
  growth), Religious Settlements (free Settler +15% border growth),
  City Patron Goddess (+25% district production in cities without a
  specialty district), God of the Open Sky (+1 culture/pasture), Goddess
  of the Hunt (+1 food +1 production/camp, GS), Lady of the Reeds and
  Marshes (+2 production/marsh, oasis, or desert floodplain in GS; +1
  pre-GS), and Goddess of Festivals (+1 culture/plantation, GS).
  Availability and the scenario determine the choice. Early Amani for
  suzerainty and era score is a candidate plan, not a fixed first title.
- **Unused GPP → faith at 1:1 (GS):** only for classes the civ can no
  longer attract (e.g. Great Prophet points after founding, or after the
  Prophet pool is exhausted). Ordinary excess for a still-recruitable
  class carries forward and is not converted; include the applicable
  faith income in forecasts.
- **Luxury amenities:** unique copies only, up to four cities each,
  subject to G3's exceptions. Evaluate the cities actually receiving
  amenities and discrete happiness thresholds before valuing a trade.
- **Placement constraints (hard):** population-limited district capacity
  is 1 + floor((p_c−1)/3): one at population 1, two at 4, three at 7.
  Only districts that consume this capacity count toward it; type-specific
  exceptions and placement restrictions come from the engine's legality
  rules. Enforce per-city type limits and all other prerequisites.
  Placement removes features without harvest yield (G5); compare
  harvesting first against builder charges, movement, delay, and the
  opportunity to lock a district's cost before deciding.

## 8. Solver architecture

This is an implementation contract. No solver, engine adapter, or complete
standalone simulator is included in this repository. A solver must declare
which rules and transitions it can reproduce before reporting an exact
forecast.

Two layers share the state in §2 and the objective in §1:

- **Strategic layer** (replanned about every 10 turns, or when an unlock,
  war, governor establishment, age change, or victory milestone changes
  the feasible plan): proposes governor/policy/dedication regimes, reserves,
  city roles, and candidate production/research schedules. Samples hidden
  world state and rival behavior from the current belief state.
- **Tactical layer** (after each meaningful intra-turn action and each turn
  boundary): enumerates legal actions and affordable bundles, rolls them
  forward under comparable shell samples, compares the expected objective
  vectors, acts, observes, and replans. O2–O6 order search; only legality
  checks and proved dominance may remove candidates. One-step yield or
  ΔT estimates do not establish dominance over delayed investments.

### 8.1 Rules and transition interfaces

The following interfaces describe required behavior, not existing APIs:

| Operation | Required result |
|---|---|
| `snapshot` | State/observation, current turn and phase, rules manifest, and visibility/provenance of each field. Hidden facts remain in the belief state. |
| `legal_actions(state)` | Actions with prerequisites, targets, resource costs, placement/occupancy limits, movement/charge requirements, and elapsed time. Includes `EndTurn` and excludes no-op cycles. |
| `purchase_quote(state, item, city, currency)` | Full-item price, eligibility, discounts, rounding, and effects on existing invested production. The quote is independent of invested production as a price discount (G6). |
| `city_yields(state, city)` | Gross tile, specialist, district/adjacency, building, trade, population, and other yields; named modifier layers; food consumption; maintenance; resource and power effects. |
| `apply_order(state, action)` | New simulated state and an event log for an intra-turn action; updates costs, charges, placements, counters, and timer starts without advancing a turn unless the action explicitly has that duration. |
| `advance_turn(state, shell_events)` | Next-turn state and ordered event log, including yields, stocks, growth, production/overflow, research/civic progress and boosts, completion rewards, timers, tourism, flight distance, and terminal checks. |
| `terminal(state)` | Target victory, other victory/defeat, timeout, or ongoing, as required by §1. |

**Engine-backed mode:** an adapter reads authoritative snapshots, legal
actions, quotes, yield breakdowns, and victory counters from the selected
game build. Rollouts use an isolated state copy and a compatible transition
provider; reading a live snapshot alone is not a forward simulator. Search
must not apply hypothetical orders to the live game.

**Standalone mode:** a transition provider implements the supported subset
from versioned data, equations, and measured fixtures. For every proposed
action it must either reproduce all dependencies above, obtain a trusted
rule result, or expose an uncertainty interval/unsupported dependency.
Missing yield terms, placement rules, loyalty, or trade-route amounts are
not silently treated as zero. Unsupported paths cannot be certified
dominated. Calibration limitations belong in §9 and in the forecast.

### 8.2 Turn boundaries and accounting

Each provider publishes the phase order for its rules manifest. At minimum
it distinguishes orders in the current turn, the committed end-turn
boundary, ordered actor/resource/completion events, and the next decision
state. Zero-time orders preserve `t`; `EndTurn` advances `t` by one.
Multi-turn commitments are expanded into turn transitions when another
decision, event, or terminal condition can intervene.

All flows use that provider's event order: record gross and net yields;
debit food consumption and maintenance; apply food/growth modifiers to
their specified layers; allocate production to the current item; process
completion rewards, overflow, and research/civic unlocks; update bilateral
tourism and powered flight acceleration; advance timers and check terminal
conditions at the game-defined phases. This list names dependencies; it
does not assert an unmeasured universal Civ VI phase order. In particular,
an unlock changing costs or production modifiers during a boundary must
use the same before/after snapshot as the target build.

Preserve purchased-unit production independently from trained-unit
production (G6), and preserve invested production for switched items (G5).
Building/district purchases that invalidate queued work need an explicit
queue-resolution result from the provider; no refund rule is assumed.
Treasury and faith satisfy stock-flow accounting with each income and
expense recorded once. Science/culture progress, inspiration increments,
and lifetime-culture totals must not double-count the same boost (G10).

Validate the transition contract against recorded before/after snapshots,
including 1/100 versus 99/100 research progress, multiple instant orders,
purchase versus training, district population boundaries, governor
establishment, project completion rewards, and tourism/victory thresholds.
See [VALIDATION.md](VALIDATION.md) for analytical checks and pending probes.

### 8.3 Combat and version discipline

Feasibility includes loyalty for new cities, housing, unit movement and
occupancy, strategic resources, policy slots, governor establishment, and
project/purchase prerequisites. **Combat (shell)** uses per-round sampled
outcomes in rollouts. Its mean is a search prior, since evaluating only
mean damage loses survival probabilities and other nonlinear effects.
The existing combat model (Combat_(Civ6)) is:

  Damage = 30 · e^(0.04·StrengthDifference) · U(0.8, 1.2), with a floor
  of 1 that the wiki itself flags "verification needed" (§9.7).

Walls contribute **+3 City CS per wall level (max +9)** inside the strength
terms, plus an outer-defense damage-reduction layer (−85% melee, −50%
ranged; bombard-strength units do full damage) and a wall-HP shield on city
HP (undamaged walls limit city damage to 1 HP per attack). The −17 ranged
penalty applies against city/district defenses generally, walled or not.
Strength degrades with HP loss: CS penalty = −10·(100 − HP)/100 (max −10;
Samurai / Nihang with Sanjo exempt); melee takes retaliation, ranged does
not. Recompute applicable terms after each attack. Provider fixtures must
confirm the formula, exceptions, rounding, and RNG law for the target build
before combat predictions are certified.

Fandom pages and older Civilopedia text can mix rulesets. Every run records
its exact build and data manifest (§2); [SOURCES.md](SOURCES.md) distinguishes
primary documentation, pinned game-data mirrors, and legacy wiki evidence.
An audit date alone does not certify an executable transition provider.

## 9. Calibration gaps and resolutions

These are evidence and implementation limits, not implicit zero-cost or
zero-yield rules. The exact engine-backed result takes precedence for a
matching rules manifest. A standalone provider must expose alternatives,
uncertainty, or an unsupported dependency until the relevant fixture exists.

1. **Faith purchases:** the approximate reference is 2 F per P, with
   Theocracy's 15% GS faith-purchase discount while adopted. Exact prices,
   item eligibility, Monumentality stacking and Moksha district purchases
   require quotes/calibration (G6, E2, E9). Builder faith prices of 80–100
   in the user's Deity games (2026-10-15) remain user-reported observations.
2. **Settler/Builder scaling — base formula retained; counting open:**
   Standard-speed C_settler(n)=80+30n and C_builder(n)=50+4n are the G12
   baselines. The earlier closure overstated certainty about what advances
   n. Starting, purchased, captured and free-granted acquisition counting
   for Settlers, and speed scaling of the increment, require a versioned
   fixture. Preserve acquisition provenance; do not infer n from the
   number of units currently alive. G12's uncertainty applies here too.
3. **Gold purchase basis corrected 2026-11-13:** G6 uses a full-item
   quote, never the remaining production as a price discount. The official
   manual supports separate purchased and trained units. The nominal 4 G/P
   baseline is inherited from the cited wiki; exact per-item rounding,
   exceptions and discount stacking remain calibration items. The earlier
   remaining-cost assertion is withdrawn. Purchasing a unique building or
   district also requires its own queue-resolution fixture (item 12).
4. **Rainforest base split — prior audit retained:** the 2026-10-11 audit
   reported Feature_Removes values of 10 production +10 food for jungle,
   and 20 production for woods, consistent with the Builder wiki example.
   The raw XML snapshot used for that audit is not stored here; record
   the active build/data hash when validating G5. The conflicting wiki
   rainforest base claim does not override a verified game-data result.
5. **Amenities — prior audit retained:** the August 2020 non-food yield
   percentages combined with December 2020 thresholds and positive-side
   bonuses are the provenance for G3. Its growth penalties come from the
   Amenities page; halted growth at five or more fewer amenities remains
   single-source (Population page). Exact modifier order and boundary
   behavior need recorded growth fixtures for the active build.
6. **Trader timing — prior audit retained:** the reported one-turn
   transfer and origin-city domestic food/production are encoded in O6.
   Route amounts, duration/renewal commitments and reassignment legality
   must be supplied by the transition provider; see item 10.
7. **Combat:** the wiki marks the damage floor of 1 as requiring
   verification. Confirm RNG law, rounding, floor, walls and exceptions
   against the active build before certifying the §8.3 combat transition.
8. **Production overflow — reopened items retained:** no-cap overflow,
   persistent invested production across queue switches, and re-boost
   routing are the prior audit's supported model. The exact chop carry
   amount remains disputed by the Production and Builder worked examples
   (66 versus 78.5). Empty-queue chops, forced-end-turn storage and project
   conversion receivers also need fixtures. Do not assert either carry
   formula or fully lossless banking as exact (G5).
9. **Ages and patronage:** G14's Golden base 24 and any difficulty term
   remain provisional. G7 patronage rounding, fixed-fee speed behavior
   and per-item discount applicability require engine quotes; GS Democracy's
   vanilla patronage bonus is removed, not an open question. Its ordinary
   gold-purchase discount must not automatically be applied to patronage.
10. **Standalone dependencies:** loyalty supply/pressure and trade-route
    amounts are not specified by complete formulas here. Neither are all
    tile/building/adjacency/yield modifier layers. §8 defines the required
    transition interfaces, but an implementation and its supported profile
    remain to be provided. User-reported sale prices in O6 are scenario
    observations and must not replace a diplomacy model.
11. **Tourism:** the authoritative adapter uses the engine's integer
    domestic, bilateral and total visiting counters (G10). The standalone
    ledger needs calibration for N_scale, fixed-point/rounding order,
    source-pool limits, inspiration/lifetime-culture accounting and
    elimination/liberation/team behavior. Track rival-to-rival transfers;
    never subtract transfers twice from an engine domestic count.
12. **Purchase and Royal Society transitions:** verify building/district
    queue cleanup or refund behavior, governor district-price discounts,
    spawn movement/activation, accepted production near project completion,
    and Pingala's interaction with Builder injections. E7's cost inequality
    is a reference comparison, not proof of a feasible or faster rush path.
13. **Rules profile and turn order:** no exact executable build profile,
    adapter or standalone event sequence has been certified. Record the
    data/mod manifest, transition version and before/after event traces
    required by §2/§8 and VALIDATION.md before claiming exact forecasts.

## 10. Changelog

The older entries below record prior audit conclusions, including ones
superseded by v5. References to earlier simulations describe work reported
in those audits; their code and logs are not present in this repository.

- **v5 correctness/specification revision (2026-11-13):** full-item purchase
  pricing replaces remaining-production pricing; GS government bonuses
  corrected; exchange directions/units and legal operator scope made
  explicit; tourism generated/domestic/visiting stocks separated, with
  engine counters authoritative and standalone limits disclosed. Added
  sufficient state and transition interfaces, a fixed-horizon objective
  defined on losses, zero-time action semantics, overlapping science
  milestone/flight scheduling, bundle-aware search, finite-difference
  production values including support cities, and timing-aware boosts.
  District capacity uses the population-1 offset. Categorical purchase,
  placement and chop rules become evaluated alternatives. README, sources
  and validation cases added. This revises the specification; it does not
  report completed gameplay calibration or an implemented solver.

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
  80+30n; the starting-settler counting conclusion is superseded by v5; Param1 20
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
  reported by two pages of the same wiki (Fandom Gold_(Civ6) and
  City_(Civ6) purchasing section, direct reads); added the
  policy-production-discount exclusion to G6. Rounding direction
  (floor-to-5 vs round-to-5) stays open as the single-source residual.
- **v4 second audit (2026-11-10):** culture win condition corrected to
  Σ_r FT_r > DT_r (G10, §5); G14 rewritten (Normal threshold sourced,
  Golden base 24 provisional, ±6 dropped, Dramatic Ages scoped out);
  Merchant Republic and Theocracy legacy accrual removed; the flat-15%
  Merchant Republic claim is superseded by v5 (Theocracy remains 15%); G4 discount test corrected to
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
