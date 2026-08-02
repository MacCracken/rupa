# Changelog

All notable changes to **rupa** are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
