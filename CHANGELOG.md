# Changelog

All notable changes to **rupa** are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.5] - 2026-08-28 — `on-accent`: the ink that goes ON an accent fill

### Added — `on-accent` token (unblocks crab M3 *#32* / the selected-row gate)

Every other rupa token names a colour you paint **with**. `accent` was published without the one
thing a consumer needs to *use* it: the ink legible on top of it. A selected row, a focused tab or a
primary button could be filled but not labelled, so each consumer guessed — and guessed differently.

- **`RU_TH_ON_ACCENT = 112`**, `RU_TH_SZ` **112 → 120**. Appended at the end, so every existing
  offset is unchanged and a consumer reading raw `RU_TH_` offsets still reads the right fields.
- **`rupa_theme_on_accent(t)`** accessor.
- **`rupa_theme_new` takes a 15th parameter**, `on_accent`. ⚠ Signature change — but the only callers
  stack-wide are rupa's own four theme constructors.
- Per-ground values, each chosen against a true sRGB (^2.4) contrast computation:

  | theme | accent | on-accent | contrast |
  |---|---|---|---|
  | mudra-dark | `0x00E5FF` | `0x0B0C0E` | 12.72:1 |
  | mudra-light | `0x0090A6` | `0x17191E` | **4.64:1** (the tightest) |
  | shanta-dark | `0xE9C46A` | `0x14110C` | 11.27:1 |
  | shanta-light | `0xC1963A` | `0x2B2620` | 5.49:1 |

  All four clear the WCAG AA floor for normal text (4.5:1), and the tests assert that floor rather
  than the numbers, so the approximation below cannot become the specification.

### Added — contrast primitives

- **`rupa_luminance(c)`** — relative luminance in permille (0..1000).
- **`rupa_contrast(a, b)`** — contrast ratio ×100, so `450` is the WCAG AA floor. Symmetric by
  construction.
- **`rupa_ink_on(fill, ink_a, ink_b)`** — picks whichever ink reads better on `fill`; ties go to
  `ink_a`.

⚠ **The gamma is squared, not ^2.4.** True sRGB linearization needs a `pow` this leaf will not carry.
The approximation reads slightly high — ~13.68 where the true ratio is 12.72, ~5.20 where it is 4.64 —
so it is for **choosing** an ink and catching a badly wrong pairing, not for quoting compliance.

⛔ **The obvious heuristic is wrong, which is why these exist.** A 299/587/114 luma against a 128
threshold calls MUDRA light's `0x0090A6` accent "dark" and picks the light ink — but the dark ink
measures better (4.64:1 against 3.57:1). Comparing real contrast gets all four grounds right.

### Testing

- 58 assertions (was 42). The four on-accent pairings are asserted **against the AA floor**, and the
  no-token fallback — painting the theme's ordinary `ink` on the accent — is asserted to **FAIL** it
  (1.27:1 and 1.32:1 on the dark grounds), so the guarantee is not vacuous.
- Mutation-proven five ways: mudra-light set to the heuristic's wrong answer (2 failures), `ink_on`'s
  comparison inverted (3), `rupa_contrast` losing its symmetry (2), the red weight dropped from
  luminance (2), and `on_accent` stored at `RU_TH_FONT`'s offset (5).

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
