# Canister.MonumentalAI

MonumentalAI models Civilization VI's economy and victory paths for
Gathering Storm, with Science and Culture victories as its planning targets.

- [Model specification](MODEL.md) — state, mechanics, objectives, and solver
  interfaces (v5, 2026-11-13).
- [Sources and evidence status](SOURCES.md) — pinned references and the
  limits of the existing audits.
- [Validation cases](VALIDATION.md) — analytical examples and required
  game-engine calibration probes.

The repository currently contains the specification; a solver and game
adapter have yet to be implemented. There are no installation or runtime
commands. Numerical rules default to Standard speed. Optional-mode rules
apply only when their mode is enabled; unresolved mechanics are tracked
in MODEL.md §9.

Every future run must record the exact game build/data hashes, enabled
content and modes, civilization/leader configuration, speed, victory
settings, horizon, and sampling seed. No executable game-build profile has
been certified by this repository.
