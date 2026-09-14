# ui-forager

A [Claude Skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) that pulls real, current UI component code from a curated set of free component/design libraries instead of writing UI from memory — then adapts it to fit your actual project instead of pasting it in unchanged.

Works with Claude Code, Claude Desktop, [OpenCode](https://opencode.ai), and [oh-my-pi](https://github.com/can1357/oh-my-pi) — anything that reads Claude-compatible skills from `~/.claude/skills/`.

## What it does

When you ask for a UI component, section, page, animation, chart, icon, or image/video asset — even without naming a library — this skill:

1. Classifies what you actually need (layout block, interactive primitive, animation, chart, template, icon, media asset)
2. Picks 1–3 candidate sources from `references/libraries.md`
3. Fetches **real, current source code** via `web_fetch` instead of hallucinating "shadcn-style" code from memory
4. Adapts it — matches your design tokens, strips unneeded deps, converts frameworks/TS↔JS, flags anything risky (CDN script injection, trademarked logos, missing dependencies) — instead of pasting it in unchanged
5. Tells you what it pulled from where

## Sources covered

**Component libraries:** shadcn/ui, Aceternity, Magic UI, Kokonut UI, Base UI, Origin UI, OriginKit, ReUI, Kibo UI, Hextaui, JollyUI, 8bitcn, Shadway, ShadcnSpace, Systaliko UI, UI Lora, Unlumen UI, SmoothUI, VengenceUI, and more

**Marketing blocks:** shadcnblocks, Tailark, PrebuiltUI, ui-layouts

**Animation-forward:** Motion Primitives, Fancy Components, Ruixen, React Bits, Anime.js

**Charts:** visx, Reaviz

**Design systems/templates:** Aura

**Icons:** Reicon

**Free stock media:** Unsplash, Pexels, Pixabay, Mixkit

**Personal stash:** `references/21st-dev.md` — a growing, hand-collected set of 21st.dev components (glassmorphism effects, WebGL image effects, pricing tables, hero sections, auth forms, etc.), each documented with its actual dependencies, gotchas, and full source — since 21st.dev doesn't expose fetchable component pages and has a daily pull limit.

See `references/libraries.md` for the full list with URLs and what each is best for.

## Install

Clone straight into your Claude skills directory:

```bash
git clone https://github.com/helix98/ui-forager-skill.git ~/.claude/skills/ui-forager
```

Windows (PowerShell):

```powershell
git clone https://github.com/helix98/ui-forager-skill.git "$env:USERPROFILE\.claude\skills\ui-forager"
```

Restart your Claude Code / OpenCode / oh-my-pi session and it'll pick it up automatically.

## Updating

```bash
cd ~/.claude/skills/ui-forager
git pull
```

## Adding to the 21st.dev stash

Since 21st.dev doesn't expose fetchable component source and has a 2-pull/day limit, new components get added by hand: paste a component's code into `references/21st-dev.md` following the existing entry format (name, category, stack, deps, summary, notes, full source), commit, and push.

## Structure

```
ui-forager/
├── SKILL.md                    # main skill instructions
└── references/
    ├── libraries.md            # categorized library list with URLs
    └── 21st-dev.md              # personal 21st.dev component stash
```

## License

MIT (or your preference — this is your personal tooling).
