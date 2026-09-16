# UI Library Reference

Pick 1-3 libraries that match the need before fetching. Most sites expose copy-paste code on a `/docs/...` or `/components/...` page — `web_fetch` that page directly. If a page is React-heavy and the code doesn't render in the fetched text, try appending `/code` or look for a "Copy code" / raw GitHub link on the page.

## Selection rubric

Prefer sources in this order:
- Existing project conventions and dependencies already present in `package.json`, imports, CSS, and component folders.
- The smallest ownership model that solves the job: copy-owned snippet, shadcn registry item, headless primitive, focused npm package, then full visual system only when the surface is broad enough.
- Accessible primitives for behavior-heavy UI; visual block libraries for static/marketing sections; domain engines for complex tables, editors, charts, node canvases, AI chat, and emails.
- Sources with clear docs, real code access, compatible licensing, maintained packages/registries, and no unnecessary runtime CDN loading.

For broad UI asks, compare one stack-fit option, one specialist/best-tool option, and one visual-polish or inspiration option before choosing. Use the compatibility references below when package size, browser support, web platform behavior, or email-client support can affect the result.

Avoid by default:
- Adding a full visual system for one small component.
- Mixing shadcn, HeroUI, Mantine, Radix Themes, daisyUI, Flowbite, and similar systems unless the user wants that layer or the app has no established system yet.
- Using a heavy engine for a simple static surface: AG Grid for a basic table, React Flow for a non-editable diagram, BlockNote/Plate for a plain textarea, or Anime.js/GSAP for a CSS-only transition.
- Carrying demo brand logos, trademarked app icons, pro-only blocks, or third-party CDN scripts into production without flagging them.

## Core component systems (primitives, forms, general UI)
- **shadcn/ui** — https://ui.shadcn.com/ — the baseline: unstyled-ish, accessible Radix-based components you own the code for. Default starting point for buttons, dialogs, forms, tables, nav. Most other libraries here assume/extend shadcn conventions (`cn()`, CVA, Tailwind).
- **Base UI** — https://base-ui.com/ — unstyled, accessible primitives (from the Radix/Floating UI team). Use when you want full styling control with zero visual opinion.
- **Ark UI** — https://ark-ui.com/ — headless accessible components built on Zag.js state machines, with React, Solid, Vue, and Svelte packages. Use when the project is non-React, multi-framework, or needs complex widgets (combobox, date picker, color picker, carousel) with behavior handled but styling fully owned.
- **Ariakit** — https://ariakit.org/ — unstyled React primitives and hooks with strong accessibility coverage and lots of real examples. Good for comboboxes, menus, dialogs, command menus, and search/filter UIs when Radix/shadcn is not already the local pattern.
- **Headless UI** — https://headlessui.com/ — Tailwind Labs' unstyled accessible primitives for React and Vue. Best when the project already uses Tailwind but not shadcn/Radix, or when Vue support matters.
- **React Aria Components** — https://react-aria.adobe.com/ — Adobe's unstyled accessible React components and hooks with built-in behavior, accessibility, internationalization, and examples for CSS/Tailwind/shadcn. Use when a React app needs serious custom-styled primitives (combobox, date picker, color picker, collection/table patterns, drop zones) and the local design system should own the look.
- **Radix Themes** — https://www.radix-ui.com/themes — styled React component kit from Radix. Use when a React app wants a cohesive packaged design system quickly rather than copy-owned shadcn source; avoid mixing with an established shadcn theme for one-off widgets because it brings its own theme provider/tokens.
- **HeroUI** — https://heroui.com/ — polished React component library built on Tailwind CSS v4 and React Aria Components (formerly NextUI). Use when an app wants a living npm dependency with attractive defaults, accessibility, and full component coverage rather than copy-paste ownership; avoid for a single component inside an app that already owns its shadcn primitives.
- **Mantine** — https://mantine.dev/ — large, mature React component and hooks system (not Tailwind-first). Strong for internal tools, admin apps, forms, notifications, modals, spotlight/command center, dropzone, carousel, and rich app workflows where adding Mantine's styling/runtime is acceptable; avoid as a casual add-on to Tailwind/shadcn projects unless the feature surface is large enough to justify it.
- **Origin UI** — https://www.originui-ng.com/ — large set of styled, ready-to-use shadcn-style components (inputs, pickers, nav) with more visual polish out of the box than raw shadcn.
- **OriginKit** — https://www.originkit.dev/ — shadcn-style component collection (distinct from Origin UI above — check both, they're separate sites).
- **Hextaui** — https://www.hextaui.com/ — shadcn-style component collection, general purpose UI.
- **ReUI** — https://reui.io/ — shadcn-compatible components, general UI + some data/table pieces.
- **Kibo UI** — https://www.kibo-ui.com/ — shadcn-based, leans toward more app/dashboard-oriented composite components (editors, kanban, etc).
- **JollyUI** — https://www.jollyui.dev/ — React Aria + Tailwind components, strong accessibility focus.
- **Park UI** — https://park-ui.com/ — source-distributed styled components built on Ark UI and Panda CSS, with multi-framework support. Good if the project already uses Panda CSS, or when you want Ark behavior plus a coherent visual system without inventing recipes from scratch.
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

## Behavior engines / UI plumbing
Use these when the hard part is interaction behavior rather than visual styling. They are installable libraries/engines, not copy-paste component galleries; fetch docs for usage and adapt the rendered markup/styles to the local project.

- **Floating UI** — https://floating-ui.com/docs/getting-started — positioning and interaction primitives for tooltips, popovers, dropdowns, menus, comboboxes, and other anchored floating elements. Use when Radix/Base/shadcn does not cover the custom behavior or when collision handling/portals/focus management need to be precise.
- **cmdk** — https://cmdk.paco.me/ — fast, unstyled command menu / accessible combobox component for React. Use for command palettes, searchable action menus, launcher UI, and app-wide quick-switchers; style it locally or use the existing shadcn command wrapper if already present.
- **Embla Carousel** — https://www.embla-carousel.com/docs/get-started/react — lightweight, dependency-free carousel engine with React/Vue/Svelte/Solid wrappers and plugins. Use for custom carousels/sliders where the visuals should be owned locally; avoid hand-rolling drag/scroll/snap behavior.
- **dnd kit** — https://dndkit.com/react/components/drag-drop-provider/ — modern drag-and-drop toolkit for React with providers, hooks, sensors, modifiers, sortable helpers, and multi-framework docs. Use for sortable lists, kanban boards, builders, upload ordering, and draggable app surfaces instead of writing pointer math from scratch.
- **TanStack Virtual** — https://tanstack.com/virtual/latest/docs/introduction — headless virtualization utility for huge lists, grids, logs, chats, and tables across React/Vue/Svelte/Solid/Lit/Angular. Use when the UI needs to render thousands of rows/items while keeping markup and styling local.
- **React DayPicker** — https://react-day-picker.js.org/ — customizable React date picker/calendar component with range selection, localization, keyboard navigation, WAI-ARIA support, and styling control. Use when a full app calendar is too much but date/range picking needs to be correct.
- **Sonner** — https://sonner.emilkowal.ski/ — opinionated React toast component. Use for polished notifications and promise/action toasts, especially in shadcn-style React apps where the existing UI already expects Sonner.
- **Vaul** — https://vaul.emilkowal.ski/ — drawer component for React. Use for mobile-friendly drawers/bottom sheets when Radix Dialog alone is not enough; style and motion should still match the app.

## Forms and validation workflows
Use these when the form behavior, validation flow, server submission model, or error handling is the hard part. They are not visual component libraries; pair them with the app's existing UI components.

- **Conform** — https://conform.guide/ — form validation library focused on progressive enhancement, accessibility, server validation, and native form behavior for React/Remix/Next-style apps. Use when the UI needs robust form state, field metadata, constraint validation, server-action workflows, or schema-backed errors without turning the form into a purely client-side widget.

## Accessibility and quality references
Use these as decision/check references when adapting components, especially for custom dialogs, menus, comboboxes, tabs, trees, grids, date pickers, form errors, keyboard flows, and color choices.

- **WAI-ARIA Authoring Practices Guide (APG)** — https://www.w3.org/WAI/ARIA/apg/ — official W3C patterns for accessible widgets, including roles, keyboard interaction, focus behavior, and examples. Use when implementing or reviewing custom interactive components instead of guessing ARIA behavior.
- **WCAG 2.2 Quick Reference** — https://www.w3.org/WAI/WCAG22/quickref/ — official W3C reference for accessibility success criteria and techniques. Use for broader UI quality checks around contrast, focus visibility, motion, input errors, target size, text alternatives, and predictable interaction.
- **MDN Web Docs** — https://developer.mozilla.org/en-US/ — canonical reference for HTML, CSS, JavaScript, Web APIs, semantic markup, layout behavior, accessibility basics, browser APIs, and examples. Use when fetched component code relies on a web-platform feature or CSS behavior that should be verified instead of guessed.
- **Can I use** — https://caniuse.com/ — browser support tables for HTML, CSS, SVG, web APIs, image formats, viewport units, and newer platform features. Use before adopting newer CSS/HTML/Web API features, especially in landing pages, mobile-heavy UI, embedded WebViews, or broad-browser products.
- **bundlejs** — https://bundlejs.com/ — quick npm package size/bundle analyzer. Use before adding a new runtime dependency for a component, chart, animation, editor, 3D scene, or media player when package weight could matter.

## Marketing pages / landing sections / blocks
- **shadcnblocks** — https://www.shadcnblocks.com/ — pre-built marketing sections (heroes, pricing, testimonials, footers) built on shadcn. Good first stop for landing page sections.
- **Tailark** — https://tailark.com/ — Tailwind marketing blocks/sections, framework-agnostic-ish Tailwind markup.
- **PrebuiltUI** — https://prebuiltui.com/ — pre-built sections/blocks, general marketing + app UI.
- **ui-layouts** — https://www.ui-layouts.com/ — layout-focused component/section collection.
- **HyperUI** — https://hyperui.dev/ — free MIT Tailwind CSS v4 copy-paste components for marketing sites, application UIs, ecommerce, and neobrutalist variants. No npm package; copy markup and adapt it.
- **Preline UI Blocks** — https://www.preline.co/blocks/ — large Tailwind block collection for marketing pages, dashboards, forms, menus, ecommerce, and templates. Some blocks are free and some are Pro; verify the exact block's availability before using.

## Tailwind copy-paste / framework-agnostic UI
- **Preline UI** — https://www.preline.co/ — open-source Tailwind CSS component library with HTML/JS/TypeScript snippets, headless plugins, blocks, templates, themes, and Figma resources. Good for non-React Tailwind projects or plain HTML prototypes; note the `preline` JS plugin and `@tailwindcss/forms` setup for interactive/form components.
- **Flowbite** — https://flowbite.com/ — broad Tailwind CSS ecosystem with components, blocks, icons, illustrations, Figma resources, and framework ports (React, Vue, Svelte, Angular, etc.). Useful for quick conventional app/marketing UI and non-shadcn stacks; prefer npm/bundler setup over CDN for production and avoid mixing its component styling into a mature shadcn system unless deliberately restyling.
- **daisyUI** — https://daisyui.com/ — Tailwind plugin that adds semantic component classes and built-in themes. Use for fast prototypes, plain HTML, and projects that want readable class names (`btn`, `card`, `modal`) instead of long utility strings. Avoid dropping it into an existing shadcn design system unless the user explicitly wants a daisyUI theme layer.

## Animation-forward / motion-heavy components
- **Magic UI** — https://magicui.design/ — animated components and effects (marquees, particles, text effects) built with Tailwind + Motion, shadcn-compatible.
- **Aceternity UI** — https://ui.aceternity.com/ — visually striking animated components (backgrounds, cards, hero effects) using Framer Motion/Tailwind.
- **Kokonut UI** — https://kokonutui.com/ — animated, modern-styled components, shadcn-adjacent.
- **Cult UI** — https://www.cult-ui.com/ — niche, motion-rich components for shadcn/ui projects, installable through the shadcn registry or copied manually. Good for texture cards/buttons, text effects, animated CTAs, and unusual hero polish; expects React, TypeScript, Tailwind v4, `cn()`, and often `motion`.
- **Animata** — https://www.animata.design/ — free MIT animated React components with categories for backgrounds, buttons, cards, charts, heroes, overlays, progress, scroll, skeletons, tabs, text, and widgets. Copy-paste or install via shadcn registry URLs (`https://animata.design/r/{category}/{name}.json`).
- **Animate UI** — https://animate-ui.com/ — open component distribution of Motion-powered React/Tailwind components, animated primitives, and animated Lucide icons using the shadcn registry model. Good when you want tasteful animation on standard primitives rather than a whole flashy section.
- **Skiper UI** — https://skiper-ui.com/ — uncommon shadcn components and interaction studies (dynamic island, progressive blur, perspective carousel, SVG follow scroll, animated icons, video player, scroll progress). Treat as a source for specialty interactions; inspect each registry item because free/pro split and dependencies vary.
- **Motion for React** — https://motion.dev/docs/react — React animation library formerly known through Framer Motion, with declarative animations, layout animation, gestures, scroll-linked motion, variants, and hooks. Use as the default engine for most custom React UI motion before reaching for heavier timeline tools.
- **Motion Primitives** — https://motion-primitives.com/ — animation primitives/building blocks (built on Motion/Framer Motion) for composing custom motion rather than fixed components.
- **Fancy Components** — https://www.fancycomponents.dev/ — animated/interactive effect components (text, cursor, hover effects).
- **Ruixen** — https://ruixen.com/ — animated + styled component collection, shadcn-adjacent.
- **React Bits** — https://reactbits.dev/ — animated React components/effects (text animations, backgrounds, interactive bits), similar niche to Fancy Components/Magic UI.
- **Anime.js** — https://animejs.com/ — standalone JS animation *engine*, not a component library. Use when a component needs a custom animation timeline/sequence beyond what CSS or Motion primitives give you; you write the animation logic against its API rather than copying a pre-made component.
- **GSAP** — https://gsap.com/docs/v3/ — professional-grade JavaScript animation engine for precise timelines, complex sequencing, SVG/canvas animation, and performance-sensitive choreography. Use for advanced animation systems where CSS/Motion is not enough; avoid for simple fades, hovers, and one-off transitions.
- **@gsap/react** — https://gsap.com/resources/React/ — official React integration for GSAP with `useGSAP()` and cleanup/context handling. Use this when adding GSAP to React apps instead of wiring effects manually.
- **GSAP ScrollTrigger** — https://gsap.com/docs/v3/Plugins/ScrollTrigger/ — GSAP plugin for scroll-driven timelines, pinned sections, scrubbed animation, scroll snapping, callbacks, and scrollytelling. Use for heavily choreographed scroll scenes; keep accessibility and reduced-motion fallbacks in mind.

## Data visualization / charts
- **visx** — https://visx.airbnb.tech/ — Airbnb's low-level D3-based React viz primitives. Use for custom/complex charts where you need fine control.
- **Reaviz** — https://reaviz.dev/ — higher-level ready-made React chart components (bar, line, pie, etc). Use when a standard chart type will do and you don't want to hand-build it with visx.
- **Recharts** — https://recharts.org/en-US/ — composable React chart library built on React components and D3 submodules. Use as a practical default for standard React charts, dashboard cards, and shadcn-style chart patterns before reaching for heavier or lower-level libraries.
- **Tremor** — https://npm.tremor.so/ — React/Tailwind components for charts, KPIs, lists, tables, filters, date pickers, and dashboards. Good for product analytics/admin surfaces where the user needs polished business charts fast; available as npm package (`@tremor/react`) and as Tremor Raw copy-paste components.
- **Nivo** — https://nivo.rocks/ — rich React dataviz components built on D3, with SVG/Canvas/API variants for many chart types (calendar, radar, treemap, sankey, choropleth, swarm, etc.). Use when Reaviz is too basic but full custom visx work is overkill.
- **Apache ECharts** — https://echarts.apache.org/en/index.html — mature, Apache-licensed charting/visualization engine with many chart types, large-data handling, interactions, maps, dashboards, and framework wrappers. Use for complex dashboards or dense interactive visualizations where Recharts/Tremor are too limited and full custom visx work is not worth it.
- **Liveline** — https://github.com/costcurve/liveline (by Benji Taylor, demo: https://benji.org/liveline) — real-time animated line/multi-series/candlestick charts for React, canvas-rendered at 60fps with zero CSS imports. Reach for this specifically for live/streaming data (price tickers, real-time metrics) rather than static charts — install via npm (`liveline`), not something to scrape from the demo page.

## Tables, grids, node editors, and rich app surfaces
- **TanStack Table** — https://tanstack.com/table/ — headless table engine for React, Vue, Solid, Svelte, and more. Use for custom data tables needing sorting, filtering, pagination, grouping, virtualization, or server-driven state while keeping all markup/styling local.
- **AG Grid Community** — https://www.ag-grid.com/react-data-grid/ — high-performance React data grid with a large feature set and a free Community edition plus paid Enterprise features. Use for spreadsheet-like internal tools, huge datasets, column management, editing, and enterprise-grade tables; avoid for simple display tables and check which desired features are Community vs Enterprise.
- **React Flow** — https://reactflow.dev/ — MIT React library for node-based editors, diagrams, workflow builders, automation canvases, and visual programming surfaces. Use instead of hand-rolling pan/zoom/drag/connect logic.
- **BlockNote** — https://www.blocknotejs.org/ — block-based React rich text editor with polished Notion-like UX out of the box. Best when the product needs a modern content editor quickly and a block document model is acceptable; avoid when a plain textarea, Markdown box, or simple WYSIWYG is enough.
- **Plate** — https://platejs.org/ — rich-text editor framework with headless plugins plus shadcn-based Plate UI components. Use when the editor must be deeply customized, plugin-heavy, collaborative/AI-enhanced, or integrated into an existing shadcn design system; avoid for lightweight notes fields.

## Maps, uploads, and domain-specific app UI
- **mapcn** — https://www.mapcn.dev/docs (registry source: https://github.com/AnmolSaini16/mapcn/blob/main/registry.json) — copy-paste React map components built on MapLibre GL, Tailwind CSS, and shadcn/ui patterns. Use for interactive maps with markers, popups, routes, arcs, GeoJSON, clusters, and controls; prefer this over hand-wrapping MapLibre for ordinary product maps.
- **Better Upload** — https://better-upload.com/docs — React upload library for direct uploads to S3-compatible storage, with copy-paste shadcn/ui components. Use when the app needs real upload behavior, not just a visual dropzone; note the storage/backend setup before treating it as a pure UI component.

## Media, time-based, and motion asset UI
Use these when the UI is driven by media, time ranges, audio waveforms, or exported motion assets. These are runtime libraries and app surfaces, not decorative defaults: check package weight, asset source/rights, accessibility, reduced-motion needs, and whether assets should be self-hosted instead of loaded from vendor URLs.

- **Vidstack Player** — https://vidstack.io/docs/player/ — framework plus UI components for custom video/audio players with React and Web Component support, media state, controls, captions, streaming/HLS, keyboard access, gestures, and production-ready layouts. Use for serious media players instead of stitching native video controls together.
- **FullCalendar** — https://fullcalendar.io/docs/react — calendar/scheduling UI engine with a React component, date grids, event rendering, and broad plugin support. Use for scheduling apps, booking calendars, availability views, and event-heavy dashboards; verify whether any Scheduler/resource feature needed is part of the free stack before committing to it.
- **wavesurfer.js** — https://wavesurfer.xyz/docs/ — interactive audio waveform renderer/player with seeking, playback, Shadow DOM styling isolation, and plugins for regions, timeline, spectrogram, minimap, recording, and zoom. Use for podcast/audio editors, voice-note review, call playback, music tools, or transcript-aligned audio UI.
- **dotLottie React** — https://docs.lottiefiles.com/en/runtimes/distributions/react — official LottieFiles React player for `.lottie` and `.json` animations, wrapping the dotLottie web player with props, refs, instance access, and event callbacks. Prefer this for modern LottieFiles/dotLottie assets, especially when `.lottie` packaging or player events matter.
- **lottie-react** — https://lottiereact.com/ — lightweight React wrapper around Lottie/lottie-web with a component and hook API. Use for plain `.json` Lottie assets in React when `.lottie` packaging is not needed and the app wants simple SSR-friendly embeds plus direct playback control.
- **lottie-web** — https://github.com/airbnb/lottie-web — Airbnb's base web player for Bodymovin-exported After Effects JSON animations. Use directly for framework-agnostic or legacy Lottie integrations; in React apps, prefer `dotLottie React` or `lottie-react` unless direct renderer/instance control is the point.
- **LottieFiles asset library** — https://help.lottiefiles.com/discovering-downloading-and-uploading-animations — marketplace/library for finding downloadable `.lottie` and Lottie JSON animation assets. Use as an asset source, not a runtime; confirm whether the selected animation is suitable for the product and download/self-host when reliability matters.
- **LottieFiles licensing guide** — https://help.lottiefiles.com/animation-licensing-basics- — licensing reference for LottieFiles assets. Check this before using marketplace animations commercially, modifying them, or carrying them into client/product work.
- **Rive React** — https://rive.app/docs/runtimes/react/react — official Rive runtime for rendering `.riv` files in React, including hooks for state machines, events, and data binding. Use for interactive vector animations, animated product UI, game-like controls, onboarding characters, progress meters, and animations that need stateful runtime input.
- **Rive Marketplace** — https://rive.app/docs/community/marketplace-overview — official place to discover and remix community Rive files. Use as a source for `.riv` assets and interaction ideas; check the file's stated license/attribution expectations before embedding it in a product.
- **Spline React** — https://github.com/splinetool/react-spline — React runtime for exported Spline scenes (`@splinetool/react-spline` + `@splinetool/runtime`). Use when the design asset is an interactive Spline scene or a page needs a real 3D object/scene; prefer self-hosting `.splinecode` when reliability, privacy, or CORS matters.

## 3D and WebGL UI
Use these when the UI needs a custom 3D/WebGL scene, not just an exported design asset. Prefer Spline for embedding an already-authored Spline scene; prefer three.js/React Three Fiber when the app needs programmatic control, generated geometry, live data, custom shaders, physics, or deep interaction.

- **three.js** — https://threejs.org/manual/en/installation.html — foundational WebGL/3D library for scenes, cameras, meshes, materials, lights, loaders, shaders, and render loops. Use for framework-agnostic or deeply custom browser 3D, and keep performance/mobile fallbacks in mind.
- **React Three Fiber** — https://r3f.docs.pmnd.rs/ — React renderer for three.js that lets React own the 3D scene graph. Use in React apps when building interactive 3D components, data scenes, product viewers, game-like UI, or WebGL backgrounds that need real code control.
- **drei** — https://github.com/pmndrs/drei — helper library for React Three Fiber with ready-made controls, loaders, cameras, text, HTML overlays, staging, environments, and utilities. Use to avoid re-implementing common three/R3F plumbing.

## AI/chat and email UI
- **AI Elements** — https://elements.ai-sdk.dev/ — Vercel's shadcn-based components for AI-native apps: conversations, messages, prompt inputs, code blocks, reasoning panels, tool displays, sources, and response actions. Use when the backend uses Vercel AI SDK or the UI needs streaming/tool-call patterns.
- **assistant-ui** — https://www.assistant-ui.com/ — React primitives and shadcn registry components for AI chat threads, thread lists, attachments, markdown, reasoning, tools, follow-up suggestions, and adapters for AI SDK/LangGraph/custom runtimes. Use for fuller assistant apps and agent workbenches rather than one-off chat bubbles.
- **LocalMode UI** — https://localmode.ai/ (docs: https://localmode.dev/docs/ui) — shadcn registry of copy-owned React components for local-first/browser-native AI: chat, RAG, vision, audio, agent steps, model download panels, browser capability gates, and privacy-oriented states. Use when the product is explicitly local/offline AI, browser inference, or privacy-sensitive AI UX; for ordinary cloud chat, prefer AI Elements or assistant-ui first.
- **React Email** — https://react.email/ — React/TypeScript components and templates for production HTML email. Use when building email templates, transactional emails, or branded email previews; email-safe layout constraints are different from web UI, so don't adapt ordinary page components blindly.
- **Really Good Emails** — https://reallygoodemails.com/ — large gallery of real email examples across lifecycle, transactional, ecommerce, SaaS, onboarding, retention, and announcement patterns. Use for email design/content structure inspiration, then implement with React Email or the project's existing email stack; treat examples as references, not copy-owned assets.
- **Can I email** — https://www.caniemail.com/ — support tables for HTML and CSS across email clients such as Apple Mail, Gmail, Outlook, Yahoo, and others. Use before relying on CSS/layout features in production email templates; normal browser support does not imply email-client support.

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
- **shadcn Registry Directory** — https://ui.shadcn.com/docs/registry/registry-index and https://ui.shadcn.com/r/registries.json — official index of public shadcn-compatible registries, including health metadata. Use to discover installable registries when the curated list has no match; still review third-party code and dependencies before adding.
- **shadcnregistry** — https://shadcnregistry.com/ — searchable community directory of shadcn registries with item counts and categories. Useful for finding niche registries (AI UI, maps, loaders, dashboards, data grids), but treat it as discovery only, not endorsement.

## Legacy / avoid-by-default sources
- **LottieFiles lottie-interactivity** — https://github.com/LottieFiles/lottie-interactivity — archived on GitHub as of June 20, 2026, so don't add it to new projects by default. For scroll/cursor-driven Lottie in new React work, prefer `dotLottie React` or `lottie-react` plus local scroll/motion logic unless maintaining an existing implementation requires this package.

## Design systems, templates & assets
- **Aura** — https://www.aura.build/design-systems — broader than a component library: full design systems, page templates, components, and image/asset packs. Check here when the ask is closer to "give this a whole cohesive look" or "I need a template to start from" rather than a single component.

## Icons
- **Reicon** — https://reicon.dev/ — free icon set. Reach for this whenever a component needs icons and the user hasn't specified an icon library (e.g. Lucide) already in use in their project — check their existing imports first so you're not introducing a second icon system.
- **Icons8** — https://icons8.com/icons — large general-purpose icon library across multiple styles (outline, filled, color, 3D, etc.). Good fallback when Reicon or the project's existing icon set doesn't have a specific icon needed, or when a particular visual style (e.g. flat color icons, isometric) is wanted. Note: Icons8 has both free and paid tiers/licensing depending on icon style and usage — check the specific icon's license terms before using in a commercial project, unlike Reicon which is straightforwardly free.
- **lucide-animated** — https://lucide-animated.com/ — MIT animated Lucide icon registry for React/shadcn projects. Use when the project already uses Lucide and needs animated icon feedback for states, navigation, or micro-interactions; avoid introducing it just for static icons.

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
- Need a Tailwind block in plain HTML or a non-React app → HyperUI, Preline UI, Flowbite
- Need semantic Tailwind component classes/themes → daisyUI
- Need something to feel alive/animated → Magic UI, Aceternity, Cult UI, Animata, Animate UI, Skiper UI, Motion for React, Motion Primitives, Fancy Components, React Bits
- Need custom React animation logic → Motion for React first, GSAP for timeline-heavy choreography, Anime.js for standalone timeline work
- Need advanced scroll/pinned/scrubbed animation → GSAP ScrollTrigger with @gsap/react in React apps
- Need a solid accessible form/dialog/table → shadcn/ui, Base UI, Ark UI, Ariakit, Headless UI, React Aria Components, JollyUI
- Need precise popover/dropdown/tooltip positioning → Floating UI
- Need a command palette/search launcher → cmdk
- Need carousel behavior with local visuals → Embla Carousel
- Need sortable/drag-and-drop UI → dnd kit
- Need huge lists/grids/chat logs without rendering every item → TanStack Virtual
- Need a date picker/range picker → React DayPicker
- Need toast notifications or a drawer/bottom sheet → Sonner or Vaul
- Need real form validation/submission behavior → Conform
- Need accessibility/platform behavior checks → WAI-ARIA APG, WCAG Quick Reference, and MDN
- Need browser support confidence for CSS/HTML/Web APIs → Can I use
- Need package size/dependency weight check → bundlejs
- Need a polished packaged React component suite → HeroUI, Mantine, Radix Themes
- Need Panda CSS + multi-framework components → Park UI
- Need a chart → Recharts (default React), Reaviz (quick), Tremor (dashboard), Nivo (many chart types), Apache ECharts (dense/complex), or visx (custom)
- Need a live/streaming chart → Liveline
- Need a complex table/data grid → TanStack Table (headless/custom) or AG Grid Community (spreadsheet-grade)
- Need a workflow/diagram/node canvas → React Flow
- Need an interactive map → mapcn
- Need real file uploads, not just a dropzone visual → Better Upload
- Need a custom video/audio player → Vidstack Player
- Need scheduling/calendar UI → FullCalendar
- Need audio waveform playback/editing → wavesurfer.js
- Need Lottie/dotLottie animation assets → dotLottie React, lottie-react, or lottie-web
- Need actual Lottie files to use → LottieFiles asset library, then check LottieFiles licensing
- Need interactive/stateful vector animation → Rive React
- Need actual Rive files to use or remix → Rive Marketplace, then check license/attribution
- Need an interactive 3D scene from a design asset → Spline React
- Need custom/programmatic 3D UI → three.js, React Three Fiber, and drei
- Need rich text/document editing → BlockNote (fast Notion-like editor) or Plate (deeply customizable shadcn/plugin editor)
- Need AI chat / reasoning / tool-call UI → AI Elements (AI SDK) or assistant-ui (full assistant workbench)
- Need local/offline/browser-native AI UI → LocalMode UI
- Need email templates → React Email
- Need real email design examples → Really Good Emails
- Need email-client HTML/CSS support checks → Can I email
- Need a retro/pixel theme → 8bitcn
- Need custom animation logic (not a packaged component) → Anime.js
- Need a whole template/design system, not just one piece → Aura
- Need icons → Reicon first, Icons8 as a broader fallback (check license per icon)
- Need animated Lucide icons → lucide-animated
- Need real photos/video, not placeholders → Unsplash, Pexels, Pixabay, Mixkit
- Need a focused installable effect (border glow, liquid metal, image reveal, AI loading indicator) → Jakubantalik's effect packages (border-beam, metal-fx, img-fx, thinking-orbs)
