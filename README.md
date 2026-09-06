# zmk-config

Personal ZMK user config, building firmware for two split keyboards from one repo. Every push
to GitHub builds all four `.uf2` files (both halves of both boards) in one Actions run — grab
whichever one you need from that run's Artifacts.

| Keyboard | Shield | Layout | Status |
|---|---|---|---|
| Lily58 | `lily58_left` / `lily58_right` (mainline ZMK) | 4x6 + thumbs | Built and in use |
| Dactyl Cygnus | `dactyl_cygnus_left` / `dactyl_cygnus_right` (vendored in this repo, forked from mainline Corne) | 3x5 + 3 thumb (36 keys) | **Not flashable yet — wiring pins are placeholders, see below** |

Both target board `nice_nano_v2` (nRF52840).

## Repo layout

- `config/` — per-keyboard keymap (`*.keymap`) and Kconfig overrides (`*.conf`). Edit these to
  change what a keyboard actually does; they take priority over any default bundled in
  `boards/shields/`.
- `boards/shields/dactyl_cygnus/` — the Cygnus's hardware definition (matrix wiring, physical
  layout, Kconfig plumbing). Only needed because the Cygnus isn't in mainline ZMK; Lily58 needs
  no equivalent folder since ZMK already ships that shield.
- `build.yaml` — the board+shield matrix GitHub Actions builds.
- `docs/dactyl-cygnus-build-notes.md` — full write-up of the Cygnus build: the Pro
  Micro/nice!nano controller mix-up, reset switch wiring, TRRS contact assignment, why ZMK over
  QMK, and the Corne-shield-forking approach used for the shield in this repo.

## Building

Push to GitHub — `.github/workflows/build.yml` runs ZMK's official build-user-config action
against `build.yaml` and uploads the four `.uf2` files as workflow artifacts. No local
toolchain needed.

## Status / what's left

- **Lily58**: working, builds and flashes normally.
- **Dactyl Cygnus**: shield compiles, but `boards/shields/dactyl_cygnus/dactyl_cygnus.dtsi`
  (row pins) and `dactyl_cygnus_left.overlay` / `_right.overlay` (column pins) still hold
  placeholder GPIO numbers copied from Corne's own wiring, marked `TODO PLACEHOLDER`. Don't
  flash it yet — key positions won't match the real switches until those are replaced with
  this board's actual soldered pins (and diode direction is confirmed). See
  `docs/dactyl-cygnus-build-notes.md` for the full context.
