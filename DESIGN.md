---
name: Retina Project Explainer
description: Premium research explainer for the Retina CAD project (dark + light themes)
colors:
  halo-blue: "#7FB2FF"
  halo-blue-light: "#4A7FCF"
  bg-void: "#030303"
  bg-paper: "#F7F8FA"
  bg-deep: "#0A0A0A"
  bg-deep-light: "#EDEEF2"
  surface-low: "#111111"
  surface-low-light: "#FFFFFF"
  surface-high: "#171717"
  surface-high-light: "#F0F1F4"
  line-smoke: "#2A2A2A"
  line-smoke-light: "#D6DAE2"
  text-high: "#F5F7FA"
  text-high-light: "#17191E"
  text-mid: "#8C919A"
  text-mid-light: "#555B66"
  text-low: "#666A73"
  text-low-light: "#657080"
  technical-surface: "#070707"
  technical-surface-light: "#EEF1F5"
  technical-ink-light: "#20252D"
  technical-line-light: "#C7CDD6"
typography:
  display:
    fontFamily: "ui-sans-serif, -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif"
    fontSize: "clamp(3rem, 6vw, 5.5rem)"
    fontWeight: 500
    lineHeight: 0.96
    letterSpacing: "-0.03em"
  headline:
    fontFamily: "ui-sans-serif, -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif"
    fontSize: "clamp(1.75rem, 3vw, 2.5rem)"
    fontWeight: 500
    lineHeight: 1.05
    letterSpacing: "-0.02em"
  body:
    fontFamily: "ui-sans-serif, -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif"
    fontSize: "1rem"
    fontWeight: 400
    lineHeight: 1.72
    letterSpacing: "0"
  label:
    fontFamily: "ui-sans-serif, -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif"
    fontSize: "0.875rem"
    fontWeight: 500
    lineHeight: 1.4
    letterSpacing: "0.01em"
rounded:
  sm: "12px"
  md: "18px"
  pill: "999px"
spacing:
  xs: "8px"
  sm: "12px"
  md: "16px"
  lg: "24px"
  xl: "32px"
components:
  panel-floating:
    backgroundColor: "{colors.surface-low}"
    textColor: "{colors.text-high}"
    rounded: "{rounded.md}"
    padding: "{spacing.lg}"
  chip-outline:
    backgroundColor: "{colors.surface-high}"
    textColor: "{colors.text-high}"
    rounded: "{rounded.pill}"
    padding: "8px 14px"
  prompt-shell:
    backgroundColor: "{colors.surface-high}"
    textColor: "{colors.text-mid}"
    rounded: "{rounded.pill}"
    padding: "20px 24px"
  button-outline:
    backgroundColor: "{colors.surface-low}"
    textColor: "{colors.text-high}"
    rounded: "{rounded.pill}"
    padding: "12px 20px"
  status-quiet:
    backgroundColor: "{colors.surface-high}"
    textColor: "{colors.text-mid}"
    rounded: "{rounded.pill}"
    padding: "8px 12px"
---

# Design System: Retina Project Explainer

## Overview

**Creative North Star: "Orbital Clinical Briefing"**

This system should feel like a research briefing projected onto a black field: calm, exact, and slightly futuristic. The mood is not cinematic chaos. It is controlled atmosphere. The page should give clinicians and collaborators the sense that the work is advanced, but already under discipline.

The design language is sparse and premium. Large white type carries the narrative. Surfaces are few, dark, and quiet. Accent color appears as a faint halo, not a rainbow. Density stays low so the distinction between current evidence and future roadmap remains easy to parse.

This system explicitly rejects the anti-references in `PRODUCT.md`: salesy SaaS landing page, busy dashboard, playful consumer AI site, flashy gradients, infographic clutter, over-carded layouts, and loud promotional copy.

Boxes are not the default grammar. Cards are a fallback, not a design language. When information can be separated by spacing, typography, rules, or sequence, do that first.

**Key Characteristics:**
- Black-led palette with one cold halo accent (dark); clean white-led palette (light)
- Sans-serif typography with surgical spacing and tight hierarchy
- Tonal depth before visible decoration
- Sparse composition, long breathing room, few containers
- Scientific trust over AI spectacle

## Colors

The palette is nearly monochrome. Its job is to stage clarity, not decorate the page. Two themes are supported: dark (default) and light. Both are detected automatically via `prefers-color-scheme` and can be toggled manually.

### Dark Theme

#### Primary
- **Cold Halo Blue** (`#7FB2FF`): Reserved for the rare moments that need signal, active states, orbital glows, selected markers, and one focal interaction per viewport.

#### Neutral
- **Void Black** (`#030303`): The dominant page field. Use it for the outer canvas and hero backdrop.
- **Deep Black** (`#0A0A0A`): Secondary field for soft sectional separation when pure black needs relief.
- **Low Surface Graphite** (`#111111`): Primary surface color for rare structural panels and anchored sections.
- **High Surface Graphite** (`#171717`): Elevated surfaces, prompt shells, pills, and layered controls.
- **Smoke Line** (`#2A2A2A`): Hairline borders, dividers, and quiet structural edges.
- **High Ink** (`#F5F7FA`): Headings, primary body copy, and all critical labels.
- **Mid Ink** (`#8C919A`): Secondary copy, metadata, and helper text.
- **Low Ink** (`#666A73`): De-emphasized annotations only. Never use this for long body copy.

**The One Halo Rule.** `#7FB2FF` is never a second background. It is a signal accent on 10% or less of any screen. Its rarity is the point.

### Light Theme

The light theme adapts the same design principles to a clean white field. The north star shifts to "Clean Clinical Document" -- like reading the same research briefing in print rather than on a projection screen. The mood stays premium, calm, and precise but on an off-white field with dark ink.

#### Primary
- **Cold Halo Blue** (`#4A7FCF`): Same role as dark theme. Slightly darker to maintain WCAG AA contrast (3.68:1 large text, 3:1 UI components) on the light background.

#### Neutral
- **Paper Field** (`#F7F8FA`): Dominant page background. Clean off-white, easier on the eyes than pure white.
- **Paper Deep** (`#EDEEF2`): Secondary background for soft sectional separation.
- **White Surface** (`#FFFFFF`): Primary surface for panels, cards, and anchored sections.
- **Elevated Surface** (`#F0F1F4`): Elevated surfaces and layered controls.
- **Light Line** (`#D6DAE2`): Hairline borders, dividers, and structural edges.
- **Light Line Soft** (`#E4E7ED`): Subtle borders and dividers.
- **Light Line Strong** (`#C6CAD4`): Emphasized structural edges.
- **Dark Ink** (`#17191E`): Headings, primary body copy, and critical labels.
- **Mid Ink** (`#555B66`): Secondary copy, metadata, and helper text.
- **Supporting Ink** (`#657080`): Secondary labels and annotations. It meets AA contrast on the paper field. Do not use it to make navigation disappear.

#### Technical Surfaces
- **Technical Surface** (`#EEF1F5`): Code, equations, contingency tables, and other dense technical specimens. This is a cool gray surface, visibly separate from the page without becoming a dark-mode island.
- **Technical Ink** (`#20252D`): All text inside a technical surface.
- **Technical Line** (`#C7CDD6`): Rules and table edges inside a technical surface.

Technical content must use these semantic tokens. Do not hard-code a black or near-black background for a technical component and rely on inherited theme colors.

#### Shadows
Shadows are retained but dialed down in opacity to suit the lighter canvas:
- **Ambient Low** (`0 8px 24px rgba(0, 0, 0, 0.05)`)
- **Ambient Mid** (`0 18px 48px rgba(0, 0, 0, 0.06)`)
- **Surface Edge** (`0 0 0 1px rgba(0, 0, 0, 0.06)`)

#### The One Halo Rule (unchanged)
`#4A7FCF` is never a second background. It is a signal accent on 10% or less of any screen. Its rarity is the point.

### Theme Detection

Theme is detected automatically via the `prefers-color-scheme` CSS media query. No manual toggle needed -- the page responds purely to the OS/browser preference.

The `:root` block also sets `color-scheme: dark light` so the browser knows both themes are supported.

Implementation pattern (add to every new doc page):
- Add `color-scheme: dark light;` to `:root`.
- Define semantic custom properties for every surface, including `--technical-surface`, `--technical-ink`, and `--technical-line`. Never theme a component by hard-coding dark literals.
- Add one final `@media (prefers-color-scheme: light)` block before `</style>` that overrides every custom property, including the technical-surface tokens. The final position lets it correct legacy component rules.
- In light mode, contents navigation uses dedicated readable tones: `#596574` for top-level links and `#687280` for nested links. `--text-low` is `#657080` and is for supporting content, not navigation.
- Keep code, formulas, tables, and dense technical specimens on `--technical-surface`; their descendants inherit `--technical-ink` and structural rules use `--technical-line`.

## Typography

**Display Font:** `ui-sans-serif, -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif`
**Body Font:** `ui-sans-serif, -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif`
**Label/Mono Font:** None by default. Use the body family for labels.

**Character:** Quiet, contemporary, clinical. The typography should feel engineered rather than editorial, with enough warmth in spacing to avoid looking sterile.

### Hierarchy
- **Display** (500, `clamp(3rem, 6vw, 5.5rem)`, 0.96): Hero statements and one-line claims only.
- **Headline** (500, `clamp(1.75rem, 3vw, 2.5rem)`, 1.05): Section heads, diagram titles, and major status labels.
- **Title** (500, `1.125rem`, 1.2): Panel titles, status blocks, and short explanatory headings.
- **Body** (400, `1rem`, 1.72): Main narrative copy. Cap measure at 65–72ch.
- **Label** (500, `0.875rem`, 0.01em, sentence case): Metadata, pills, and compact UI labels.

**The Silent Hero Rule.** Hero typography carries scale and spacing, not effects. No serif swaps, no gradient text, no over-tight tracking below `-0.04em`.

## Elevation

Depth comes from tonal layering, blur-free softness, and restrained ambient shadow. Surfaces should emerge from black by value shift first, then by a narrow edge, then by a soft shadow. If a panel is already bordered, the shadow stays tight and quiet.

### Shadow Vocabulary
- **Ambient Low** (`0 8px 24px rgba(0, 0, 0, 0.22)`): Hover lift for pills, chips, and small controls.
- **Ambient Mid** (`0 18px 48px rgba(0, 0, 0, 0.34)`): Large prompt shells and hero modules.
- **Halo Edge** (`0 0 0 1px rgba(255,255,255,0.08)`): Structural edge for dark surfaces when a border is needed without looking boxed.

**The Atmosphere, Not Glass Rule.** Use tonal layering and ambient shadow. Decorative glassmorphism is forbidden.

## Components

### Buttons
- **Shape:** Full pill (`999px`) or softly curved pill edges for major calls to action.
- **Primary:** Dark surface (`#171717`) with high-ink label (`#F5F7FA`), quiet outline (`#2A2A2A`), padding around `12px 20px`.
- **Hover / Focus:** Border brightens toward `rgba(255,255,255,0.18)`, shadow tightens, halo accent may appear on an icon or focus ring.
- **Secondary / Ghost:** Same silhouette, less fill, same text contrast. Never louder than the primary action.

### Chips
- **Style:** Small full-pill capsules on `#171717` with `#F5F7FA` or `#8C919A` text and a single smoke-line border.
- **State:** Selected chips may borrow the halo accent in text or ring only. Do not fill chips with saturated blue.

### Panels / Containers
- **Use Case:** Rare structural grouping only. A panel must earn its presence by clarifying status, separating a major region, or anchoring a diagram. Do not use repeated cards as the page's default layout.
- **Corner Style:** Gently curved edges (`18px`).
- **Background:** Mostly `#111111`, with `#171717` reserved for stronger elevation.
- **Shadow Strategy:** Ambient, not wide. If a border exists, shadow blur stays controlled.
- **Border:** Single smoke-line edge (`#2A2A2A` or `rgba(255,255,255,0.08)`).
- **Internal Padding:** `24px` default for large panels, `16px` for compact blocks.

### Inputs / Fields
- **Style:** Large dark prompt shells with pill geometry, muted placeholder text, and one clear send or advance affordance.
- **Focus:** The ring sharpens, the text brightens, and the action control gains the halo accent.
- **Error / Disabled:** Prefer copy and opacity shifts over loud error red unless the state is truly destructive.

### Navigation
- **Style:** Low-contrast top navigation on black, sentence-case labels, generous spacing, quiet hover brightening.
- **Active State:** Contrast increase or subtle underline. No bright tabs, no boxed menu chrome.
- **Language / Utility Controls:** Treat as compact pills, not as separate noisy widgets.

### Signature Component
- **Prompt Shell:** The hero interaction is a long, dark, rounded container that feels like an invitation to inspect or query the research. It is the one place where the system can feel slightly more tactile than the rest of the page.

## Do's and Don'ts

### Do:
- **Do** keep the page field in the `#030303` to `#111111` range (dark) or `#F7F8FA` to `#FFFFFF` range (light) and let whitespace do the work.
- **Do** maintain WCAG AA contrast in both themes: `#F5F7FA` on dark surfaces, `#17191E` on light surfaces for primary copy.
- **Do** reserve `#7FB2FF` (dark) or `#4A7FCF` (light) for one focal interaction, one selected state, or one ambient halo per viewport.
- **Do** keep long-form explanatory copy within 65-72ch and sentence case.
- **Do** separate implemented evidence from roadmap with hierarchy, pacing, and grouping before adding any extra color.
- **Do** prefer rhythm, rules, diagram flow, and type hierarchy before reaching for another box.
- **Do** include the automatic theme detection block in every doc page: the `@media (prefers-color-scheme: light)` CSS block overriding `:root` CSS custom properties.
- **Do** keep formulas, code, and dense technical tables on the semantic technical surface in both themes. In light mode, they are cool gray with dark ink, never a black panel.
- **Do** give contents navigation its own readable light-theme tones instead of using de-emphasized annotation color.

### Don't:
- **Don't** make this feel like a salesy SaaS landing page.
- **Don't** make this feel like a busy dashboard.
- **Don't** make this feel like a playful consumer AI site.
- **Don't** use flashy gradients, infographic clutter, over-carded layouts, or loud promotional copy.
- **Don't** use cards as the default answer. Repeated card grids, metric-card walls, and feature-card rows are boring here and immediately push the page toward generic SaaS/dashboard language.
- **Don't** trap every fact in its own boxed module; that kills pacing and makes the explainer feel congested.
- **Don't** use serif hero type, repeating grid overlays, rainbow cohort color systems, or metric-card walls as the page's main grammar.
- **Don't** pair a visible border with a giant decorative shadow or blur-heavy glass treatment.
- **Don't** hard-code dark technical backgrounds or pale inherited text inside a component. Theme through semantic tokens so both modes remain coherent.
- **Don't** use the low-ink token for a contents index in light mode.

## Antipatterns
- Repeated equal-weight cards for every section.
- Nested panels inside panels when spacing and a rule would do.
- KPI tiles presented like a SaaS dashboard.
- Roadmap and implemented work rendered with the same container weight, same glow, or same visual certainty.
