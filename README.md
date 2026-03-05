# flow-canvas

A Claude Code skill that generates interactive state flow diagrams using `@xyflow/react` + `dagre` auto-layout. Describe your flow in natural language or arrow notation, and get a fully working, interactive canvas with live component previews.

## Installation

### Option 1: Symlink (local development)

```bash
ln -s /path/to/flow-canvas ~/.claude/skills/flow-canvas
```

### Option 2: Add to project's `skills-lock.json`

```json
{
  "flow-canvas": {
    "path": "/path/to/flow-canvas"
  }
}
```

### Option 3: Copy into your project

```bash
cp -r flow-canvas/ your-project/.claude/skills/flow-canvas/
```

## Usage

In any Claude Code session, describe a flow:

```
/flow-canvas cart → shipping → payment → confirmation
```

Or use natural language:

```
/flow-canvas Create a flow diagram for our checkout process.
The user starts at the cart, then enters shipping details,
then payment. If payment fails, they retry. If it succeeds,
they see a confirmation page.
```

## What Gets Generated

**First invocation** — creates shared infrastructure + your flow:

```
src/
├── components/flow/
│   ├── FlowCanvas.tsx        # Canvas + dagre layout + toolbar
│   ├── ComponentNode.tsx     # Live component renderer
│   ├── PlaceholderNode.tsx   # Decision/placeholder nodes
│   └── FlowEdge.tsx          # Bezier edges with "+" button
├── config/flows/
│   └── checkout.ts           # Your flow definition
└── app/flows/checkout/
    └── page.tsx              # Route page
```

**Subsequent invocations** — only adds 2 files per flow (definition + route page). Infrastructure is reused.

## Features

- Automatic dagre graph layout (toggle between left-to-right and top-to-bottom)
- Live React component previews inside nodes
- Lazy loading for all component previews
- Decision nodes with editable notes
- Click "+" on any edge to insert a new node
- MiniMap, zoom controls, and pan
- Dark mode support
- Works with Next.js (App Router, Pages Router) and Vite

## Requirements

- React 18+
- `@xyflow/react` and `@dagrejs/dagre` (auto-installed by the skill)
- `lucide-react` for icons (auto-installed if missing)
- Tailwind CSS recommended (falls back to inline styles)
