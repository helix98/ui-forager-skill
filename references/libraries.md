# UI Library Reference

Pick 1-3 libraries that match the need before fetching. Most sites expose copy-paste code on a `/docs/...` or `/components/...` page — `web_fetch` that page directly. If a page is React-heavy and the code doesn't render in the fetched text, try appending `/code` or look for a "Copy code" / raw GitHub link on the page.

## Core component systems (primitives, forms, general UI)
- **shadcn/ui** — https://ui.shadcn.com/ — the baseline: unstyled-ish, accessible Radix-based components you own the code for. Default starting point for buttons, dialogs, forms, tables, nav. Most other libraries here assume/extend shadcn conventions (`cn()`, CVA, Tailwind).
- **Base UI** — https://base-ui.com/ — unstyled, accessible primitives (from the Radix/Floating UI team). Use when you want full styling control with zero visual opinion.
- **Origin UI** — https://www.originui-ng.com/ — large set of styled, ready-to-use shadcn-style components (inputs, pickers, nav) with more visual polish out of the box than raw shadcn.
- **OriginKit** — https://www.originkit.dev/ — shadcn-style component collection (distinct from Origin UI above — check both, they're separate sites).
- **Hextaui** — https://www.hextaui.com/ — shadcn-style component collection, general purpose UI.
- **ReUI** — https://reui.io/ — shadcn-compatible components, general UI + some data/table pieces.
- **Kibo UI** — https://www.kibo-ui.com/ — shadcn-based, leans toward more app/dashboard-oriented composite components (editors, kanban, etc).
- **JollyUI** — https://www.jollyui.dev/ — React Aria + Tailwind components, strong accessibility focus.
- **8bitcn** — https://www.8bitcn.com/ — shadcn components restyled in an 8-bit/retro pixel aesthetic. Use for playful/retro-themed UI.
- **Shadway** — https://shadway.online/ — shadcn-style component variants.
- **ShadcnSpace** — https://shadcnspace.com/ — more shadcn-compatible component variants/blocks.
- **Systaliko UI** — https://systaliko-ui.vercel.app/ — shadcn-style component collection.
- **efferd** — https://efferd.com/ — component collection (check current offering when fetched, site is smaller/less documented).
- **ui.tripled.work** — https://ui.tripled.work/ — component collection (check current offering when fetched).
- **UI Lora** — https://www.uilora.com/get-started/web/components — web component collection, general UI.
- **Unlumen UI** — https://ui.unlumen.com/ — component collection (check current offering when fetched).
- **SmoothUI** — https://smoothui.dev/ — component collection leaning toward smooth/polished micro-interactions.
- **VengenceUI** — https://www.vengenceui.com/ — component collection (check current offering when fetched).

## Marketing pages / landing sections / blocks
- **shadcnblocks** — https://www.shadcnblocks.com/ — pre-built marketing sections (heroes, pricing, testimonials, footers) built on shadcn. Good first stop for landing page sections.
- **Tailark** — https://tailark.com/ — Tailwind marketing blocks/sections, framework-agnostic-ish Tailwind markup.
- **PrebuiltUI** — https://prebuiltui.com/ — pre-built sections/blocks, general marketing + app UI.
- **ui-layouts** — https://www.ui-layouts.com/ — layout-focused component/section collection.

## Animation-forward / motion-heavy components
- **Magic UI** — https://magicui.design/ — animated components and effects (marquees, particles, text effects) built with Tailwind + Motion, shadcn-compatible.
- **Aceternity UI** — https://ui.aceternity.com/ — visually striking animated components (backgrounds, cards, hero effects) using Framer Motion/Tailwind.
- **Kokonut UI** — https://kokonutui.com/ — animated, modern-styled components, shadcn-adjacent.
- **Motion Primitives** — https://motion-primitives.com/ — animation primitives/building blocks (built on Motion/Framer Motion) for composing custom motion rather than fixed components.
- **Fancy Components** — https://www.fancycomponents.dev/ — animated/interactive effect components (text, cursor, hover effects).
- **Ruixen** — https://ruixen.com/ — animated + styled component collection, shadcn-adjacent.
- **React Bits** — https://reactbits.dev/ — animated React components/effects (text animations, backgrounds, interactive bits), similar niche to Fancy Components/Magic UI.
- **Anime.js** — https://animejs.com/ — standalone JS animation *engine*, not a component library. Use when a component needs a custom animation timeline/sequence beyond what CSS or Motion primitives give you; you write the animation logic against its API rather than copying a pre-made component.

## Data visualization / charts
- **visx** — https://visx.airbnb.tech/ — Airbnb's low-level D3-based React viz primitives. Use for custom/complex charts where you need fine control.
- **Reaviz** — https://reaviz.dev/ — higher-level ready-made React chart components (bar, line, pie, etc). Use when a standard chart type will do and you don't want to hand-build it with visx.
- **Liveline** — https://github.com/costcurve/liveline (by Benji Taylor, demo: https://benji.org/liveline) — real-time animated line/multi-series/candlestick charts for React, canvas-rendered at 60fps with zero CSS imports. Reach for this specifically for live/streaming data (price tickers, real-time metrics) rather than static charts — install via npm (`liveline`), not something to scrape from the demo page.

## Standalone effect libraries (GitHub, install via npm — not browsable component sites)
These are real installable npm packages with source on GitHub, not visual component catalogs like the rest of this file — `web_fetch` their GitHub README for usage, then `npm install` the package rather than copy-pasting source.
- **Fancy** — https://github.com/danielpetho/fancy — source repo behind fancycomponents.dev (listed above under Animation-forward). Check here for the latest source/examples if the live site's docs are thin on a specific component.
- **Jakubantalik's effect libraries** — a small family of focused, single-purpose animated React effect packages, each its own npm install:
  - **Libraries** (`github.com/Jakubantalik/Libraries`) — collection of effects: Border Beam, Liquid Gooey, Thinking Orbs, and more (also individually published, see below).
  - **border-beam** (`npm install border-beam`) — animated traveling/breathing glow border around any element (cards, buttons, inputs).
  - **metal-fx** (`npm install metal-fx`) — animated WebGL "liquid metal" ring effect for buttons/chips/icons, with optional proximity reflection on neighboring elements.
  - **img-fx** (`npm install img-fx`) — animated WebGL shader-driven image reveal/loading mosaic effect.
  - **thinking-orbs** — dotted thought-orb loading indicators, tuned for AI/agent UI loading states specifically.
  - **transitions.dev** (`github.com/Jakubantalik/transitions.dev`) — collection of copy-ready CSS-only interaction transitions (card resize, number pop-in, modal open/close, etc.) with a `prefers-reduced-motion` guard built into every snippet; also installable as its own agent skill (`npx skills add Jakubantalik/transitions.dev`) if the user wants it as a separate skill rather than pulled through this one.

## Meta-directories (not a library themselves — use to discover more)
- **Libraries.dev** — https://github.com/Jakubantalik/Libraries.dev — a directory/index of UI libraries rather than a component source itself. Useful if none of the libraries above fit and a broader search is needed, but don't treat entries found there as pre-vetted — apply the same scrutiny (check stack, deps, license) as any newly discovered library before using.

## Design systems, templates & assets
- **Aura** — https://www.aura.build/design-systems — broader than a component library: full design systems, page templates, components, and image/asset packs. Check here when the ask is closer to "give this a whole cohesive look" or "I need a template to start from" rather than a single component.

## Icons
- **Reicon** — https://reicon.dev/ — free icon set. Reach for this whenever a component needs icons and the user hasn't specified an icon library (e.g. Lucide) already in use in their project — check their existing imports first so you're not introducing a second icon system.
- **Icons8** — https://icons8.com/icons — large general-purpose icon library across multiple styles (outline, filled, color, 3D, etc.). Good fallback when Reicon or the project's existing icon set doesn't have a specific icon needed, or when a particular visual style (e.g. flat color icons, isometric) is wanted. Note: Icons8 has both free and paid tiers/licensing depending on icon style and usage — check the specific icon's license terms before using in a commercial project, unlike Reicon which is straightforwardly free.

## Free stock media (images & video)
- **Unsplash** — https://unsplash.com/ — free stock photos, generally the best default for polished/editorial-looking photography.
- **Pexels** — https://www.pexels.com/ — free stock photos and videos.
- **Pixabay** — https://pixabay.com/ — free stock photos, videos, and illustrations/vectors.
- **Mixkit** — https://mixkit.co/ — free stock video clips, music, and sound effects — go here specifically for video backgrounds/hero clips.
- Use these whenever a component needs a placeholder or final image/video and the user hasn't supplied their own — don't invent a fake image URL from memory.

## Community marketplace
- **21st.dev** — component marketplace, huge variety, crowd-sourced. Treat it as a normal part of the pool, not a fallback: check `references/21st-dev.md` (the user's collected components) the same way you'd check any other library above, before and alongside the others. The only difference from every other library here is mechanical — never `web_fetch` 21st.dev's site directly (it doesn't expose usable component source that way, and the user has a 2-fetch/day cap on manually pulling new ones there). If nothing in the stash fits, just say so and either use a different library or note that a 21st.dev pull could be added to the stash later — don't treat it as a dead end or make a big deal of the limitation.

## Picking quickly
- Need a marketing page section fast → shadcnblocks, Tailark, PrebuiltUI
- Need something to feel alive/animated → Magic UI, Aceternity, Motion Primitives, Fancy Components, React Bits
- Need a solid accessible form/dialog/table → shadcn/ui, Base UI, JollyUI
- Need a chart → Reaviz (quick) or visx (custom)
- Need a live/streaming chart → Liveline
- Need a retro/pixel theme → 8bitcn
- Need custom animation logic (not a packaged component) → Anime.js
- Need a whole template/design system, not just one piece → Aura
- Need icons → Reicon first, Icons8 as a broader fallback (check license per icon)
- Need real photos/video, not placeholders → Unsplash, Pexels, Pixabay, Mixkit
- Need a focused installable effect (border glow, liquid metal, image reveal, AI loading indicator) → Jakubantalik's effect packages (border-beam, metal-fx, img-fx, thinking-orbs)
