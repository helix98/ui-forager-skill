---
name: ui-forager
description: Pulls and adapts UI components from a curated set of free component/design libraries (shadcn/ui, Aceternity, Magic UI, Kokonut UI, Base UI, Origin UI, ReUI, Kibo UI, Tailark, visx, Ruixen, shadcnblocks, Motion Primitives, React Bits, Anime.js, and more), plus icon sources, free stock photo/video sites, and a personal stash of 21st.dev components — see references/libraries.md and references/21st-dev.md for full lists. Use whenever the user asks to build, improve, redesign, or add a UI component, section, page, animation, chart, icon, or image/video asset, or says "make this look better" — even without naming a library. Also use when a library is named explicitly or the user wants to browse/compare options. Don't write a component from memory when a fitting one likely exists here — check first. Covers picking the right source, fetching real code via web_fetch, and adapting it to the user's project instead of pasting it in unchanged.
---

# UI Forager

Finds real, current component code from a curated set of free UI libraries, then adapts (not just copies) it into the user's project. The point is to stop reinventing buttons/cards/heroes from memory — go get a real implementation, then reshape it.

## Workflow

1. **Classify the need.** What kind of thing is being requested — layout block (hero, pricing, nav), interactive primitive (dialog, combobox, form), animation/motion effect, chart/data-viz, full-page template/design system, icon, or a stock photo/video asset? Check `references/libraries.md` (and `references/21st-dev.md`, the user's personal component stash) to pick 1-3 candidate sources whose category and style fit. Don't default to shadcn/ui for everything — e.g. reach for Anime.js or Motion Primitives for animation, visx or Reaviz for charts, shadcnblocks/Tailark for marketing sections, Aura for a whole template/design system, Reicon for icons, Unsplash/Pexels/Pixabay/Mixkit for real images/video.

2. **Detect the user's actual stack first.** Look at the project (package.json, existing components, CSS setup) to see what framework, styling approach (Tailwind version, CSS modules, etc.), and component conventions (e.g. shadcn's `cn()` + `class-variance-authority` pattern) are already in use. This determines how much adaptation a fetched component needs. If nothing exists yet, ask or use whatever the user says.

3. **Fetch real code, don't guess.** Use `web_fetch` on the specific library's docs/component page (see per-library notes in the reference file for URL patterns — most expose a `/docs/components/<name>` or similar route with copy-paste code blocks). Do not write the component from memory/training data pretending it's from that library — actually fetch the current source. If a docs page doesn't have the code inline, `web_fetch` the linked source file (e.g. a raw GitHub URL if shown). A few entries in `references/libraries.md` (the "Standalone effect libraries" section) are real installable npm packages rather than copy-paste component sites — for those, fetch the GitHub README for usage/props, then `npm install` the package instead of copying source out of the repo.

4. **Adapt, don't just paste.**
   - Rewrite class names / tokens to match the user's existing design system (colors, spacing scale, border radii, font) rather than importing the library's defaults verbatim.
   - Strip dependencies the user doesn't already have unless the component needs them — check package.json before adding a new package, and tell the user what you're adding and why.
   - Convert between frameworks if needed (e.g. an Aceternity/Next.js component into a plain Vite+React or a different meta-framework) — fix imports, remove Next-specific APIs (`next/image`, `next/link`, `"use client"` if irrelevant), etc.
   - If a component is TypeScript and the project is JS (or vice versa), convert it.
   - Feel free to combine pieces from more than one library (e.g. Aceternity's animated background behind a shadcn/ui form) — that's the point of having many sources.

5. **Explain the swap.** Briefly tell the user which library/component you pulled from and what you changed, so they can go look at the original if they want. Don't silently attribute a heavily-modified component to the source library without noting the changes.

6. **Flag runtime CDN script loading before using it.** A few stashed/fetched components load a third-party library at runtime from a CDN instead of a normal npm install + import — either via `document.createElement('script')` injected into the parent page, or via a `<script src="cdn...">` tag baked into an iframe's embedded HTML (e.g. the Tailwind Play CDN inside an isolated "study" component). Treat both as a supply-chain/runtime-dependency consideration worth surfacing, not a neutral style choice — mention it explicitly, note whether it's scoped to a sandboxed iframe (lower risk to the parent app) or the main page (higher risk), and suggest a normal npm install instead when a real equivalent exists.

7. **Pull only what's needed from multi-component bundles.** Some stashed/fetched files export several independent pieces at once (e.g. a whole auth-page toolkit exporting an input, a reveal animation, a form, decorative orbit graphics, etc). Don't dump the entire file in when only one piece is wanted — extract just the needed export(s) and their direct dependencies, and say what was left out.

8. **Flag real brand logos before reusing them.** Some demo content (e.g. "trusted by" logo clouds) hardcodes real, trademarked company logos as placeholder content. Don't carry those into a real project as-is implying a partnership/customer relationship that may not exist — flag it and suggest either the user's actual partner logos, a neutral icon set, or confirming they have rights to display the specific logos shown.

## Special cases

- **21st.dev**: Part of the normal pool, not a fallback — check `references/21st-dev.md` (the user's personal stash of manually-collected 21st.dev components) alongside `references/libraries.md` when picking candidates, same as any other library. The only thing that's different about it mechanically: never `web_fetch` the 21st.dev site directly — it doesn't expose usable source that way, and the user has a 2-fetch/day cap on manually pulling new ones. If nothing in the stash fits, just move on to another library; don't dwell on it or frame it as a dead end. When the user pastes a new component and says it's from 21st.dev, append it to `references/21st-dev.md` using the existing entries as the format template (name, category, stack, deps, short summary/notes, then the full code block).
- **Network note**: `bash_tool`'s network is restricted to package registries (npm, pypi, GitHub, etc.) — it can't reach these design sites directly. All fetching of docs/component pages must go through the `web_fetch` tool, not `curl`/`wget` in bash. Installing an npm dependency the component needs (e.g. `framer-motion`, `motion`, `clsx`) is fine via bash/npm once you know it's needed.
- **When nothing fits well**: if none of the libraries have a good match for something very custom, say so and build it plain rather than forcing a bad-fit component in.
- **Don't over-fetch**: pick the 1-2 most likely candidates from the reference file first; only browse further if the first attempt is a poor fit or the user asks to compare options.

## Reference

See `references/libraries.md` for the full categorized list of component libraries, design systems, icon sources, and free stock media sites, with URLs and what each is best for — read it before picking a source.

See `references/21st-dev.md` for the user's personally collected 21st.dev components — check it as a normal part of source selection, not a special case, whenever a request might match something there.
