# Changelog

All notable changes to **rupa** are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.4] - 2026-08-17 — toolchain pin to 6.5.27

### Changed — `cyrius = "6.5.5"` -> **6.5.27**

Stack-wide sweep so every repo in the desktop stack declares one toolchain. Pins had drifted across
three lines (6.5.5 / 6.5.20 / 6.5.21) while the installed wrapper was 6.5.27, so every build ran with
a drift warning and the declared graph did not describe what was actually compiled.

⚠ **THE ARTIFACT CHANGED, so this is not a cosmetic edit.** The build went `76632 -> 80864` bytes and the binary differs. The pin is not a comment: it selects the stdlib snapshot under `~/.cyrius/versions/<pin>/lib`, so moving it swaps the library code this repo compiles against.

⛔ **THIS CORRECTS A HALF-TRUTH IN THE 1.0.4 ENTRY**, which reads *"the pin was documentation, not
enforcement — `cyrius build` compiles with the INSTALLED `cycc`"*. That is true **of the compiler** and
it is not the whole mechanism, and believing it was cost a wrong prediction during this sweep: the
bump was expected to be byte-neutral and was not. `cycc` is indeed the installed binary either way —
but the **stdlib** resolves from `~/.cyrius/versions/<pin>/lib`, so the pin decides which library
sources get compiled in. Measured before any other change: the pin bump ALONE moved these bytes.

⚠ The vendored `lib/` was then re-synced to the 6.5.27 bundled set, clearing the
`./lib/ shadows version-pinned` warning. Tests re-run green after both changes.

## [0.1.3] - 2026-08-17 — the system motion vocabulary, and the guards that make an override safe

### Added — `src/motion.cyr`: roles the desktop grants, overridable per widget, clamped

⭐ **The operator's ruling, implemented literally:** *"compositor grants the motion, apps can override
per-widget, but lets guard against any impossible or highly destructive behaviors."*
Four roles — `INSTANT` / `QUICK` / `CALM` / `BUSY` — plus three easings. ⚠ **Roles, not durations:** an
app asking for "220 ms ease-out" has hardcoded a look; one asking for `CALM` inherits whatever the
desktop decides CALM means and moves with everything else when that changes. Exactly why the colour
tokens are named `accent`/`alert` and not hex.

⚠ **IT LIVES HERE FOR THE SAME REASON THE COLOUR TOKENS DO.** The compositor's chrome, the dhancha
toolkit and apps must move as ONE system. A vocabulary owned by the compositor could not be read by a
client-side toolkit (a toolkit cannot depend on the compositor); one owned by dhancha would leave the
compositor's own chrome inventing its own.

⛔ **This file animates nothing.** Roles, override resolution, and "how far through at time t" as pure
arithmetic. Drawing belongs to whoever owns pixels — which is what makes every guard testable on a host
with no screen and no clock.

### The guards — the substance, not the trim

⛔⛔ **THE FLASH BAND.** Periodic visual change between roughly **3 Hz and 55 Hz** is the
photosensitive-epilepsy trigger band, and a widget "pulsing to show it is busy" at 10 Hz sits inside it.
Any periodic role is floored at `RU_MO_PERIOD_MIN_MS = 340` (~2.94 Hz, below the 3 Hz floor) and **no
override may go under it**. `rupa_motion_cycle` re-floors independently, because a caller can reach it
without passing through the duration clamp and the guard must not depend on which door was used.

⛔⛔ **REDUCED MOTION OUTRANKS THE APP.** A per-widget override is a choice WITHIN the envelope; "the
operator cannot tolerate motion" IS the envelope. Vestibular disorders make this a harm question, not a
preference. ⚠ The clamp order is load-bearing — reduced is applied LAST, so an override that happens to
land inside the envelope cannot defeat it.
⭐ **But BUSY is never silenced**, only slowed: removing the sole indication that the system is working
replaces motion with a **lie about system state**, leaving the operator unable to tell working from hung.

⚠ Impossible values answer rather than divide: a zero or negative duration is FINISHED (255), not a
divide-by-zero; negative elapsed is the start; an unknown role grants nothing; an unknown easing falls
back to the grant rather than being clamped into a neighbouring curve. A zero duration is not "fast",
it is ABSENT — a control with no acknowledgement reads as dead — so it is floored; a stall-length one is
capped, because that bound is on what the operator believes about the machine.

**Verification** — 39 assertions, and mutation-tested on both harm guards: removing the flash-band floor
fails 3, and letting an app defeat reduced-motion fails 3.

## [0.1.2] - 2026-08-02

### Changed — cyrius pin 6.4.71 -> 6.5.5

Toolchain catch-up across the whole desktop stack, cut together so the next burn runs binaries built
by ONE compiler rather than 6 different ones.

⚠ **The pin was documentation, not enforcement.** `cyrius build` compiles with the INSTALLED `cycc`,
prints a `toolchain drift` warning, and carries on — so this project was already being built by 6.5.5
before this bump. Verify provenance with `~/.cyrius/versions/<pin>/bin/cyrius` when it matters.

⭐ What the gap actually contained, for a reader deciding whether to care:
- **6.5.1** made overload-suffix arity a hard **error** where it used to warn. Latent arity
  mismatches are now build failures instead of silently-wrong code — good, and the reason this
  sweep surfaced real defects elsewhere in the stack.
- **6.4.75** fixed `fn_table` growth past 8192 silently corrupting six fn-indexed side tables.
- **6.5.0** added file-scoped `private` / per-item `public` — the first real answer to this
  ecosystem's duplicate-`fn`-silently-shadows hazard.
- **6.4.82** completed the agnos GPU syscall wrapper band to `#82`-`#95`, so `sys_gpu_shader_op`
  (#92) and `sys_gpu_modeset_op` (#93) no longer need a raw `syscall()` behind an `#ifdef`.

### Verification

Host + `--agnos` builds green; 1 suite passes; `distlib` regenerated.

## [0.1.1] - 2026-07-23

### Changed — cyrius pin 6.4.61 → 6.4.71

Toolchain refresh across the draw stack. Materialised `lib/` re-synced (`cyrius lib sync --full`).
No source change; build + tests green at the new pin.

## [0.1.0] - 2026-07-12 — the desktop theme-token core

Extracted from `aethersafha/src/theme.cyr` when the theme gained its second consumer
(dhancha), so a single sovereign source feeds the compositor chrome, the widget toolkit,
and apps — a toolkit/app cannot depend on the compositor, so the tokens move down to a
shared leaf.

### Added

- **`RupaTheme` token model** — a 14-field theme (colors as logical `0xRRGGBB`):
  `bg / panel / widget / line / ink / mute / faint / accent / alert / held` plus `radius`
  and `font` (permille). `rupa_theme_new` + full accessors.
- **The four grounds** — `rupa_theme_mudra_dark` / `_mudra_light` / `_shanta_dark` /
  `_shanta_light`, the exact hexes from the consolidated design (MUDRA the seal · SHANTA
  stillness, each dark + light).
- **Packer-agnostic channel helpers** — `rupa_color_r/g/b` (rupa carries no framebuffer
  packer, so it stays a pure leaf; each consumer packs to its own format).
- **Registry + active theme** — `rupa_theme_by_name` (self-contained `rupa_streq`) for
  config-driven selection, and the system's single active theme (`rupa_theme_active`
  lazy-defaults to MUDRA · Carbon; `rupa_theme_set_active` / `_set_active_name`).
- **`tests/theme.tcyr`** — 39 assertions locking every ground's tokens, the channel
  helpers, the registry, and the active selector.

Design doc: `agnosticos/docs/development/designs/desktop_consolidated/theme-system.html`.
