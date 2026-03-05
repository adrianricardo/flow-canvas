---
name: flow-canvas
description: >
  Generate interactive state flow diagrams using @xyflow/react + dagre auto-layout.
  Triggers on: "flow canvas", "flow diagram", "state flow", "visualize flow",
  "A → B → C" arrow patterns describing state transitions.
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
  - AskUserQuestion
model: opus
---

# Flow Canvas — Claude Code Skill

Generate interactive, auto-layouted flow diagrams that render live React components at each node. Uses `@xyflow/react` for the canvas and `@dagrejs/dagre` for automatic graph layout.

## Execution Protocol

Follow these 8 steps in order. Do not skip steps.

### Step 1: Parse Flow Description

Extract nodes and edges from the user's natural language or arrow notation.

- Each item separated by `→` or described as a step becomes a **node**
- Words like "if", "check", "decide", "branch" indicate a **decision** node (variant: `'decision'`)
- Everything else is a **component** node (variant: `'component'`)
- Edge labels come from transition descriptions (e.g., "on success", "if paid")

Build an internal representation:
```
nodes: [{ id, label, variant }]
edges: [{ source, target, label? }]
```

### Step 2: Confirm with User

Use `AskUserQuestion` to show the inferred graph as ASCII art and ask for confirmation.

Example:
```
I parsed your flow as:

  [Cart] → [Shipping] → [Payment] →(check: paid?)
                                       ├─ yes → [Confirmation]
                                       └─ no  → [Payment Error]

5 nodes (4 component, 1 decision), 5 edges.
Does this look right?
```

Options: "Looks good", "Let me adjust" (with description field for corrections).

### Step 3: Detect Framework

Scan the project to determine the tech stack:

1. **Package manager** — check for `bun.lockb` (bun), `yarn.lock` (yarn), `pnpm-lock.yaml` (pnpm), else npm
2. **Framework** — check `next.config.*` (Next.js), `vite.config.*` (Vite), `remix.config.*` (Remix)
3. **Next.js router** — if Next.js, check for `app/` directory (App Router) vs `pages/` (Pages Router)
4. **TypeScript** — check for `tsconfig.json`
5. **Tailwind** — check for `tailwind.config.*` or Tailwind imports in CSS
6. **Path aliases** — read `tsconfig.json` for `paths` config (e.g., `@/` → `src/`)

Store the detected config for code generation in later steps.

### Step 4: Discover Components

For each component node, attempt to find a matching React component in the codebase:

1. **Glob** for files matching the node label: `**/{NodeLabel,node-label,nodeLabel}.{tsx,jsx}`
2. **Grep** for `export default` or `export function {NodeLabel}` patterns
3. **Read** the top candidate to confirm it's a valid React component (has JSX return)

Record matches as `{ nodeId, componentPath, exportName }`.

### Step 5: Confirm Component Mappings

Use `AskUserQuestion` to show discovered mappings:

```
Component mappings:

  Cart       → src/components/CartPage.tsx (default export)
  Shipping   → src/components/ShippingForm.tsx (default export)
  Payment    → No match found
  Confirmation → src/components/ConfirmationPage.tsx (default export)

Nodes without matches will render as placeholders.
Proceed with these mappings?
```

Options: "Looks good", "Let me adjust".

### Step 6: Install Dependencies

Check if `@xyflow/react` and `@dagrejs/dagre` are already in `package.json`. If not, install them using the detected package manager:

```bash
# yarn
yarn add @xyflow/react @dagrejs/dagre

# npm
npm install @xyflow/react @dagrejs/dagre

# pnpm
pnpm add @xyflow/react @dagrejs/dagre

# bun
bun add @xyflow/react @dagrejs/dagre
```

Also install `lucide-react` if not present (used for icons in nodes/edges).

### Step 7: Generate Files

Read the reference files in this skill's `references/` directory for canonical implementations:

- `references/flow-definition-schema.md` — FlowNodeData types + how to structure flow definitions
- `references/component-wiring-patterns.md` — Canonical code for FlowCanvas, ComponentNode, PlaceholderNode, FlowEdge

**Infrastructure files** (generate only if `components/flow/FlowCanvas.tsx` doesn't exist):

| File | Purpose |
|---|---|
| `components/flow/FlowCanvas.tsx` | Canvas + dagre layout + direction toggle + reset |
| `components/flow/ComponentNode.tsx` | Renders mapped components with lazy loading |
| `components/flow/PlaceholderNode.tsx` | Decision diamonds + editable placeholder nodes |
| `components/flow/FlowEdge.tsx` | Bezier edges with "+" insert button |

**Per-flow files** (always generate):

| File | Purpose |
|---|---|
| `config/flows/{flowName}.ts` | Node/edge definitions + component map |
| `app/flows/{flowName}/page.tsx` | Route page (dynamic import, SSR disabled) |

Adapt all generated code to the detected framework:
- **Path aliases**: use `@/` or relative paths based on tsconfig
- **Vite**: use `React.lazy()` instead of `next/dynamic`
- **Pages Router**: generate in `pages/flows/` instead of `app/flows/`
- **No Tailwind**: use inline styles or CSS modules
- **JavaScript**: omit type annotations

### Step 8: Report Completion

Print a summary:

```
Flow canvas generated!

Files created:
  src/components/flow/FlowCanvas.tsx      (shared infrastructure)
  src/components/flow/ComponentNode.tsx    (shared infrastructure)
  src/components/flow/PlaceholderNode.tsx  (shared infrastructure)
  src/components/flow/FlowEdge.tsx         (shared infrastructure)
  src/config/flows/checkout.ts            (flow definition)
  src/app/flows/checkout/page.tsx          (route)

Visit: http://localhost:3000/flows/checkout
```

For second+ flows, only list the 2 new files and note that infrastructure is reused.

## Important Notes

- Always use `"use client"` directive for all generated components (they use hooks and browser APIs)
- The FlowCanvas accepts `nodes`, `edges`, and `componentMap` as props — it does NOT import from a hardcoded flow definition
- Component nodes use `React.lazy()` / `next/dynamic` for code splitting
- All styling uses Tailwind utility classes with dark mode support via `dark:` variants
- Node dimensions: component nodes default to 280px wide, placeholder/decision nodes to 200px
- The dagre layout runs automatically — no manual positioning needed in flow definitions
