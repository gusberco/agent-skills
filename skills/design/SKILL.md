---
name: design
description: Reads, creates, validates, and applies DESIGN.md files — Google's format for describing a visual identity to coding agents. Use when a DESIGN.md file exists in the project, when the user asks to create a design system, when implementing UI that must follow brand tokens, or when linting/comparing design systems.
---

# Design System (DESIGN.md)

Work with DESIGN.md files — a format for encoding a product's visual identity as machine-readable design tokens plus human-readable rationale. Tokens give agents exact values; prose tells them why those values exist.

## How It Works

1. **Detect** — Check whether a `DESIGN.md` file exists in the project root
2. **Parse** — Extract YAML front-matter tokens and markdown section prose
3. **Validate** — Run the linter to catch broken references, contrast failures, or structural issues
4. **Apply** — Map tokens to the codebase (Tailwind config, CSS variables, component props)
5. **Maintain** — Update the file when the design system evolves; commit it alongside the code

## File Structure

A valid DESIGN.md has two layers:

```
---                        ← YAML front matter (machine-readable tokens)
version: alpha
name: My Brand
colors:
  primary: "#1A1C1E"
  secondary: "#6C7278"
typography:
  h1:
    fontFamily: Public Sans
    fontSize: 48px
    fontWeight: 600
    lineHeight: 1.1
    letterSpacing: -0.02em
rounded:
  sm: 4px
  md: 8px
spacing:
  sm: 8px
  md: 16px
components:
  button-primary:
    backgroundColor: "{colors.primary}"
    textColor: "#ffffff"
    rounded: "{rounded.md}"
    padding: 12px
---

## Overview          ← Markdown body (human-readable rationale, ## headings)
## Colors
## Typography
## Layout
## Elevation & Depth
## Shapes
## Components
## Do's and Don'ts
```

## Token Reference

### Token Types

| Type | Format | Example |
|------|--------|---------|
| Color | `#` + hex (sRGB) | `"#1A1C1E"` |
| Dimension | number + unit (px, em, rem) | `48px`, `-0.02em` |
| Token Reference | `{path.to.token}` | `{colors.primary}` |
| Typography | object with font properties | see above |

### Typography Properties

- `fontFamily` — string
- `fontSize` — Dimension
- `fontWeight` — number (400, 600, 700…)
- `lineHeight` — Dimension or unitless number (multiplier)
- `letterSpacing` — Dimension
- `fontFeature` — CSS `font-feature-settings` value
- `fontVariation` — CSS `font-variation-settings` value

### Component Properties

`backgroundColor`, `textColor`, `typography`, `rounded`, `padding`, `size`, `height`, `width`

Variants use related key names: `button-primary`, `button-primary-hover`, `button-primary-active`.

### Recommended Token Names

**Colors:** `primary`, `secondary`, `tertiary`, `neutral`, `surface`, `on-surface`, `error`

**Typography:** `headline-display`, `headline-lg`, `headline-md`, `body-lg`, `body-md`, `body-sm`, `label-lg`, `label-md`, `label-sm`

**Rounded:** `none`, `sm`, `md`, `lg`, `xl`, `full`

## CLI Commands

```bash
# Validate a DESIGN.md file
npx @google/design.md lint DESIGN.md

# Compare two versions
npx @google/design.md diff DESIGN.md DESIGN-v2.md

# Export to Tailwind theme
npx @google/design.md export --format tailwind DESIGN.md > tailwind.theme.json

# Export to W3C DTCG tokens.json
npx @google/design.md export --format dtcg DESIGN.md > tokens.json

# Print the full format spec (useful context for agents)
npx @google/design.md spec
```

## Lint Rules

| Rule | Severity | What it checks |
|------|----------|----------------|
| `broken-ref` | error | Token references `{colors.primary}` that don't resolve |
| `missing-primary` | warning | Colors defined but no `primary` color |
| `contrast-ratio` | warning | `backgroundColor`/`textColor` pairs below WCAG AA (4.5:1) |
| `orphaned-tokens` | warning | Color tokens defined but never referenced by a component |
| `token-summary` | info | Count of tokens in each section |
| `missing-sections` | info | Optional sections absent when other tokens exist |
| `missing-typography` | warning | Colors defined but no typography tokens |
| `section-order` | warning | Sections out of canonical order |

## Applying Tokens to Code

### CSS Custom Properties

```css
:root {
  --color-primary: #1A1C1E;
  --color-secondary: #6C7278;
  --color-tertiary: #B8422E;
  --color-neutral: #F7F5F2;

  --font-size-h1: 48px;
  --font-family-heading: 'Public Sans', sans-serif;

  --rounded-sm: 4px;
  --rounded-md: 8px;

  --spacing-sm: 8px;
  --spacing-md: 16px;
}
```

### Tailwind Config (from export)

```bash
npx @google/design.md export --format tailwind DESIGN.md > tailwind.theme.json
```

Then merge into `tailwind.config.js`:

```js
const designTokens = require('./tailwind.theme.json');

module.exports = {
  theme: {
    extend: designTokens,
  },
};
```

### Programmatic API

```js
import { lint } from '@google/design.md/linter';

const report = lint(markdownString);
console.log(report.findings);      // Finding[]
console.log(report.summary);       // { errors, warnings, info }
console.log(report.designSystem);  // Parsed DesignSystemState
```

## Workflow

### Reading an Existing DESIGN.md

1. Read the YAML front matter — these are normative values
2. Read each `##` section for rationale and application guidance
3. Run `npx @google/design.md lint DESIGN.md` to catch issues
4. Apply tokens to the codebase via CSS variables, Tailwind config, or component props

### Creating a New DESIGN.md

1. Gather brand assets: colors (hex), fonts, spacing scale, corner radii
2. Draft the YAML front matter with at minimum `name` and `colors.primary`
3. Write `## Overview` with brand personality description
4. Write `## Colors`, `## Typography`, `## Layout`, `## Shapes`, `## Components`
5. Run `npx @google/design.md lint DESIGN.md` and fix any errors
6. Commit the file: `git add DESIGN.md && git commit -m "add: DESIGN.md"`

### Updating DESIGN.md

1. Update the spec first — tokens and prose both
2. Run `npx @google/design.md diff DESIGN.md DESIGN-v2.md` to review changes
3. Update code that references changed tokens
4. Run the linter to verify no regressions

## Section Order (Canonical)

1. Overview (also: "Brand & Style")
2. Colors
3. Typography
4. Layout (also: "Layout & Spacing")
5. Elevation & Depth (also: "Elevation")
6. Shapes
7. Components
8. Do's and Don'ts

Sections can be omitted; those present must follow this order. Duplicate section headings are an error.

## Consumer Behavior for Unknown Content

| Scenario | Behavior |
|----------|----------|
| Unknown section heading | Preserve; do not error |
| Unknown color token name | Accept if value is valid |
| Unknown typography token name | Accept as valid typography |
| Unknown component property | Accept with warning |
| Duplicate section heading | Error; reject the file |

## Red Flags

- Hardcoding hex values in components when a DESIGN.md token exists
- Skipping the linter before applying tokens — contrast failures and broken refs will silently corrupt the UI
- Modifying token values in code without updating DESIGN.md — the file is the source of truth
- Using unsupported token reference paths (e.g., `{colors}` instead of `{colors.primary}`)
- Applying tokens before running `lint` — at minimum, check for errors before implementing

## Verification

After creating or updating a DESIGN.md:

- [ ] `npx @google/design.md lint DESIGN.md` exits 0 (no errors)
- [ ] All `{token.references}` resolve to defined tokens
- [ ] At least `colors.primary` is defined
- [ ] Component `backgroundColor`/`textColor` pairs pass WCAG AA (4.5:1)
- [ ] Section order matches the canonical order
- [ ] File is committed to version control
