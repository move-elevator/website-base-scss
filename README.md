# @move-elevator/website-base-scss

A shared SCSS foundation for website projects. It holds the parts that are otherwise
copy-pasted between projects — unit/typography **tools**, structural **tokens**, and
element **base** styles — behind one versioned package that each project consumes.

What it deliberately does **not** ship are brand values. Palette, type, spacing, rhythm
and elevation are set by the project's designer and change with every brand, so there is
nothing to reuse. The package defines the *contract* instead: the base styles read
`--color-*`, `--font-*` and `--line-height-*` custom properties with a safe fallback, and
the project declares the values. The only colors in the package are `$black` and `$white`
— neutral primitives used as last-resort fallbacks.

Because it is Sass, the package is a **superset**: projects import only the parts they
need. Functions, mixins and variables emit **no CSS** until used, so importing the full
toolbox costs nothing. Only the opt-in `base` and `utility` entries emit rules.

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
@use "@move-elevator/website-base-scss/src/base";         // element styles (opt-in)
@use "@move-elevator/website-base-scss/src/utility";      // utility classes (opt-in)

@layer tokens {
  :root {
    --color-text: #2d3246;
    --color-background: #ffffff;
    --color-border: #d0d0d0;
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

Each tool has a runnable sheet in [`example/`](example):

| File | Shows |
| --- | --- |
| [`fluid-clamp.scss`](example/fluid-clamp.scss) | `fluid-clamp()` with the default and a custom breakpoint range, plus `px-to-rem()`. |
| [`vw.scss`](example/vw.scss) | `vw()` against the default 1440px design width, an explicit one, and shorthand lists. |
| [`font-face.scss`](example/font-face.scss) | `font-face()` woff2-only, a weight variant, and a multi-format call. |
| [`icon.scss`](example/icon.scss) | `icon-mask()` vs `icon-background()`. |
| [`visually-hidden.scss`](example/visually-hidden.scss) | The mixins on project selectors, including the focusable skip-link variant. |
| [`z-index.scss`](example/z-index.scss) | Every `z()` step of the scale. |

`media()` has no sheet because it needs the optional `include-media` peer, which this
repo does not install.

## Entry points

| Import | Emits CSS | Purpose |
| --- | --- | --- |
| `@move-elevator/website-base-scss/src` | `@layer` order only | Default: layer order + all tools + all tokens (zero output). |
| `.../src/layers` | `@layer` order only | The cascade layer order on its own. |
| `.../src/tools` | no | Functions & mixins (`fluid-clamp`, `font-face`, `icon`, `visually-hidden`, `vw`). |
| `.../src/tools/media` | no | `media()` re-exported from include-media, pre-wired to the token breakpoints. |
| `.../src/tokens` | no | Breakpoints, `$black`/`$white`, the font-weight scale and the `z()` lookup. |
| `.../src/base` | yes | Element styles, all in `@layer base`. |
| `.../src/base/<name>` | yes | A single base partial (e.g. `base/button`). Load `layers` yourself — only the barrel forwards it. |
| `.../src/utility` | yes | Utility classes, in `@layer utilities`. |
| `.../src/utility/<name>` | yes | A single utility partial, same caveat about `layers`. |

`base` forwards `html`, `button`, `figure`, `focus`, `hidden`, `hr`, `iframe`, `img`,
`p`, `selection`, `strong` and `video`. **`body` and `headline` are deliberately not in
that set** — they style typography, which is brand territory. Opt into them explicitly
when a project wants them:

```scss
@use "@move-elevator/website-base-scss/src/base/body";
@use "@move-elevator/website-base-scss/src/base/headline";
```

## Cascade layers

The package declares its layer order once:

```css
@layer tokens, base, components, utilities, overrides;
```

The package only emits into `base` (element styles) and `utilities` (utility classes);
the rest is reserved for the project. Put your styles in `@layer overrides` (or leave
them unlayered) and they win over the base **regardless of selector specificity or
source order** — no `!important`, no specificity battles.

Custom properties are the exception: declare the palette in `@layer tokens`, not
unlayered. Unlayered declarations beat every layer, so an unlayered `:root` palette would
defeat a component that re-scopes a variable (`.card--inverted { --color-brand: … }`)
despite its higher specificity. `tokens` comes first precisely so those declarations stay
the weakest and anything later can re-scope them.

A layer the package never declares — say a project-specific `layout` — is appended to the
**end** of the order when first used, so it would outrank `overrides`. To slot one in,
declare the full order yourself in a local partial and `@use` it before the package,
since Sass requires `@use` rules to come first:

```scss
// src/_layers.scss (in your project)
@layer tokens, base, layout, components, utilities, overrides;
```

```scss
// src/entry.scss
@use "layers";                                     // your order wins, it is emitted first
@use "@move-elevator/website-base-scss/src/base";
```

### The two `!important` rules

`[hidden] { display: none !important }` (in `base`) and `.visually-hidden` (in
`utilities`) are the only `!important` declarations in the package, and both are
deliberate. For important declarations the cascade **reverses** layer order — earlier
layers win, and unlayered important declarations are weakest of all:

```
normal:     tokens → base → components → utilities → overrides → unlayered   (strongest)
important:  unlayered → overrides → utilities → components → base → tokens   (strongest)
```

So `[hidden]` outranks any `display` a later layer sets, which is what keeps the
attribute working once components start setting `display: flex`. The flip side is that
neither rule can be overridden from `overrides` — not even with `!important`. That is the
intended trade for an accessibility guarantee; use `.visually-hidden-focusable`, or don't
apply the class.

## The custom property contract

The base styles never hard-code a brand value. They read custom properties with a
fallback, so the package works standalone and the project overrides what it needs by
declaring these on `:root`:

| Custom property | Fallback | Used by |
| --- | --- | --- |
| `--color-background` | `$white` | `html` |
| `--color-text` | `$black` | `html` |
| `--color-border` | `$black` | `hr` |
| `--color-selection-background` | `$black` | `::selection` |
| `--color-selection-text` | `$white` | `::selection` |
| `--size-focus` / `--style-focus` / `--color-focus` / `--offset-focus` | `3px` / `dashed` / `$black` / `3px` | `:focus-visible` |

The opt-in typography partials add their own:

| Custom property | Fallback | Used by |
| --- | --- | --- |
| `--font-family-text` | `sans-serif` | `base/body` |
| `--font-weight-text` | `$font-weight-regular` | `base/body` |
| `--line-height-text` | `1.5` | `base/body` |
| `--color-headline` | `inherit` | `base/headline` |
| `--font-family-headline` | `inherit` | `base/headline` |
| `--font-weight-headline` | `$font-weight-semi-bold` | `base/headline` |
| `--line-height-headline` | `1.1` | `base/headline` |
| `--margin-block-headline` | `0.75em 0` | `base/headline` |

## Tools

| Tool | Signature | Configurable |
| --- | --- | --- |
| `fluid-clamp()` | `fluid-clamp($min-size, $max-size, $min-breakpoint, $max-breakpoint, $unit: vw)` | `$fluid-clamp-baseline: 16px`, `$fluid-clamp-min-breakpoint: 320px`, `$fluid-clamp-max-breakpoint: 1920px` |
| `px-to-rem()` | `px-to-rem(24px)` — relative to the same baseline | `$fluid-clamp-baseline` |
| `vw()` | `vw($pixels, $base-vw: $layout-vw)` — one value or a shorthand list | `$layout-vw: 1440px` |
| `font-face()` | `@include font-face($font-name, $file-name, $weight: 400, $style: normal, $formats: woff2)` | `$font-path`, `$font-format-hints` |
| `icon-mask()` / `icon-background()` | `@include icon-mask($identifier)` — mask (tintable) or background SVG | `$icon-path: "../Icons/"` |
| `visually-hidden()` / `visually-hidden-focusable()` | `@include visually-hidden` — hide visually, keep it for assistive tech | — |
| `z()` | `z("sticky")` — named z-index lookup (from `tokens`) | `$z-index` map |
| `media()` | `@include media(">=tablet") { … }` — from `tools/media` | `$breakpoints` map |

`vw()` takes a single value or a list, so shorthand properties convert in one call.
Values that are not numbers pass through untouched:

```scss
.hero {
  height: vw(720px);      // 50vw
  margin: vw(15px 15px);  // 1.0416666667vw 1.0416666667vw
  padding: vw(40px auto); // 2.7777777778vw auto
}
```

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

The tokens that remain are structural, not brand-specific. Every one is a `!default`
variable, so `@use "…/tokens" with (…)` wins:

- **Breakpoints** — `$breakpoints` (`phone`, `phone-wide`, `tablet`, `tablet-wide`,
  `desktop`, `desktop-wide`, `full-hd`); also feeds `media()`.
- **Colors** — `$black`, `$white`. Neutral primitives, used as the fallbacks above.
- **Font weights** — `$font-weight-thin` (100) through `$font-weight-heavy` (900).
- **Layering** — the `$z-index` map (`below`, `base`, `raised`, `sticky`, `overlay`,
  `floating`, `modal`, `notification`), read through `z()`.

Configuration must appear on the **first** `@use` of `tokens` in the compilation (before
`base` loads it), so keep it at the top of your entry sheet.

## Development

The repo ships a [DDEV](https://ddev.com/) config, so the toolchain runs in the container
(Node 24) and nothing needs to be installed on the host. `ddev start` also installs the
npm dependencies via its post-start hook:

```bash
ddev start
ddev npm run test:build     # compile every example to example/.dist/
ddev npm run lint           # run every check over the whole tree
ddev npm run fix            # autofix styles and normalise package.json
```

`lint` aggregates three checks, each also runnable on its own:

```bash
ddev npm run lint:style             # stylelint the package sources
ddev npm run lint:editorconfig      # .editorconfig conformance
ddev npm run lint:package:normalize # package.json key order and formatting
```

`test:build` compiles the whole `example/` directory, so adding a file there — say
`example/font-face.scss` — is enough to have it covered; no script to touch. Files
prefixed with `_` are treated as Sass partials and skipped.

### Quality gates

A **pre-commit hook** runs the same checks against your *staged* files only, so unrelated
work in progress never blocks a clean commit. It is wired up by `npm install` — and
therefore by `ddev start` on a fresh clone — which points `core.hooksPath` at
`.githooks/`. In an existing clone, enable it once with:

```bash
ddev npm install            # or: git config core.hooksPath .githooks
```

The hook delegates to [lint-staged](https://github.com/lint-staged/lint-staged)
(`.lintstagedrc.json`) inside the container. Use `git commit --no-verify` to bypass it.

> lint-staged briefly hides unstaged changes to *partially staged* files while the checks
> run. If a run is interrupted, that work is recoverable from the backup stash — see
> `git stash list`. Your commit is unaffected either way, since git commits the index.

Every push additionally runs the whole-tree checks plus `test:build` in the **CGL**
workflow (`.github/workflows/cgl.yml`), so nothing depends on the hook having run.

Released automatically with semantic-release. Pairs with
[`@move-elevator/stylelint-config-scss`](https://github.com/move-elevator/stylelint-config-scss).
