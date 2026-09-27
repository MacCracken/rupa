# Changelog

All notable changes to **rupa** are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.8] - 2026-09-26 — toolchain 6.6.6, and four CI gates that could not fail

### Changed — toolchain `6.6.2` → `6.6.6`

- **No source change for the toolchain.** Both bundled modules and both suites are untouched; the
  edits to `src/lib.cyr` and `programs/smoke.cyr` are the gate fixes below, and this section's
  numbers are measured without them. **114 assertions** (42 motion + 72 theme) pass unchanged;
  `lint` 0 warnings, `vet` clean, `distlib --check` current, and **0 of 6** files move under the
  6.6.6 formatter — measured by formatting in place and diffing, not by `fmt --check` alone.
- **Zero compiler rejections and zero fail-open sites.** No build, suite, `vet` or `distlib` run
  prints a warning or error. The range's fail-open classes have no site here: a pre-6.6.4
  `cyrius.lock` failing open for one resolve (rupa has no lock), a `private` impl method's forward
  call (no `private`, no `impl`), and `cyrlint --exit-with-count` over several files dropping a read
  error (CI lints one file per call, without the flag). The four `callptr` sites — the allocator
  vtable's `alloc_via` / `realloc_via` / `free_via` / `reset_via` in `lib/alloc.cyr`, byte-identical
  to 6.6.2's — still reach no pair-returning target: every function any `allocator_new` in the
  closure installs is declared `: i64`. ⚠ The closure DOES carry one pair-returning type — `Str`,
  16 B, `lib/str.cyr`, unchanged — and it passes 6.6.6's new 9–16 B return-shape check without a
  diagnostic. rupa's own code declares no struct and makes no indirect call.
- ⚠ **The artifact changed**, and would have with no stdlib change at all: 6.6.5 pads every call
  emitted at odd expression-stack depth to keep `rsp` 16-byte aligned, so no binary is
  byte-comparable across it by construction (rupa calls no C or SSE code, so the misalignment it
  fixes never bit here). Smoke **80,904 → 85,192 B** plain, **19,464 → 19,656 B** under
  `CYRIUS_DCE=1`.
- ⭐ **Both cross-target builds are now diagnostic-free.** `--agnos` no longer warns
  `undefined function '_agnos_getenv'`: 6.6.6's `lib/io.cyr` includes `args_agnos.cyr` directly,
  where 6.6.2 forward-referenced it and `getenv` trapped unless something else pulled the peer in.
  `--aarch64` no longer flags `raw syscall 32` in `lib/io.cyr`: 6.6.4 moved `xflock` onto
  `SYS_FLOCK` (the x86 number, renumbered by ESYSXLAT), ending a false positive on correct code.
  rupa calls neither `getenv` nor `flock`; both were stdlib noise in every cross build. Plain
  sizes: agnos 84,976 → 89,144 B; aarch64 199,648 → 265,336 B, 86 % of that growth unreachable
  stdlib (166,972 → 223,396 B) — `CYRIUS_DCE=1` does not shrink aarch64 output at either pin.

### Changed — `lib/` re-resolved from the 6.6.6 stdlib

- `rm -rf lib && cyrius deps`, as CI does: **21 → 23 files**, each byte-equal to
  `~/.cyrius/versions/6.6.6/lib/`. Joining: `alloc_cx.cyr` (now included by `alloc.cyr`) and
  `args_agnos.cyr` (by `io.cyr`, agnos target only). 14 of the 21 carried files changed; `alloc_agnos`,
  `alloc_macos`, `alloc_windows`, `args_macos`, `atomic`, `result` and `str` did not.
- ⚠ **`lib/` is not what gets compiled.** MEASURED at both pins: with `lib/fmt.cyr` deliberately
  corrupted, `build`, `test`, `lint`, `vet`, `check` and `distlib --check` all succeed and the smoke
  binary is byte-identical (85,192 B at 6.6.6, 80,904 B at 6.6.2). For this stdlib-only chain the
  stdlib comes from `~/.cyrius/versions/<pin>/lib` — **the pin is the update**; re-resolving `lib/`
  keeps the local mirror truthful. No `cyrius.lock` is written for a stdlib-only manifest.
- `[deps].stdlib` unchanged: no target needs a new module at 6.6.6.

### Unchanged — `dist/`

- `dist/rupa.cyr` differs only in its `# Version:` stamp (433 lines, 23,169 B), and
  `dist/rupa.deps` is byte-identical — the same **8** leaves. ⚠ 6.6.6 made `lib/string.cyr` include
  `alloc` itself, which is why rekha's sidecar dropped `alloc`; rupa's `src/lib.cyr` includes `alloc`
  directly and the sidecar is the declared list plus an include scan, so a consumer resolves exactly
  what it did at 0.1.7. **No new toolchain floor**: the bundle's code is 0.1.7's.

### Fixed — four CI gates that passed the defects they exist to catch

Found while auditing the bump, and each fix PROVEN rather than argued: the defect was planted in a
copy of the tree, and each version's own `ci.yml`, include chain and smoke were run — the steps'
exact `run:` scripts under GitHub's `bash -e`, both at 6.6.6.

| planted defect | 0.1.7 CI | 0.1.8 CI |
|---|---|---|
| an untracked `TODO` | **passes** | fails at Lint |
| a file cyrlint fails on (unreadable: exit 1) | **passes** | fails at Lint |
| `dist/rupa.deps` missing its `fmt` leaf, committed | **passes** | fails at dist |
| `dist/rupa.deps` deleted, committed | **passes** | fails at dist |
| a syntax error in `src/motion.cyr` | **passes** the smoke build | fails at Build |
| the flash-band floor removed from `rupa_motion_duration` | **passes** the smoke steps | fails at Run smoke |
| mudra-dark's accent `0x00E5FF` → `0x00E5FE` | **passes** the smoke steps | fails at Run smoke |

The controls did not move: a lint warning, a stale bundle and a bundle that does not compile fail
under both; a `TODO` cross-referenced to a CHANGELOG entry passes under both; the clean tree passes
every build-job step under both. ⚠ The last two rows were never unguarded VALUES — the suites fail
both under either CI. What was false was the smoke's claim to run.

- **Lint.** ⛔ The step promised *"any `warn` line (warning or untracked-deferral) fails CI"*. cyrlint
  prints an untracked deferral as `deferral line N:`, which the `warn` grep never matched, and plain
  `cyrius lint` exits 0 either way; `|| true` also discarded the only sign of a file it cannot read
  (exit 1, no `warn` line). Now `--strict-deferrals` (exit 2 on an untracked deferral; forwarded
  since 6.6.5), a `deferral` grep, and the exit code. ⚠ Three signals, because `--strict-deferrals`
  still exits 0 on a warning.
- **dist.** The step diffed `dist/rupa.cyr` only — the sidecar consumers resolve their stdlib from was
  never compared, and a deleted file has no diff. Now `cyrius distlib --check` (bundle and sidecar,
  byte-compared, nothing written), then plain `cyrius distlib` — setu 0.8.11's shape. ⛔ MEASURED why
  the second line stays: on a bundle that does not compile, committed as distlib wrote it, `--check`
  exits **0** and only plain distlib's self-compile exits 1. The old step had that coverage by
  accident; `--check` alone would have dropped it.
- **The smoke never compiled `src/motion.cyr`.** Bundled since 0.1.3, but `src/lib.cyr` — the chain the
  smoke builds — never included it, so only its own suite and distlib's self-compile compiled it. It
  is included now, and the smoke checks three contracts the suite already pins: CALM is granted
  260 ms, a 10 Hz BUSY override floors to `RU_MO_PERIOD_MIN_MS`, a zero duration answers FINISHED.
  Smoke 85,192 → **89,392 B** plain, 19,656 → **19,760 B** under `CYRIUS_DCE=1`; both cross builds stay
  diagnostic-free. ⚠ `vet` is unaffected either way: it lists only the scanned program's direct
  include (`OK src/lib.cyr`, 1 dep), before and after.
- **CI built the smoke and never ran it.** It is not a `programs/*_test.cyr`, so none of its
  `return 1` checks, theme or motion, ever executed in CI. A `Run smoke` step runs it.
- The format step's comment said the file must come before `--check`; flag position stopped mattering
  at 6.6.5. It now also records that `--check` accepts a 2- or 4-space continuation indent where
  `fmt` writes 2 — the leniency rupa itself asked for at 0.1.3 (cyrius v6.5.28) — so passing `--check`
  is not the same as being canonical. All six files are canonical (measured, above).

### Fixed — the README's version line

- `README.md`'s `Version:` line had read `0.1.2` since 0.1.2; it now reads `0.1.8`.

## [0.1.7] - 2026-09-11

### Changed

- **Toolchain `6.5.36` → `6.6.2`.** No source change. Zero compiler rejections and
  zero fail-open sites; the four `callptr` sites reach no pair-returning target.
  **114 assertions** pass unchanged.

## [0.1.6] - 2026-08-31 — the scrim token

### Added — `rupa_theme_scrim` / `rupa_theme_scrim_a`

⛔⛔ **A SCRIM IS A PALETTE DECISION, NOT A CONSUMER'S.** dhancha needed a modal backdrop for its
0.9.23 sheet and had exactly two bad options: name a colour itself, or fake the dim with a scanline
dither — and it shipped the dither. Neither is a decision a widget toolkit gets to make: **how far
the page recedes behind a dialogue is a palette question**, and it differs between a carbon void and
rice paper. crab's ADR 0001 forbids the app from naming colours; the same argument reaches one layer
further up.

⚠ **TWO FIELDS, NOT ONE.** A scrim is a colour **and** a coverage. Packing the alpha into the colour
would collide with sadish's legacy `sd_alpha_of` rule, where a 0 byte means **opaque**.

| theme | scrim | alpha |
|---|---|---:|
| MUDRA dark | the carbon void `0x0B0C0E` | 178 (70 %) |
| SHANTA dark | the warm void `0x14110C` | 178 (70 %) |
| MUDRA light | its **ink** `0x17191E` | 102 (40 %) |
| SHANTA light | its **ink** `0x2B2620` | 102 (40 %) |

⛔ **The light grounds dim with their INK and at far lower alpha.** The void at 70 % over rice paper
would black the page out rather than let it recede — and that asymmetry is the token's entire
justification. The suite asserts `dark_alpha > light_alpha` in both families, so if they ever
equalise one of the two is wrong.

⚠ **Appended at the end of the record**, for the same reason `on_accent` was at 0.1.5: every offset
above is unchanged, so a consumer reading raw `RU_TH_` offsets against an older build still reads the
right fields. Only `RU_TH_SZ` grows (120 → 136), and rupa owns the allocation.

### Changed — toolchain pin 6.5.35 → 6.5.36

### Verified

**58 → 72** assertions, 0 failed.

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
