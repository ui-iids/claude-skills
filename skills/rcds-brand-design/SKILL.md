---
name: rcds-brand-design
description: Build University of Idaho RCDS / IIDS research web applications and data products using the `@ui-iids/brand-assets` npm package. The package ships two tiers of components — strict brand primitives (header, footer, logos, CSS tokens) that should be used directly with prop customization, and loose theme pages (Agricultural, DataHub, etc.) that are style references, not scaffolds. The team builds many sites and explicitly does not want to lock into rigid templating: theme pages should guide visual style (palette emphasis, motifs, rhythm) for pages you write yourself, not wrap your content. Use this skill whenever the project depends on `@ui-iids/brand-assets`, or whenever the user mentions RCDS, IIDS, U of Idaho researcher-facing sites, brand-assets components, or theme pages like AgriculturalPage / DataHubPage / DiseaseModelingPage / InformationalPage / PromotionalPage / SearchPage / StoryMapPage.
---

# RCDS brand design (using `@ui-iids/brand-assets`)

You are helping a developer build a researcher-facing web application or
data product for the University of Idaho's **Research Computing and Data
Services (RCDS)** / Institute for Interdisciplinary Data Sciences (IIDS).
They are consuming the `@ui-iids/brand-assets` package as a dependency.

The package ships two very different kinds of thing under one roof, and
treating them the same way is the most common mistake. **You must
internalize the distinction below before writing any code.** Read it
carefully.

- **Brand primitives** — `BrandHeaderV2`, `BrandFooterV2`, the logo
  components, the CSS tokens. These are real, reusable components
  meant to be used directly and customized via props. They carry the
  university's identity, and they should look the same across every
  RCDS product. Lock these in.
- **Theme pages** — `AgriculturalPage`, `DataHubPage`,
  `DiseaseModelingPage`, `InformationalPage`, `PromotionalPage`,
  `SearchPage`, `StoryMapPage`. These are **style references**, not
  scaffolds. They illustrate how an agricultural product *should
  feel* — color emphasis, motifs, hero treatment, spacing rhythm —
  not how it must be structured. RCDS builds many sites, and the team
  explicitly does not want every "agricultural" site to share an
  identical layout.

The skill below is built around this distinction. When in doubt, ask:
am I touching a brand primitive (strict) or a theme page (loose)?

## Brand primitives (strict — use as-is, customize via props)

These carry the U of Idaho identity and must survive any customization.
Reach for them directly; customize through props, not by rewriting.

- **CSS tokens** in `@ui-iids/brand-assets/styles/branding.css`. All
  use the `--uidaho-` prefix — core colors (`--uidaho-pride-gold`,
  `--uidaho-black`, `--uidaho-white`, `--uidaho-silver`) and semantic
  aliases (`--uidaho-brand-primary`, `--uidaho-brand-on-primary`,
  `--uidaho-brand-text`, `--uidaho-brand-surface`,
  `--uidaho-brand-muted`, `--uidaho-brand-font-sans`,
  `--uidaho-brand-accent-border`, etc.). Always reference variables;
  never hardcode `#191919` or `#F1B300`. Open
  `node_modules/@ui-iids/brand-assets/styles/uidaho-branding.css` to
  see the full current list — that file is the source of truth, not
  this skill.
- **Logos.** Use `BrandLogo` or the named logo components
  (`UidahoFullColorHorizontal`, `Rcds`, `IidsStackedInverse`, etc.).
  Never inline your own SVG of a U of Idaho mark; never recolor a logo
  outside the variants the package already provides.
- **Headers & footers.** `BrandHeaderV2` and `BrandFooterV2` are the
  canonical site furniture. Customize them through their props
  (`title`, `subtitle`, `navItems`, `cta`, the `*Color` / `*ColorCustom`
  props) — don't fork them, and don't hand-roll your own.

What's variable on a brand primitive: copy, which optional rows
appear (pass `cta={null}`, `utilityLinks={[]}`), color emphasis within
the palette via the per-component `*Color` props, the logo variant.

What's invariant: the structural silhouette (utility row → brand row →
nav), the typography pairings, the palette itself, the logo treatment.

## Theme pages (loose — treat as a style guide, not a scaffold)

This is the part the team cares about most. Theme pages are
**guidance, not infrastructure**. Their job is to show you what a
"DataHub-flavored" or "Agricultural-flavored" page *looks and feels
like* — not to be dropped in around your content.

**Read them as reference, then build your own page.** Open the theme
page's source in `node_modules/@ui-iids/brand-assets/react-components/Pages/Themes/<Theme>/`
and study:

- Which brand tokens it emphasizes (which gold? which surface?).
- What motifs or graphic treatments recur (full-bleed hero?
  stat-card grid? data table styling? map-forward layout?).
- How it spaces and rhythms its sections.
- Which header/footer variant it pairs with.

Then write a page in your consumer repo that *evokes* those choices
with your actual content. Two products in the same theme family
should share a *visual vocabulary*, not a structural template. A
salmon habitat atlas and a wheat-yield dashboard might both feel
"Agricultural" — same palette emphasis, same kind of imagery — while
having entirely different layouts.

**Direct use is allowed but not the default.** If the theme page's
structure genuinely fits the product (e.g. a quick informational site
where `InformationalPage`'s layout is exactly right), importing it
directly is fine. But do not default to wrapping content in a theme
page. The team has explicitly said: do not lock RCDS into rigid
templating across sites.

When you propose a layout to the user, do not say "I'll use
`AgriculturalPage`." Say "I'll style this in the Agricultural theme —
[the specific cues you're picking up]." That phrasing keeps the
choice loose and visible.

## Installation and wiring

The package is **not published to the npm registry** — install it
directly from GitHub:

```bash
npm install github:ui-iids/brand-assets
# pin to a tag or commit for production:
npm install github:ui-iids/brand-assets#v0.1.0
```

The repo's `prepare` script runs `vite build` on install, so the
`dist/` artifacts are generated in the consumer's `node_modules`
automatically. Node **20+** is required (see `engines`).

Peer dependencies (`react`, `react-dom` 18.3+, `react-bootstrap`,
`bootstrap`, `@fortawesome/react-fontawesome`,
`@fortawesome/fontawesome-svg-core`, `@fortawesome/free-solid-svg-icons`)
are auto-installed by npm 7+ when missing. If the consumer is on an
older npm or sees peer-resolution warnings, add them explicitly.

On first install, the package's `postinstall` script prints a one-line
notice telling the developer they can run
`npx @ui-iids/brand-assets install-skills` to copy this skill (and any
others the package ships) into the consumer repo's `.claude/skills/`.
Suggest that command when you see the package as a fresh dependency
without `.claude/skills/rcds-brand-design/` present.

Import the stylesheets **once**, at the app entry point (e.g.
`main.tsx` / `App.tsx`), in this order:

```ts
import "bootstrap/dist/css/bootstrap.min.css";
import "@ui-iids/brand-assets/styles/branding.css"; // brand tokens (CSS vars)
import "@ui-iids/brand-assets/dist/index.css";       // component styles
```

The brand tokens stylesheet must load **before** the component
stylesheet so the CSS variables are defined when components reference
them.

## Upgrading

The package is versioned via git tags. To consume the latest changes:

1. Bump the pin in `package.json` (e.g.
   `"@ui-iids/brand-assets": "github:ui-iids/brand-assets#v0.2.0"`).
2. `npm install` — this re-runs the package's `prepare` build and
   re-prints the install-skills hint.
3. `npx @ui-iids/brand-assets install-skills --force` — overwrites the
   consumer repo's `.claude/skills/rcds-brand-design/` with the version
   shipped in the new package release.

Step 3 matters: the skill file in the consumer repo is a *copy* taken
at the last install, not a live link. If you skip it, Claude will keep
following the old skill (old export list, old token names, old
guidance) even though the components have moved on. Always pair a
package upgrade with a skill refresh.

If the consumer wants the latest unreleased changes,
`#main` works as a pin, but pin to a tag for anything that ships.

## What the package exports

All exports come from the package root (`@ui-iids/brand-assets`):

- **Headers**: `BrandHeader` (v1), `BrandHeaderV2` (current default).
- **Footers**: `BrandFooter` (v1), `BrandFooterV2` (current default).
- **Theme pages** (reference implementations — see the "Theme pages"
  section above for how to treat them): `InformationalPage`,
  `DataHubPage`, `DiseaseModelingPage`, `AgriculturalPage`,
  `PromotionalPage`, `SearchPage`, `StoryMapPage`. Each accepts
  `ThemePageProps`: `header?`, `footer?`, `headerVariant?`,
  `footerVariant?`, `currentPath?`, plus its own theme-specific props
  and `children`. These are exported so you *can* use them directly
  when the structure genuinely fits, but the default move is to read
  their source as a style reference and write your own page.
- **Logos**: `BrandLogo` (generic, takes a `LogoKey`), plus named
  components for every official lockup. `LOGOS`,
  `DEFAULT_LOGO_BY_ORG`, and `DEFAULT_POWERED_BY_BY_ORG` are the
  lookup tables.
- **Nav helpers**: `NavItem` type and `isNavItemActive(item, currentPath)`.

Default header/footer variant for theme pages is **v2**. Prefer v2 for
new work; v1 is kept for compatibility with existing sites.

## Public API surface — what you may import

Only these import paths are public API. Anything else is internal and
may move, rename, or break between versions without notice:

- **`@ui-iids/brand-assets`** — the root entry. All components, types,
  helpers, and constants listed above come from here. This is the
  only path that should appear in `import { ... }` statements.
- **`@ui-iids/brand-assets/styles/branding.css`** — the brand-tokens
  stylesheet. Imported once at the app root.
- **`@ui-iids/brand-assets/dist/index.css`** — the bundled component
  stylesheet. Imported once at the app root.

**Do not deep-import from internal paths.** Even though the package
ships its `react-components/` source for transparency, paths like
`@ui-iids/brand-assets/react-components/Headers/BrandHeaderV2` or
`@ui-iids/brand-assets/dist/Headers/...` are **not** part of the public
API. They are blocked by the package's `exports` map in most build
setups, and where they aren't blocked they will break silently on a
refactor. If something you need isn't reachable from the root entry,
that's a signal it should be exported — propose it upstream rather
than reaching in.

To discover the current export list at any version, read
`node_modules/@ui-iids/brand-assets/react-components/index.ts` (the
authoritative export manifest). When this skill and that file
disagree, **trust the file** — the package may have moved on since
this skill was last updated.

## How to build a page (the default pattern)

The default pattern composes brand primitives directly — header,
footer, and your own body content — drawing styling cues from the
relevant theme page's source.

```tsx
import { BrandHeaderV2, BrandFooterV2 } from "@ui-iids/brand-assets";

export function ProjectHome() {
  return (
    <>
      <BrandHeaderV2
        organization="rcds"
        title="Salmon Habitat Atlas"
        subtitle="A research data product of U of Idaho RCDS"
        navItems={[
          { label: "Overview", href: "/" },
          { label: "Data", href: "/data" },
          { label: "Methods", href: "/methods" },
        ]}
        currentPath="/"
        cta={{ label: "Request access", href: "/access" }}
      />
      <main>
        {/*
          Your page body. Style it to feel like the Agricultural theme
          (or DataHub, or whichever fits) — palette emphasis, motifs,
          spacing rhythm — using brand tokens. Read the theme page
          source for cues; do not wrap your content in the theme page
          unless its structure genuinely fits.
        */}
      </main>
      <BrandFooterV2
        office={{ name: "RCDS", address: "875 Perimeter Dr, Moscow, ID" }}
      />
    </>
  );
}
```

Using a theme page directly is also legal — it accepts fully-configured
`header`/`footer` React nodes as props and renders defaults otherwise.
But before reaching for it, ask whether you're using it because its
*structure* truly fits, or just because it carries the right *style*.
If only the style matters, build your own page and pull the style cues
from the theme page's source.

## Decision rule

The rule depends on which kind of thing you're touching.

**For brand primitives** (header, footer, logos, tokens) — these are
strict. Order of preference:

1. **Customize via props** (almost always). New copy, nav items, color
   emphasis, hidden rows, logo variant. No code changes to the package.
2. **Compose around it.** Wrap a primitive in a thin consumer-side
   component that pre-fills project defaults (e.g. a `ProjectHeader`
   that supplies the same nav across pages).
3. **Propose upstream.** If a primitive genuinely can't express what
   you need, the right move is a PR against `ui-iids/brand-assets`,
   not a local fork. Tell the user.
4. **Fork as last resort.** Only if the change is truly product-
   specific and the props can't express it. Flag the fork to the user
   — it drifts and misses upstream fixes.

**For theme pages** — these are loose. Order of preference:

1. **Read for style, build your own page** (the default). Open the
   theme page's source, pull color emphasis / motifs / rhythm cues
   into a page you write from scratch using brand primitives.
2. **Use the theme page directly.** Only when its structure happens
   to fit the product exactly. Don't reach for this just because the
   theme name matches the product domain.
3. **Don't propose theme pages upstream as a way to "lock in" a
   layout.** New theme pages added to the package should expand the
   style vocabulary (a new aesthetic family), not pin down how
   future products in an existing family must look.

## Brand tokens (the source of truth)

`@ui-iids/brand-assets/styles/branding.css` defines CSS custom
properties for colors, spacing, typography, and shadows. Whenever you
write product-specific CSS in the consumer app:

- Reference brand variables (`var(--uidaho-pride-gold)`,
  `var(--uidaho-black)`, `var(--uidaho-brand-primary)`,
  `var(--uidaho-brand-text)`, etc.) rather than hardcoding values.
- The semantic aliases (`--uidaho-brand-*`) are usually the right choice
  over the raw color names — they re-point if the palette shifts.
- If you don't know whether a token exists, **read
  `node_modules/@ui-iids/brand-assets/styles/uidaho-branding.css`
  before guessing**. Variable names not defined there will silently
  fall through and produce unstyled output.
- If you find yourself reaching for a value that *should* be a brand
  token but isn't, that's a signal to propose it upstream — tell the
  user.

## Logos

Use `BrandLogo` for parameterized cases and the named components
(`Rcds`, `IidsStackedInverse`, `UidahoFullColorHorizontal`, etc.)
when the lockup is fixed. `DEFAULT_LOGO_BY_ORG["rcds"]` gives the
canonical lockup for an organization. Never inline an `<svg>` copy
of a U of Idaho mark in the consumer app.

## What a good product looks like

- Imports brand and component CSS once at the app root, in the right order.
- Uses `BrandHeaderV2` and `BrandFooterV2` directly as site furniture; customizes them through props.
- Page bodies are written for the actual product content, drawing visual cues (palette emphasis, motifs, spacing) from the relevant theme page's source rather than wrapping content inside a theme page.
- Consumer-side CSS only references brand tokens; no hardcoded `#191919` or `#F1B300`.
- Two RCDS products in the same theme family feel like cousins, not clones — same vocabulary, different structures.
- Treats the U of Idaho identity (palette, logos, header/footer silhouette) as fixed and the product's content, structure, and feature set as the variable layer.

## When the user asks you to do something off-brand

Push back briefly. Explain which invariant the request crosses
(palette, logo treatment, structural silhouette), offer a branded
alternative, and only proceed off-brand if the user confirms they
understand. RCDS products represent the university — drift here has
institutional cost, not just aesthetic cost.
