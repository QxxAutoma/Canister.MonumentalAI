# Sources and evidence status

This register supports the v5 corrections dated 2026-11-13. It separates
direct evidence from inherited claims and the calibration still needed
for a forward simulator. No game build or adapter has been certified.

## Primary documentation and game-data evidence

| Reference | Relevant evidence | Scope and limitation |
|---|---|---|
| [Official Civilization VI manual](https://steamcdn-a.akamaihd.net/steam/apps/289070/manuals/CIV_VI_25TH_ONLINE_MANUAL_ENG.pdf), printed p. 96 | Purchasing a unit creates a separate copy while the unit under construction retains its production timeline. | Supports G6's full-item purchase semantics. Does not establish all GS discounts, purchase eligibility, rounding, or the queue treatment of unique buildings/districts. |
| [Official Antarctic Late Summer Update, April 2, 2019](https://steamcommunity.com/games/289070/announcements/detail/1730979046031331465) ([publisher feed copy](https://store.steampowered.com/news/posts/?appids=289070%2C353370&enddate=1560877460&feed=steam_community_announcements)) | GS Democracy uses a 15% gold-purchase discount; Royal Society contribution changed from 3% to 2% per builder charge. | Historical patch evidence; compare the active build/data manifest before applying it. |
| [Expansion2_RemoveData.xml](https://github.com/AutoDevJapan/civ6-banseiki/blob/f0e993f8dd486620d9b41471456fd106b12c700b/tools/civ6-data-rs/fixtures/game/DLC/Expansion2/Data/Expansion2_RemoveData.xml#L337-L345) | Deletes `DEMOCRACY_PATRONAGE_GOLD_DISCOUNT` and Merchant Republic's gold-purchase modifiers. | Game XML mirrored as repository fixtures, pinned to commit f0e993f; a mirror is not proof of a user's installed build. |
| [Expansion1_Governments.xml in Expansion2](https://github.com/AutoDevJapan/civ6-banseiki/blob/f0e993f8dd486620d9b41471456fd106b12c700b/tools/civ6-data-rs/fixtures/game/DLC/Expansion2/Data/Expansion1_Governments.xml#L277-L307) | Defines Merchant Republic's 15% district-production bonus and Democracy's 15% gold-purchase bonus. | Same pinned mirror. Governor-city gold modifiers appear in the same file at lines 391–401. |
| [In-game district concept text mirror](https://www.civilopedia.net/en-US/standard-rules/concepts/cities_10/) | Population-limited districts unlock at population 1, 4, 7, and every three additional citizens. | Standard-rules concept text supports the basic offset; GS district and civilization exceptions require the active rules provider. |
| [WorldRankings.lua, domestic/total counters and threshold](https://github.com/mrobaczyk/civ6/blob/310319f5a7f42ac65b0979a3e7572129fb17cc69/Base/Assets/UI/PartialScreens/WorldRankings.lua#L1667-L1690), [bilateral counter](https://github.com/mrobaczyk/civ6/blob/310319f5a7f42ac65b0979a3e7572129fb17cc69/Base/Assets/UI/PartialScreens/WorldRankings.lua#L1766-L1768) | Game UI reads `GetStaycationers`, `GetTouristsTo`, and `GetTouristsFrom`; required tourists exceed alive major non-teammate domestic stocks. | Pinned UI mirror at commit 310319f. The core methods' arithmetic, divisor and rounding are not exposed by this Lua. |
| [Official Gathering Storm manual](https://cdn.2kgames.com/manuals/civilization-vi/gathering_storm/2KGMKT_CIV6_Gathering_Storm_PC_Online_Manual_V7_US.pdf), printed p. 11 | Science victory's satellite, Moon, Mars and exoplanet project sequence. | Supports the milestone chain; turn ordering, duration and acceleration still require the active transition provider. |

## Inherited evidence

The G1–G14 entries retain their earlier wiki/page-name provenance where no
new primary-source audit was performed. A label such as “exact” describes
the formula's intended domain; it does not certify the complete transition
or game build. The old audit's raw XML snapshots, game saves, simulation
code, and test logs are not present here. Its user-reported prices are
observations, not universal exchange rates.

Common legacy page references include
[City](https://civilization.fandom.com/wiki/City_(Civ6)),
[District](https://civilization.fandom.com/wiki/District_(Civ6)),
[Builder](https://civilization.fandom.com/wiki/Builder_(Civ6)),
[Production](https://civilization.fandom.com/wiki/Production_(Civ6)),
[Tourism](https://civilization.fandom.com/wiki/Tourism_(Civ6)), and
[Great People](https://civilization.fandom.com/wiki/Great_People_(Civ6)).
These are secondary sources and can combine rulesets or disagree, as G5
already records. Repeated claims on pages within the same wiki are not
independent game measurements.

## Evidence required for a certified profile

Record build number and hashes of the loaded game data, DLC/mod list and
load order, all scenario settings, and the transition provider's version.
Each numerical calibration needs its source/revision, complete starting
snapshot, action sequence, expected and observed account/event deltas,
and seed if randomness applies. Store discrepancies and supported scope
alongside the fixture. [VALIDATION.md](VALIDATION.md) lists the minimum
probes for the corrected specification.
