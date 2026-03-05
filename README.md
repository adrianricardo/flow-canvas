# flow-canvas

Generate interactive state flow diagrams with live component previews. Powered by `@xyflow/react` + `dagre` auto-layout.

## Install

```bash
npx add-skill adrianricardo/flow-canvas
```

## Use

In Claude Code:

```
/flow-canvas cart → shipping → payment → confirmation
```

Automatically detects your framework, installs packages, discovers matching components, and generates a working flow canvas route. Super simple.

## What It Does

- Parses your flow from natural language or arrow notation
- Finds existing React components in your codebase that match each node
- Generates an interactive canvas with dagre auto-layout, live component previews, and a minimap
- Works with Next.js (App Router, Pages Router) and Vite

## Generated Files

**First flow** creates shared infrastructure + your flow definition:

```
src/
├── components/flow/          # Shared (generated once)
│   ├── FlowCanvas.tsx
│   ├── ComponentNode.tsx
│   ├── PlaceholderNode.tsx
│   └── FlowEdge.tsx
├── config/flows/
│   └── checkout.ts           # Your flow
└── app/flows/checkout/
    └── page.tsx              # Route
```

**Additional flows** only add 2 files each. Infrastructure is reused.
