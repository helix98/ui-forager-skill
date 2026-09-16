<div align="center">
  <img src="./assets/ui-forager-banner.svg" alt="UI Forager: find real UI source, adapt it to your project, ship with confidence" width="900" />

  <p><strong>Find real UI source. Adapt it to the project. Keep the result yours.</strong></p>

  <p>
    <a href="https://github.com/helix98/ui-forager-skill/blob/main/LICENSE"><img src="https://img.shields.io/badge/license-MIT-74C6B2?style=flat-square" alt="MIT License" /></a>
    <a href="https://github.com/helix98/ui-forager-skill/blob/main/SKILL.md"><img src="https://img.shields.io/badge/format-SKILL.md-F1D45A?style=flat-square" alt="SKILL.md format" /></a>
    <a href="https://github.com/helix98/ui-forager-skill/commits/main"><img src="https://img.shields.io/github/last-commit/helix98/ui-forager-skill?style=flat-square&color=F26D5B" alt="Last commit" /></a>
  </p>
</div>

UI Forager is a Claude-compatible skill for finding current UI components, interaction engines, motion assets, data surfaces, email templates, and real media, then adapting them to the project that needs them.

It is designed for the moment when "make it look better" is too vague for a useful search, but writing another generic component from memory is not good enough either.

## The Short Version

```text
brief -> inspect the project -> compare sources -> fetch real code -> adapt -> verify
```

The skill checks the project's framework, existing design system, dependencies, licensing, runtime risk, browser or email support, accessibility, and visual fit before it chooses a source.

## What It Covers

| Need | Good starting points |
| --- | --- |
| Components and primitives | shadcn/ui, Base UI, Ark UI, Ariakit, Headless UI, React Aria |
| Hard interaction behavior | Floating UI, cmdk, Embla, dnd kit, TanStack Virtual, React DayPicker |
| Forms and app workflows | Conform, Better Upload, TanStack Table, AG Grid, React Flow |
| Motion and visual polish | Motion, GSAP, Anime.js, Magic UI, Aceternity, Animate UI |
| Charts and data surfaces | Recharts, Reaviz, Tremor, Nivo, Apache ECharts, visx |
| Media and time-based UI | Vidstack, FullCalendar, wavesurfer.js, dotLottie, Rive, Spline |
| 3D and WebGL | three.js, React Three Fiber, drei |
| Editors and AI surfaces | BlockNote, Plate, AI Elements, assistant-ui, LocalMode UI |
| Email and visual references | React Email, Really Good Emails, Can I Email |
| Images and video | Unsplash, Pexels, Pixabay, Mixkit |

The full directory, URLs, stack notes, dependency expectations, and legacy warnings live in [`references/libraries.md`](./references/libraries.md). The personal 21st.dev stash lives in [`references/21st-dev.md`](./references/21st-dev.md).

## Why It Is Different

- **Source over imitation:** fetch the current component or documented package instead of pretending a memory-based recreation came from a library.
- **Adaptation over pasting:** match the project's tokens and conventions, remove demo baggage, and keep only the files and dependencies the feature needs.
- **Fit over fashion:** choose the source for the job, visual direction, interaction complexity, and audience conditions, not because it is the familiar default.
- **A usable fallback:** account for keyboard use, focus, text zoom, reduced motion, forced colors, localization, image blocking, and slow devices when the surface depends on them.
- **A visible tradeoff:** explain what was chosen, what changed, what was rejected, and what still needs verification.

## Install

Clone the skill into the Claude skills directory:

```bash
git clone https://github.com/helix98/ui-forager-skill.git ~/.claude/skills/ui-forager
```

Windows PowerShell:

```powershell
git clone https://github.com/helix98/ui-forager-skill.git "$env:USERPROFILE\.claude\skills\ui-forager"
```

Restart the Claude Code, Claude Desktop, OpenCode, or oh-my-pi session after installing so it can discover the skill.

Update an existing checkout:

```bash
cd ~/.claude/skills/ui-forager
git pull
```

## Try It

```text
Use $ui-forager to build a checkout form that fits the existing app.
Use $ui-forager to compare three approaches for a product launch hero.
Use $ui-forager to find a responsive email template for a post-purchase message.
Use $ui-forager to add a chart, but keep it readable in high contrast and without relying on color alone.
```

The result should include the source used, the reason it fit, the adaptation made, the important dependency or license notes, and the remaining QA risks.

## Quality Bar

Before a source is carried into a project, check:

- Is the exact source current, documented, and appropriate for the project's stack?
- Is it free to use in this context, and are attribution or pro-only limits clear?
- Does it bring a reasonable dependency and runtime cost?
- Does it preserve semantic structure, keyboard access, focus, readable text, and useful fallbacks?
- Are demo logos, customer claims, stock assets, CDN scripts, and placeholder URLs removed or explicitly approved?
- Was the adapted result checked in the conditions that matter: mobile, browser support, email client, reduced motion, dark mode, slow data, or large content?

## Adding a Source

Read [`CONTRIBUTING.md`](./CONTRIBUTING.md) before opening a pull request. New entries are most useful when they include a real URL, what the source is best for, supported frameworks, ownership model, dependencies, license or free-tier notes, accessibility considerations, and one reason to choose it over an obvious alternative.

The repository also has an issue form for library requests so useful context is captured before the directory grows.

## Repository Map

```text
ui-forager/
├── SKILL.md                         # the operating instructions
├── references/
│   ├── libraries.md                 # curated sources and decision notes
│   └── 21st-dev.md                  # hand-collected component stash
├── assets/
│   └── ui-forager-banner.svg        # repository identity asset
├── CONTRIBUTING.md                  # source and contribution standards
└── CODE_OF_CONDUCT.md               # community expectations
```

## License

MIT. Individual libraries, assets, and examples listed in the directory may have their own licenses; verify the source's terms before shipping it.
