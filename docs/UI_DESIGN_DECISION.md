# UI Design Decision: zen-sci-portal

**Date:** 2026-02-19
**Decision owner:** Supervisor (Cruz-delegated)
**Input:** DesignIntelligencePlatform/research/landscape-research-wide.md (2026-02-15)

---

## Decision Summary

**Question:** What UI design plugins should be made available for the zen-sci-portal project?

**Answer:**
1. **Install shadcn-svelte on the portal** (Svelte 5 component library — not a Claude plugin, a pnpm package)
2. **No new Claude plugins needed right now** — Context7 is already installed and provides up-to-date Svelte 5, shadcn-svelte, and Tailwind docs on demand
3. **Figma MCP is worth adding later** if Cruz wants design files before build; not required for code-first development

---

## Reasoning

### What the landscape research showed

The February 15 wide research (DIP project) maps the Claude Code & Cowork plugin ecosystem for UI/UX design. Three clear signals:

1. **MCP-native is the direction.** Every new integration is MCP-first. For a project already MCP-native (zen-sci-portal talks to MCP servers via the Gateway), this ecosystem is well-aligned.

2. **shadcn/ui is the AI-native component standard.** The research rates it 4/5, describes it as "copy-paste, not npm install" — the design philosophy matches how I build: own the code, don't depend on runtime abstractions. The Plugged Platform already uses shadcn/ui (React, "new-york" style, neutral base). The Svelte 5 equivalent is **shadcn-svelte**.

3. **The ecosystem gap is orchestration, not components.** Point-solution component libraries are mature. The gap is intelligence and cross-tool workflow — which is exactly what zen-sci-portal IS. So the project doesn't need to solve a design gap; it needs good components to express what it already does.

### What the portal already has

The portal has a solid Tailwind setup:
- Brand color scale (blue, `brand-500: #3b5bdb`)
- Dark mode (`darkMode: 'class'`)
- Custom component classes (`.btn-primary`, `.btn-secondary`)
- `@tailwindcss/typography` plugin

This is enough to build competent UI. What it lacks is higher-level primitives: Select, Dialog, Tabs, Combobox, Toast — things needed for the invoke_tool surface.

### Why shadcn-svelte (not Skeleton, not Flowbite)

| Option | Rationale | Verdict |
|---|---|---|
| **shadcn-svelte** | Same design system as Plugged Platform. Svelte 5 runes native. Tailwind-based. Copy-paste ownership. Accessible (bits-ui primitives). | ✅ **Chosen** |
| Skeleton UI | Good design system, but opinionated themes can clash with existing brand setup | ❌ Skip |
| Flowbite Svelte | Good but Flowbite's design language differs from the clean-scientific aesthetic zen-sci needs | ❌ Skip |
| Bits UI (raw) | Lower-level than needed for the velocity required now | ❌ Not yet |

**Key alignment factor:** The Plugged Platform (another TresPies project) uses shadcn/ui with the "new-york" style and neutral base. Using shadcn-svelte on zen-sci-portal creates a consistent design language across the TresPies product stack.

### Why no new Claude plugins right now

Context7 (already installed in this Cowork environment) provides up-to-date documentation for:
- Svelte 5 / SvelteKit 2
- shadcn-svelte
- Tailwind CSS 3
- Tauri v2
- @tauri-apps/api

That covers 100% of what I need to build the invoke_tool surface. No additional plugin is needed for the current phase.

**When to add Figma MCP:** When Cruz wants to design before building — wireframes, visual specifications, layout exploration. Install via: the official Figma MCP server at `https://developers.figma.com/docs/figma-mcp-server/`. At that point, I can read Figma files directly and implement from design specs.

---

## Install Instructions for Cruz

Run on your machine in the `zen-sci-portal/` directory:

```bash
# 1. Initialize shadcn-svelte
pnpm dlx shadcn-svelte@latest init

# Respond to prompts:
# - Style: New York (matches Plugged Platform)
# - Base color: Neutral
# - CSS variables: Yes
# - Tailwind config: tailwind.config.js

# 2. Add the components needed for the invoke_tool surface:
pnpm dlx shadcn-svelte@latest add button input label select tabs badge card
```

This adds ~8 components as owned source files in `src/lib/components/ui/`. No runtime library dependency.

---

## What I'll build without waiting for the install

The invoke_tool surface component (`ToolInvoker.svelte`) will be built using the existing Tailwind system and `.btn-primary` / `.btn-secondary` patterns already in `app.css`. When shadcn-svelte is installed, the component is easily upgraded to use the richer primitives. The logic and Tauri API calls are the same regardless.
