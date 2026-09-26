# AGENTS.md

## This is NOT the Astro you know

This project runs **Astro 7**. APIs, conventions, and file structure may all differ
from your training data. Astro does not ship docs inside `node_modules`, so before
writing any Astro-specific code (config, content collections, actions, sessions,
adapters, image/font APIs, middleware), verify against one of these — in order:

1. The installed type definitions: `node_modules/astro/dist/types/public/*.d.ts`,
   and the feature's own folder (e.g. `node_modules/astro/dist/actions/`).
2. The current docs at <https://docs.astro.build> — fetch them, don't recall them.
3. The Astro releases page for what changed in the installed version:
   <https://github.com/withastro/astro/releases>.

Check the installed version first (`package.json` / `node_modules/astro/package.json`),
and heed deprecation notices in the types and release notes.

Work-in-progress. Only what is written below is established — do not infer other
conventions from the code.

## How this project is layered

Three layers, with different rules about who may change them. Knowing which one you
are touching answers most "should I change this?" questions.

| Layer | Lives in | Who owns it |
| :-- | :-- | :-- |
| **Tokens** | `src/styles/tokens.css` | The designer. Compose from these; never invent a parallel scale. |
| **Contracts** | Class names, semantics, ARIA wiring, component props | Frozen. Other code and the skills depend on these names. |
| **Implementation** | The CSS behind a contract (`form.css`, component `<style>`) | Disposable. Restyle it freely — the contract is what survives. |

`src/styles/form.css` is the clearest example: it is meant to be replaced wholesale,
while the markup contract it styles (`.input-group`, `.input`, `.field-error`) stays
put. See `.claude/skills/create-form/SKILL.md`.

## Toolchain is pinned

Formatting and linting must produce identical results on every machine, so the
tools that decide pass/fail are pinned to exact versions in `package.json`:
`prettier`, `prettier-plugin-astro`, `stylelint`, `postcss-html`, `typescript`,
`@astrojs/check`. Prettier can change its output in a minor release — on a
caret range one `pnpm update` reformats the whole repo and buries a real diff.
Feature dependencies (`astro`, `gsap`, `@astrojs/sitemap`) keep caret ranges;
the committed lockfile covers reproducibility there.

Do not widen these to `^` or `~`. To upgrade one, change the exact version
deliberately and commit the reformat separately from any other change.

`pnpm build` runs `format:check` first, so unformatted code fails the build
regardless of whether the author's editor has a Prettier plugin.

## Enforcement

The CSS rules below are **not advisory** — `stylelint.config.js` and
`.stylelint/plugins.js` enforce them, and `pnpm build` runs the linter before
`astro check`. A violation fails the build.

Everything in "Working with tokens" is guidance rather than a lint rule, because it
needs judgement. Follow it anyway.

Deliberate exceptions are fine when you say why:

```css
/* stylelint-disable-next-line declaration-property-value-disallowed-list -- Slack brand blue, not themed */
background-color: #4a154b;
```

## Rules

### CSS: every `<style>` tag must follow the `@layer` setup

The canonical layer order is declared in `src/styles/global.css`:

```css
@layer base, component, utility;
```

Any `<style>` tag added to an `.astro` file must re-declare that order and place
its rules in the appropriate layer:

```astro
<style>
@layer base, component, utility;

@layer component {
  .my-thing { ... }
}
</style>
```

See `src/components/Button.astro` for the reference implementation.

**Both lines are required.** Layer priority is set by first-declaration order, so
without the `@layer base, component, utility;` line a component's `@layer component`
block declares `component` first, ranking it *below* `base` — and the base reset
(`button { all: unset }`, `*:not(dialog) { margin: 0 }`) silently wipes out the
component's styling.

The build minifier strips the order statement from the output. That is expected:
it reads the declaration, reorders the layer blocks to match, then drops it as
redundant. Its absence from `dist/` does not mean it was ignored.

Never leave rules unlayered. Unlayered CSS beats every layered rule regardless of
specificity, so an unlayered component style silently overrides the `utility`
layer and `u-*` classes stop working on it.

### CSS: `:hover` must be gated behind a hover-capable media query

Never write a bare `:hover` rule:

```css
@media (hover: hover) and (pointer: fine) {
  .my-thing:hover { ... }
}
```

The media query goes inside the `@layer` block, not around it.

### CSS: no hardcoded colors on theme-reactive properties

`color`, `background*`, `border*`, `fill`, `stroke`, `outline-color`, and
`text-decoration-color` must take their value from a token, never a hex literal.

```css
/* wrong — renders fine in light, silently breaks in dark */
color: #202020;

/* right */
color: var(--color-text-strong);
```

Every color in the system resolves through `light-dark()` in `tokens.css`. A hex
literal opts out of that: the page looks correct in light mode, breaks in dark, and
nothing catches it — the build passes, `astro check` passes, and a review done in
light mode sees nothing wrong.

Gradient stops, shadows, and third-party brand colors are not theme-reactive and are
not covered by this rule.

## Marking placeholders

Starter values that are *wrong until someone sets them* — a brand name, the site
URL, a default author — carry a `bp-placeholder:` comment beside them. The text
after the colon is the instruction, and it may wrap onto following comment lines.

```js
// bp-placeholder: Site URL. Feeds canonical tags, absolute OG image URLs,
// robots.txt and the sitemap — set this before any deploy.
site: "https://your-site.com",
```

`scripts/check-placeholders.mjs` collects them. `pnpm build` reports them and
carries on; `pnpm build:release` fails while any remain.

**Introducing a placeholder value means adding the marker.** The script matches
only `bp-placeholder:` and reads the hint from the comment, so it never needs
editing — but an unmarked placeholder is invisible to it and will ship.

**A `bp-placeholder` is not a TODO.** A TODO is work to do eventually; a
placeholder is a value that is incorrect right now. Ordinary TODOs are not
gated — a release should not be blocked by a "review this code" note.

The prefix is deliberate: a bare `placeholder` would collide with the HTML
attribute and the `::placeholder` pseudo-element used throughout the form CSS.

## Working with tokens

`src/styles/tokens.css` is the design system. It is the vocabulary — compose from it
rather than inventing values beside it.

| Group | Tokens | Use for |
| :-- | :-- | :-- |
| Color | `--color-text-*`, `--color-bg-*`, `--color-border-*` | All themed color. Semantic layer only — reach for `--primitive-*` only when defining a semantic token. |
| Type size | `--text-h1`…`--text-h6`, `--text-xs`…`--text-xl`, `--text-eyebrow` | Font sizes. Fluid; do not write raw `font-size` values. |
| Type style | `--weight-*`, `--leading-*`, `--tracking-*` | Weight, line-height, letter-spacing. |
| Space | `--space-2`…`--space-96` | Padding and margin on the page rhythm. |
| Space (relative) | `--space-em-*` | Spacing that should scale with the element's own font size. |
| Gap | `--gap-xs`…`--gap-xl` | Flex/grid `gap`. |
| Layout | `--page-gutter`, `--section-padding-default`, `--container-max-default` | Page gutters, vertical section rhythm, and max widths. |
| Radius | `--radius-sm/md/lg` | Corner radii. |
| Motion | `--duration-*`, `--ease-*` | Transitions. `--duration-*` collapses to ~0 under `prefers-reduced-motion`, so use it rather than a literal. |

**Never invent a value where a token exists.** `padding: var(--space-24)`, not
`padding: 1.5rem`. This is not enforced by the linter — a rule banning raw lengths
fires on ~64 legitimate uses in this repo — so it is on you.

**Raw lengths are fine where no token applies.** Hairline borders (`1px`), icon
box sizing (`1em`), optical nudges (`0.05em`), and one-off layout constraints
(`max-width: 40rem`, `65ch`) are not scale decisions and have no token. Use a
literal and move on.

**The space scale is static below `--space-12` and fluid at or above it.** So there
is no token for a *constant* large value — if you need a fixed `4rem` that must not
scale with the viewport, write it literally.

**Adding a token is a design decision, not an implementation one.** If something
genuinely needs a new token, say so and let the designer add it. Do not add it to
`tokens.css` as a side effect of building a component.
