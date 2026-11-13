# Validation cases

These cases make the v5 corrections checkable. Analytical examples verify
the mathematical specification; they do not certify game mechanics or an
implemented solver. No game-engine calibration suite is included yet.

## Analytical cases

| Case | Assumptions/input | Expected result |
|---|---|---|
| Full-price unit purchase (G6) | Item base production 100; 90 already invested; gold multiplier 4; no discounts or rounding adjustment | Purchase quote is 400 gold, not 40; existing training investment is a separate account. |
| Explicit currency units (E1/E2) | Unmodified base conversion | 4 gold buys 1 production-equivalent; approximately 2 faith buys 1 production-equivalent for eligible purchases. |
| Instant actions (O1) | Two legal non-repeating zero-time orders followed by `EndTurn` | The clock advances by exactly one turn. |
| Loss handling (§1/O1) | A: win probability 0.9 at elapsed turn 20; B: 0.8 at elapsed turn 10; all other outcomes fail before H | Objective vectors are A=(0.1,18), B=(0.2,8); choose A by the first component. For equal win probabilities, compare the second component. |
| Research state (§2) | Two active 100-cost technologies with progress 1 and 99 respectively; both gain 10 science; no boosts or cost changes | First remains incomplete at 11; second completes. Overflow handling is a provider-specific fixture. |
| Late reachable eureka (O4) | Cost 100; progress 60; science 10/turn; no useful alternative research; critical boost arrives after 20 turns | Ordinary completion in 4 turns beats waiting 20. Boost reachability cannot force a pause. |
| Discrete purchase bundle (O2) | Budget 10; independent benefits: A costs 6/saves 12; B and C each cost 5/save 9 | B+C saves 18 versus greedy A's 12. Ratios order search but cannot prove optimality. |
| Overlapping milestones (§5) | Two sequential five-turn projects; release turns 10 and 20; one available city | Completion turns 15 and 25; summing all research and execution as disjoint phases would incorrectly give 30. |
| Population gate (O6) | Standard population-limited districts; no ability exceptions | Capacities at populations 1,3,4,6,7 are 1,1,2,2,3. |
| Support-city value (O3) | A city without a spaceport supplies a builder that reaches a Royal Society project earlier than any alternative | The support city's production may have positive victory-time value; spaceport ownership alone cannot prune it. |

## Required engine probes

- **Purchases and governments:** compare full quotes before/after investing
  production, unit versus building/district queue resolution, GS Merchant
  Republic versus Democracy, patronage versus ordinary purchases, faith
  discounts, rounding, and units' remaining movement after purchase.
- **Tourism:** record engine domestic/visiting and bilateral counters on
  both sides of integer thresholds; inspiration culture increments;
  met-status and source modifiers; tourist transfers involving third
  parties; original player count versus eliminated players; team games
  and elimination. These establish the standalone approximation's domain.
- **Turn ordering:** record an unlock, district placement/completion,
  cache refresh, population change and production modifier change near
  the same boundary. Verify events, stocks and costs against the exact
  build rather than choosing an order from the prose.
- **Movement and timers:** verify builder charges, movement, occupied
  spawn tiles, trader commitments, governor establishment and per-city
  per-turn Royal Society limits.
- **Growth and production:** probe amenity/housing/loyalty boundaries,
  starvation, food consumption, chop carry amounts, empty queues,
  switching, project conversions and purchased unique items.
- **Science victory:** verify project release/completion order and
  rewards, expedition launch/travel timing, parallel laser projects,
  power loss/restoration and terminal checks at the relevant phases.

Every probe records the rules manifest, initial snapshot, action log,
expected/observed result and source. Unsupported mechanics stay explicit
in MODEL.md §9 until those results are available.
