# @move-elevator/website-base-scss

A shared SCSS foundation for website projects. It holds the parts that are otherwise
copy-pasted between projects — unit/typography **tools**, structural **tokens**, and
element **base** styles — behind one versioned package that each project consumes.

What it deliberately does **not** ship are brand values. Palette, type, spacing, rhythm
and elevation are set by the project's designer and change with every brand, so there is
nothing to reuse. Instead the package defines the *contract*: the base styles read
`--color-*`, `--font-*` and `--line-height-*` custom properties with a safe fallback, and
the project declares the values.

Because it is Sass, the package is a **superset**: projects import only the parts they
need. Functions, mixins and variables emit **no CSS** until used, so importing the full
toolbox costs nothing. Only the opt-in `base` entry emits rules.

## Installation

```bash
npm i -D @move-elevator/website-base-scss
```

Optional peer dependency (only if you use the `media()` mixin):

```bash
npm i -D include-media
```

Compile with Dart Sass, resolving the package from `node_modules`:

```bash
sass --load-path=./node_modules src/entry.scss:dist/style.css
```

## Usage

Declare the layer order, configure the structural tokens, declare the project's brand
values, then pull in the base styles:

```scss
@use "@move-elevator/website-base-scss/src/layers";       // declare @layer order first
@use "@move-elevator/website-base-scss/src/tokens" with ( // overrides at import time
  $breakpoints: (phone: 320px, tablet: 768px, desktop: 1248px)
);
@use "@move-elevator/website-base-scss/src/base";         // element resets (opt-in)

@layer tokens {
  :root {
    --color-text: #2d3246;
    --color-background: #ffffff;
    --color-headline: #000000;
    --color-border: #d0d0d0;
    --font-family-text: "Barlow", sans-serif;
    --font-family-headline: "Barlow", sans-serif;
  }
}
```

Use the tools anywhere:

```scss
@use "@move-elevator/website-base-scss/src/tools" as *;

.teaser {
  font-size: fluid-clamp(16px, 19px);
}
```

Media queries (needs the `include-media` peer):

```scss
@use "@move-elevator/website-base-scss/src/tools/media" as *;

.grid { @include media(">=tablet") { grid-template-columns: repeat(3, 1fr); } }
```

A runnable version of all of this is in [`example/`](example).

## Entry points

| Import | Emits CSS | Purpose |
| --- | --- | --- |
| `@move-elevator/website-base-scss/src` | `@layer` order only | Default: layer order + all tools + all tokens (zero output). |
| `.../src/layers` | `@layer` order only | The cascade layer order on its own. |
| `.../src/tools` | no | Functions & mixins (`fluid-clamp`, `vw`, `font-face`, `visually-hidden`, `icon`, `space`). |
| `.../src/tools/media` | no | `media()` re-exported from include-media, pre-wired to the token breakpoints. |
| `.../src/tokens` | no | Breakpoints, the font-weight scale and the `z()` lookup. |
| `.../src/base` | yes | Element resets & base typography, each in a cascade layer. |
| `.../src/base/<name>` | yes | A single base partial (e.g. `base/button`) when you want only part of it. |

## Cascade layers

The package declares its layer order once:

```css
@layer reset, tokens, base, layout, components, utilities, overrides;
```

Everything it emits lands in one of these layers. Put your project styles in
`@layer overrides` (or leave them unlayered) and they win over the base **regardless of
selector specificity or source order** — no `!important`, no specificity battles. Load
`layers` (or the default entry, which forwards it) before anything that emits CSS.

## The custom property contract

The base styles never hard-code a brand value. They read custom properties with a
fallback, so the package works standalone and the project overrides what it needs by
declaring these on `:root`:

| Custom property | Fallback | Used by |
| --- | --- | --- |
| `--color-background` | `Canvas` | `html` |
| `--color-text` | `CanvasText` | `html` |
| `--color-headline` | `inherit` | `h1`–`h6`, `.h1`–`.h6` |
| `--color-border` | `currentColor` | `hr` |
| `--color-selection-background` | `Highlight` | `::selection` |
| `--color-selection-text` | `HighlightText` | `::selection` |
| `--font-family-text` | `sans-serif` | `body` |
| `--font-family-headline` | `inherit` | `h1`–`h6`, `.h1`–`.h6` |
| `--font-weight-text` | `$font-weight-regular` | `body` |
| `--font-weight-headline` | `$font-weight-semi-bold` | `h1`–`h6`, `.h1`–`.h6` |
| `--line-height-text` | `1.5` | `body` |
| `--line-height-headline` | `1.1` | `h1`–`h6`, `.h1`–`.h6` |
| `--margin-block-headline` | `0.75em 0` | `h1`–`h6`, `.h1`–`.h6` |
| `--focus-size` / `--focus-style` / `--focus-color` / `--focus-offset` | `3px` / `dashed` / `CanvasText` / `3px` | `:focus-visible` |

The system color fallbacks (`Canvas`, `CanvasText`, `Highlight`) keep the base usable in
forced-colors mode before any palette is declared.

## Tools

| Tool | Signature | Configurable |
| --- | --- | --- |
| `fluid-clamp()` | `fluid-clamp($min-size, $max-size, $min-breakpoint: 320px, $max-breakpoint: 1920px, $unit: vw)` | `$fluid-clamp-baseline: 16px` |
| `vw()` | `vw($pixels, $base-vw: $layout-vw)` | `$layout-vw: 1440px` |
| `font-face()` | `@include font-face($font-name, $file-name, $weight: 400, $style: normal, $formats: woff2)` — `font-display: swap`, `local()` fallback | `$font-path: "../Fonts/"`, `$font-format-hints` |
| `icon-mask()` / `icon-background()` | `@include icon-mask($identifier)` — mask (tintable) or background SVG | `$icon-path: "../Icons/"` |
| `visually-hidden()` / `visually-hidden-focusable()` | `@include visually-hidden` — hide visually, keep it for assistive tech | — |
| `space-headline()` / `space-list()` / `space-text()` | `@include space-headline { … }` — adjacent-sibling spacing hooks | — |
| `z()` | `z("nav")` — named z-index lookup (from `tokens`) | `$z-index` map |
| `media()` | `@include media(">=tablet") { … }` — from `tools/media` | `$breakpoints` map |

`font-face()` emits woff2 only. Pass `$formats` a list to add further formats, in the
order browsers should prefer them — the mixin maps each file extension to its CSS
`format()` hint (`ttf` → `truetype`, `otf` → `opentype`) and errors on an unknown one:

```scss
@include font-face("Barlow", "barlow-v12-latin-regular");                    // woff2
@include font-face("Barlow", "barlow-v12-latin-600", 600);
@include font-face("Barlow", "barlow-v12-latin-italic", 400, italic, (woff2, woff));
```

Tool-level knobs are configured on the tool itself, before the `tools` barrel loads the
same module unconfigured:

```scss
@use "@move-elevator/website-base-scss/src/tools/icon" with ($icon-path: "../Icons/");
@use "@move-elevator/website-base-scss/src/tools" as *;
```

## Configuration

The remaining tokens are structural, not brand-specific. Every one is a `!default`
variable, so `@use "…/tokens" with (…)` wins:

- **Breakpoints** — `$breakpoints` (`phone`, `phone-wide`, `tablet`, `tablet-wide`,
  `desktop`, `desktop-wide`, `full-hd`); also feeds `media()`.
- **Font weights** — `$font-weight-light`, `$font-weight-regular`, `$font-weight-medium`,
  `$font-weight-semi-bold`, `$font-weight-bold`.
- **Layering** — the `$z-index` map, read through `z()`.

Configuration must appear on the **first** `@use` of `tokens` in the compilation (before
`base` loads it), so keep it at the top of your entry sheet.

## Roadmap

- **v2** — shared components (`Grid`, `Section`, `Button`, `Link`, `List`, `Form`) in
  `@layer components`, plus vendor overrides (Splide, Leaflet, Plyr, Flatpickr, Lightbox).
- **v3** — TYPO3 backend styles (RTE, backend cosmetics, login).
- **Later** — theming helpers (`light-dark()`, `prefers-color-scheme`) and a `container()`
  mixin; migration of the Gulp-based projects.

## Development

The repo ships a [DDEV](https://ddev.com/) config, so the toolchain runs in the
container (Node 24) and nothing needs to be installed on the host. `ddev start` also
installs the npm dependencies via its post-start hook:

```bash
ddev start
ddev npm run test:build     # compile every example to example/.dist/
ddev npm run lint:style     # stylelint the package sources
```

`test:build` compiles the whole `example/` directory, so adding a file there — say
`example/font-face.scss` — is enough to have it covered; no script to touch. Files
prefixed with `_` are treated as Sass partials and skipped.

Released automatically with semantic-release. Pairs with
[`@move-elevator/stylelint-config-scss`](https://github.com/move-elevator/stylelint-config-scss).
