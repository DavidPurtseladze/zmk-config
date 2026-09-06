# Dactyl Cygnus Build Notes — Controller, Wiring, and Firmware

Personal reference doc summarizing everything worked out before starting the firmware setup.

---

## 1. The core problem: two different controllers

The tutorial being followed solders onto a **real Pro Micro** (ATmega32u4 chip). That chip
uses Arduino-style pin names: `A0, A1, A2, A3, 14, 15, 16`, etc.

My board is a **nice!nano v2** (or a compatible clone) — this uses an **nRF52840** chip, which
has totally different pin names: `P0.31, P0.29, P0.02, P1.15, P1.13`, written on silkscreen as
`031, 029, 002, 115, 113`.

**Key takeaway:** These boards are *physically* pin-compatible (same footprint, same shape,
solder in the same spot) but *not* the same chip, so the pin **names** never match the
tutorial's Pro Micro names. Solder to the same **physical location** shown in the tutorial —
never match by label text. When it's time to write firmware config, use my board's actual
`P0.xx` / `P1.xx` names, not the tutorial's Arduino names.

---

## 2. Reset switch (6x6x4.3mm tactile button)

- These buttons have 4 legs, but electrically only **2 nodes** — legs on the same side are
  internally tied together.
- Solder to **two diagonal legs** (one from each side) to guarantee bridging the two separate
  nodes.
- It doesn't matter which specific leg on each side, and it doesn't matter which leg goes to
  RST vs which goes to GND — a reset switch has no polarity or direction. It just needs to
  briefly short RST to GND.
- To verify before soldering: use a multimeter's continuity mode. Two legs that beep together
  (untouched) are the same node — don't pick both of those.

---

## 3. TRRS (half-to-half connector)

- A TRRS jack has 4 physical contacts: **Tip, Ring1, Ring2, Sleeve**. Each one is a separate
  wire straight through the cable (tip-to-tip, ring1-to-ring1, etc.) — unlike the reset switch,
  **these are not interchangeable.**
- Typically carries: **VCC (power), GND, and one or two data lines** (serial data, or
  SDA/SCL if using I2C).
- Two separate decisions:
  1. **Controller-side pin** (which physical pin on my board is VCC/GND/data) — this is
     **fixed by the firmware**, not something I choose freely.
  2. **TRRS contact assignment** (which contact = which signal) — this **is** a free choice,
     but once picked, it must be **identical on both halves**, since the cable just passes
     each contact straight through. If left and right disagree on what "tip" means, the
     halves won't talk to each other correctly, and worst case, power could get shorted into
     a data or ground line.
- Safest approach: copy the tutorial's/firmware's own contact ↔ signal mapping exactly, and
  make sure the controller-side pin used for each signal is translated correctly for my board
  (nRF52840 pin names, not Pro Micro names).
- **Never plug/unplug a TRRS cable while either half is powered** — since VCC rides on one of
  the wires, hot-plugging can damage the controller.

---

## 4. The firmware fork: QMK vs ZMK

This turned out to be the most important issue in the whole build.

- **QMK** has full official support for AVR chips (real Pro Micro). Its support for the
  nRF52840 is **not official** — Nordic's nRF52 SDK has licensing terms that block it from
  being merged into mainline QMK. What exists are old, community side-projects, not something
  to rely on for a real build.
- **ZMK** is the firmware actually built for nRF52840 boards like the nice!nano. It's the
  actively maintained, well-supported option for this hardware.
- Confirmed my nice!nano v2 already effectively "used ZMK" once before — picking `nice_nano_v2`
  during a previous Lily58 build was choosing a ZMK board target, not a QMK one, even if it
  wasn't labeled that way at the time.
- The Dactyl Cygnus designer (juhakaup) is aware of this and publishes an **official ZMK
  config** for wireless/nRF52840 builds of the Cygnus, separate from any QMK keymaps floating
  around (which assume a real Pro Micro).

**Conclusion: build this keyboard with ZMK, targeting board `nice_nano_v2`, using a Cygnus
ZMK shield/config (juhakaup's own, or the community `tokyo2006/zmk-for-cygnus` repo).**

The wiring (matrix, reset switch, TRRS) doesn't change based on this choice — only the
firmware config and build process does.

---

## 5. Wired vs Bluetooth for the two halves

ZMK actually offers a choice here, separate from the central-to-PC connection:

- **Central → computer:** always via USB cable, works the same in ZMK either way.
- **Half → half (over the TRRS cable):**
  - **Bluetooth split** (default/most common) — halves talk over BLE. Mature, well-tested,
    handles two-piece splits great.
  - **Wired split (UART over TRRS)** — a newer, less battle-tested ZMK feature. Works for a
    simple one-central/one-peripheral setup (which the Cygnus is), but most existing Cygnus
    ZMK configs assume Bluetooth split, so wired split would mean hand-editing the devicetree
    config (`compatible = "zmk,wired-split";`) rather than using an off-the-shelf config.
  - A common middle ground: use normal BLE split between halves, but just always keep the
    board plugged into USB for power — gets a "wired-feeling," always-on keyboard without
    touching the experimental wired-split path.

---

## 6. Confirmed variant: 3x5 + 3 thumb (36 keys)

Settled: this Cygnus is wired **3 rows x 5 columns + 3 thumb keys per side** (36 keys total),
not the wider default Cygnus column count. That number matters because it's exactly the same
key count and shape as the **"5 column" alternate layout that ZMK's mainline `corne` shield
already ships** (Corne is normally 3x6+3=42 keys, but it includes a pre-built 5-column/36-key
physical layout + matrix-transform for people who drop the outer pinky column).

That match makes forking a good option here, instead of depending on a third-party repo or
writing a physical layout from scratch:

- **Fork mainline Corne's shield, trimmed to 5 columns** *(chosen approach)* — start from
  ZMK's official, well-tested `corne` shield, reuse its existing 5-column physical layout and
  matrix-transform (`layouts/foostan/corne/5column.dtsi`, already part of ZMK core, no extra
  dependency needed), and rewrite only the pin assignments to match this board's actual
  wiring. Gets a real, working shield fully under our own control.
- juhakaup's official Cygnus ZMK config — only useful if it happens to publish a variant that's
  actually 3x5+3, not just the standard Cygnus column count.
- `tokyo2006/zmk-for-cygnus` — community repo, would need to match this exact variant and stay
  maintained; less control than forking directly.

Important nuance on "just run Corne on it": you can't drop the mainline `corne` shield in
*unmodified* — it assumes 42 keys, this board has 36 — but forking it and trimming it down (as
done here, in `boards/shields/dactyl_cygnus/`) is exactly the shortcut that instinct pointed at.

**Implemented in this repo:** `boards/shields/dactyl_cygnus/` — `dactyl_cygnus.dtsi` (shared
matrix-transform + physical layout + kscan definition), `dactyl_cygnus_left.overlay` /
`_right.overlay` (per-half column wiring), `dactyl_cygnus.keymap` (default layer forked from
Corne's, outer pinky column dropped). **Pin numbers in these files are still placeholders**
copied from Corne's own wiring so the config compiles — they need to be replaced with this
board's actual soldered pins (see each file's `TODO PLACEHOLDER` comments) before flashing.

## 7. One repo or several?

ZMK doesn't require one repo per keyboard — `build.yaml` supports multiple board+shield combos
from a single repo (list each combination under `include:`), and each shield can sit in its
own `boards/shields/<name>/` subfolder.

**Decided: merged into the existing `lily58` repo (`dato12/zmk-config`)** as a second shield,
rather than keeping a separate `dactyl-cygnus` repo. One push builds both keyboards' firmware
in the same GitHub Actions run; updating either board's config only ever means updating this
one repo. Shield hardware definition lives at `boards/shields/dactyl_cygnus/`, user-editable
keymap/conf at `config/dactyl_cygnus*`, same pattern as `lily58`'s own files.

---

## 8. Next steps / what's still open

- [x] Confirm exact Cygnus variant — 3x5 + 3 thumb, 36 keys (§6).
- [x] Decide shield source — fork mainline ZMK `corne`, trimmed to 5 columns (§6).
- [x] Scaffold the ZMK build (`build.yaml`, `.github/workflows/build.yml` — GitHub Actions,
      no local toolchain needed).
- [ ] Get the real wiring diagram (row/col pin assignments, diode direction) and replace the
      placeholder pins in `boards/shields/dactyl_cygnus/`.
- [ ] Decide Bluetooth split vs wired split for the two halves.
- [ ] Flash both halves once wiring is fully soldered and matrix-tested.
