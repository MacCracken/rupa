# rupa

**रूप — form / appearance**

Version: 0.1.0

The sovereign **desktop theme-token core** for AGNOS, written in pure Cyrius. rupa is
the ONE source of the desktop's visual identity: the compositor chrome
([aethersafha](https://github.com/MacCracken/aethersafha)), the widget toolkit
([dhancha](https://github.com/MacCracken/dhancha)), and apps
([crab](https://github.com/MacCracken/crab),
[jalwa](https://github.com/MacCracken/jalwa)) all read the same tokens, so the whole
desktop moves as one.

## Two temperaments, one philosophy

Six divergent aesthetic explorations were consolidated to **two themes**, unified by the
question every one of them was really answering — *how do you make trust visible?*

- **MUDRA** (मुद्रा — *the seal*) — the sovereign **default**: signed, radius-0, the cyan
  verify-seal. Trust reads **loud**.
- **SHANTA** (शान्त — *stillness*) — the **focus** mode: warm, thin, one gold firefly +
  sage. Trust reads **quiet** (luminance).

Each ships in **dark and light** — four grounds from one token set.

## Tokens

A `RupaTheme` carries logical **`0xRRGGBB`** colors (rupa is packer-agnostic — each
consumer packs to its own framebuffer format via `rupa_color_r/g/b`):

| Slot | Role |
|------|------|
| `bg` | desktop void / root background |
| `panel` | panel / titlebar / card surface |
| `widget` | raised widget / window body / input surface |
| `line` | hairline / divider / border |
| `ink` / `mute` / `faint` | primary / secondary / tertiary text |
| `accent` | verify seal / focus / firefly — the one bold hue |
| `alert` | unsigned / destructive / error |
| `held` | pending / sandbox / held |
| `radius` / `font` | corner radius px · font scale (permille) |

## API

```cyrius
var t = rupa_theme_mudra_dark();          # or _mudra_light / _shanta_dark / _shanta_light
var cyan = rupa_theme_accent(t);          # 0x00E5FF
var r = rupa_color_r(cyan);               # -> pack with your own bhumi_xrgb / sd_rgb / ...

rupa_theme_set_active_name("shanta-dark");  # the system's single active theme
var active = rupa_theme_active();           # lazy-defaults to mudra-dark
```

## Build

```sh
cyrius deps                               # resolve stdlib into lib/
cyrius build programs/smoke.cyr build/rupa-smoke
cyrius test                               # tests/*.tcyr
```

rupa is a **leaf** — no upstream deps beyond the Cyrius stdlib. Consumers pull
`dist/rupa.cyr` via a `[deps.rupa]` git-tag entry.

## License

GPL-3.0-only.
