# Component Wiring Patterns

Canonical implementations for all generated infrastructure files. Claude reads these and adapts them to each project's framework, styling, and conventions.

## FlowCanvas.tsx

The main canvas component. Accepts `nodes`, `edges`, and `componentMap` as props.

```tsx
"use client";

import { useCallback, useMemo, useState } from 'react';
import {
  ReactFlow,
  ReactFlowProvider,
  Controls,
  Background,
  MiniMap,
  Position,
  useNodesState,
  useEdgesState,
  useReactFlow,
  type Node,
  type Edge,
} from '@xyflow/react';
import '@xyflow/react/dist/style.css';
import Dagre from '@dagrejs/dagre';
import { ArrowDownUp, RotateCcw } from 'lucide-react';

import ComponentNode from './ComponentNode';
import PlaceholderNode from './PlaceholderNode';
import FlowEdge from './FlowEdge';
import type { FlowNodeData, FlowComponentMap } from '@/config/flows/types';

// Node dimensions for dagre layout
const COMPONENT_NODE_WIDTH = 280;
const COMPONENT_NODE_HEIGHT = Math.ceil(COMPONENT_NODE_WIDTH * 1.4) + 60;
const PLACEHOLDER_NODE_WIDTH = 200;
const PLACEHOLDER_NODE_HEIGHT = 100;

type LayoutDirection = 'TB' | 'LR';

function getLayoutedElements(
  nodes: Node<FlowNodeData>[],
  edges: Edge[],
  direction: LayoutDirection
) {
  const g = new Dagre.graphlib.Graph().setDefaultEdgeLabel(() => ({}));

  g.setGraph({
    rankdir: direction,
    nodesep: 100,
    ranksep: 120,
    marginx: 40,
    marginy: 40,
  });

  nodes.forEach((node) => {
    const isComponent = node.type === 'componentNode';
    g.setNode(node.id, {
      width: isComponent ? COMPONENT_NODE_WIDTH : PLACEHOLDER_NODE_WIDTH,
      height: isComponent ? COMPONENT_NODE_HEIGHT : PLACEHOLDER_NODE_HEIGHT,
    });
  });

  edges.forEach((edge) => {
    g.setEdge(edge.source, edge.target);
  });

  Dagre.layout(g);

  const isHorizontal = direction === 'LR';

  const layoutedNodes = nodes.map((node) => {
    const dagreNode = g.node(node.id);
    const isComponent = node.type === 'componentNode';
    const w = isComponent ? COMPONENT_NODE_WIDTH : PLACEHOLDER_NODE_WIDTH;
    const h = isComponent ? COMPONENT_NODE_HEIGHT : PLACEHOLDER_NODE_HEIGHT;

    return {
      ...node,
      targetPosition: isHorizontal ? Position.Left : Position.Top,
      sourcePosition: isHorizontal ? Position.Right : Position.Bottom,
      position: {
        x: dagreNode.x - w / 2,
        y: dagreNode.y - h / 2,
      },
    };
  });

  return { nodes: layoutedNodes, edges };
}

const nodeTypes = {
  componentNode: ComponentNode,
  placeholderNode: PlaceholderNode,
};

const edgeTypes = {
  flowEdge: FlowEdge,
};

interface FlowCanvasInnerProps {
  initialNodes: Node<FlowNodeData>[];
  initialEdges: Edge[];
  componentMap: FlowComponentMap;
}

function FlowCanvasInner({ initialNodes, initialEdges, componentMap }: FlowCanvasInnerProps) {
  const [direction, setDirection] = useState<LayoutDirection>('LR');
  const [showResetConfirm, setShowResetConfirm] = useState(false);
  const { fitView } = useReactFlow();

  const initialLayout = useMemo(
    () => getLayoutedElements(initialNodes, initialEdges, direction),
    // eslint-disable-next-line react-hooks/exhaustive-deps
    [direction]
  );

  const [nodes, setNodes, onNodesChange] = useNodesState(initialLayout.nodes);
  const [edges, setEdges, onEdgesChange] = useEdgesState(initialLayout.edges);

  const runAutoLayout = useCallback(
    (dir: LayoutDirection = direction) => {
      const { nodes: layoutedNodes, edges: layoutedEdges } =
        getLayoutedElements(initialNodes, initialEdges, dir);
      setNodes(layoutedNodes);
      setEdges(layoutedEdges);
      setTimeout(() => fitView({ padding: 0.2 }), 0);
    },
    [direction, initialNodes, initialEdges, setNodes, setEdges, fitView]
  );

  const toggleDirection = useCallback(() => {
    const newDir = direction === 'TB' ? 'LR' : 'TB';
    setDirection(newDir);
    runAutoLayout(newDir);
  }, [direction, runAutoLayout]);

  const handleReset = useCallback(() => {
    if (!showResetConfirm) {
      setShowResetConfirm(true);
      setTimeout(() => setShowResetConfirm(false), 3000);
      return;
    }
    runAutoLayout();
    setShowResetConfirm(false);
  }, [showResetConfirm, runAutoLayout]);

  return (
    <div className="w-full h-full relative">
      <ReactFlow
        nodes={nodes}
        edges={edges}
        onNodesChange={onNodesChange}
        onEdgesChange={onEdgesChange}
        nodeTypes={nodeTypes}
        edgeTypes={edgeTypes}
        fitView
        fitViewOptions={{ padding: 0.2 }}
        minZoom={0.1}
        maxZoom={1.5}
        proOptions={{ hideAttribution: true }}
      >
        <Controls position="bottom-left" />
        <Background gap={20} size={1} />
        <MiniMap
          nodeStrokeWidth={3}
          pannable
          zoomable
          position="bottom-right"
          className="!bg-background !border-border"
        />
      </ReactFlow>

      {/* Toolbar */}
      <div className="absolute top-4 right-4 flex items-center gap-2">
        <button
          onClick={toggleDirection}
          className="flex items-center gap-2 px-3 py-2 rounded-lg text-sm font-medium shadow-sm transition-colors bg-background text-foreground border border-border hover:bg-muted"
        >
          <ArrowDownUp className="w-4 h-4" />
          {direction === 'TB' ? 'Top \u2192 Bottom' : 'Left \u2192 Right'}
        </button>
        <button
          onClick={handleReset}
          className={`flex items-center gap-2 px-3 py-2 rounded-lg text-sm font-medium shadow-sm transition-colors ${
            showResetConfirm
              ? 'bg-red-600 text-white hover:bg-red-700'
              : 'bg-background text-foreground border border-border hover:bg-muted'
          }`}
        >
          <RotateCcw className="w-4 h-4" />
          {showResetConfirm ? 'Confirm reset' : 'Reset layout'}
        </button>
      </div>
    </div>
  );
}

interface FlowCanvasProps {
  nodes: Node<FlowNodeData>[];
  edges: Edge[];
  componentMap: FlowComponentMap;
}

export default function FlowCanvas({ nodes, edges, componentMap }: FlowCanvasProps) {
  return (
    <ReactFlowProvider>
      <FlowCanvasInner
        initialNodes={nodes}
        initialEdges={edges}
        componentMap={componentMap}
      />
    </ReactFlowProvider>
  );
}
```

### Adaptation Notes — FlowCanvas

- **Vite / non-Next.js**: Remove `"use client"` directive. Everything else stays the same.
- **No Tailwind**: Replace utility classes with inline styles or CSS modules. Key styles needed:
  - Container: `width: 100%; height: 100%; position: relative`
  - Toolbar: `position: absolute; top: 16px; right: 16px; display: flex; gap: 8px`
  - Buttons: standard button styling with hover states
- **MiniMap className**: The `!bg-background` classes assume a CSS variable `--background` exists. For non-shadcn projects, use inline styles or omit.
- **ComponentMap propagation**: The `componentMap` prop is passed through React context or prop drilling to `ComponentNode`. The simplest approach is to store it in a module-level variable that `ComponentNode` reads, set before rendering.

---

## ComponentNode.tsx

Renders a live React component inside the flow node, using lazy loading.

```tsx
"use client";

import { memo, Suspense } from 'react';
import { Handle, Position, type NodeProps } from '@xyflow/react';
import type { FlowNodeData, FlowComponentMap } from '@/config/flows/types';

const DEFAULT_WIDTH = 280;

interface ComponentNodeProps extends NodeProps {
  data: FlowNodeData;
}

// Module-level ref — set by FlowCanvas before render
let _componentMap: FlowComponentMap = {};

export function setComponentMap(map: FlowComponentMap) {
  _componentMap = map;
}

function ComponentNode({ data, targetPosition, sourcePosition }: ComponentNodeProps) {
  const { label, componentId } = data;
  const entry = componentId ? _componentMap[componentId] : null;

  const nodeWidth = entry?.width ?? DEFAULT_WIDTH;
  const PANEL_WIDTH = entry?.width ?? 509;
  const SCALE = nodeWidth / PANEL_WIDTH;

  const Component = entry?.component;
  const defaultProps = entry?.defaultProps ?? {};

  return (
    <div className="relative">
      <Handle
        type="target"
        position={targetPosition ?? Position.Top}
        className="!w-3 !h-3 !bg-muted-foreground/40 !border-2 !border-background"
      />

      {/* Label pill */}
      <div className="absolute -top-8 left-1/2 -translate-x-1/2 whitespace-nowrap">
        <span className="px-2.5 py-1 rounded-full text-xs font-medium bg-foreground text-background shadow-sm">
          {label}
        </span>
      </div>

      {/* Live component preview */}
      <div
        className="overflow-hidden rounded-xl border border-border bg-background shadow-md"
        style={{ width: nodeWidth, height: nodeWidth * 1.4 }}
      >
        <div
          className="pointer-events-none origin-top-left"
          style={{
            width: PANEL_WIDTH,
            transform: `scale(${SCALE})`,
          }}
        >
          {Component ? (
            <Suspense
              fallback={
                <div className="flex items-center justify-center h-64 text-muted-foreground text-sm">
                  Loading...
                </div>
              }
            >
              <Component {...defaultProps} />
            </Suspense>
          ) : (
            <div className="flex items-center justify-center h-64 text-muted-foreground text-sm">
              No component mapped
            </div>
          )}
        </div>
      </div>

      {/* Component ID */}
      {componentId && (
        <div className="mt-1.5 text-center">
          <code className="text-[10px] text-muted-foreground font-mono">{componentId}</code>
        </div>
      )}

      <Handle
        type="source"
        position={sourcePosition ?? Position.Bottom}
        className="!w-3 !h-3 !bg-muted-foreground/40 !border-2 !border-background"
      />
    </div>
  );
}

export default memo(ComponentNode);
```

### Adaptation Notes — ComponentNode

- **Module-level componentMap**: The `setComponentMap` + module variable pattern avoids prop drilling through xyflow's node rendering. FlowCanvas calls `setComponentMap(componentMap)` before rendering. An alternative is React context.
- **Scaling**: The default scales a 509px-wide component into a 280px preview. If the project's components are narrower, adjust `PANEL_WIDTH` or let each entry specify its own `width`.
- **No Tailwind**: Replace Handle classes with inline styles. Key: handles should be small circles (12px) with a subtle border.

---

## PlaceholderNode.tsx

Decision diamonds and placeholder nodes with editable notes.

```tsx
"use client";

import { memo, useCallback } from 'react';
import { Handle, Position, useReactFlow, type NodeProps } from '@xyflow/react';
import { GitBranch } from 'lucide-react';
import type { FlowNodeData } from '@/config/flows/types';

function PlaceholderNode({ id, data, targetPosition, sourcePosition }: NodeProps) {
  const nodeData = data as FlowNodeData;
  const { label, variant, notes } = nodeData;
  const { setNodes } = useReactFlow();
  const isDecision = variant === 'decision';

  const handleNotesChange = useCallback(
    (e: React.FocusEvent<HTMLTextAreaElement>) => {
      setNodes((nds) =>
        nds.map((n) =>
          n.id === id ? { ...n, data: { ...n.data, notes: e.target.value } } : n
        )
      );
    },
    [id, setNodes]
  );

  const stopPropagation = useCallback((e: React.MouseEvent) => {
    e.stopPropagation();
  }, []);

  return (
    <div
      className={`relative flex flex-col items-center justify-center gap-2 px-6 py-4 rounded-xl ${
        isDecision
          ? 'bg-amber-50 dark:bg-amber-950/30 border-2 border-amber-300 dark:border-amber-700 min-w-[200px]'
          : 'bg-muted/30 border-2 border-dashed border-muted-foreground/30 min-w-[200px]'
      }`}
    >
      <Handle
        type="target"
        position={targetPosition ?? Position.Top}
        className="!w-3 !h-3 !bg-muted-foreground/40 !border-2 !border-background"
      />

      {isDecision && (
        <GitBranch className="w-5 h-5 text-amber-600 dark:text-amber-400" />
      )}

      <span className={`text-sm font-semibold ${
        isDecision ? 'text-amber-800 dark:text-amber-200' : 'text-muted-foreground'
      }`}>
        {label}
      </span>

      <textarea
        defaultValue={notes || ''}
        onBlur={handleNotesChange}
        onMouseDown={stopPropagation}
        onMouseUp={stopPropagation}
        onClick={stopPropagation}
        placeholder="Add notes..."
        className="w-full min-h-[40px] text-xs bg-transparent border-none outline-none resize-none text-center text-muted-foreground placeholder:text-muted-foreground/40"
        rows={2}
      />

      <Handle
        type="source"
        position={sourcePosition ?? Position.Bottom}
        className="!w-3 !h-3 !bg-muted-foreground/40 !border-2 !border-background"
      />
    </div>
  );
}

export default memo(PlaceholderNode);
```

### Adaptation Notes — PlaceholderNode

- **Already generic** — this component has no project-specific dependencies
- **No Tailwind**: the amber color scheme for decisions can use inline `backgroundColor` / `borderColor`
- **lucide-react**: if the project doesn't use lucide, replace `GitBranch` with an SVG or emoji

---

## FlowEdge.tsx

Bezier edges with an inline "+" button to insert new nodes.

```tsx
"use client";

import { memo, useState, useCallback } from 'react';
import {
  getBezierPath,
  EdgeLabelRenderer,
  useReactFlow,
  type EdgeProps,
} from '@xyflow/react';
import { Plus } from 'lucide-react';

function FlowEdge({
  id,
  sourceX,
  sourceY,
  targetX,
  targetY,
  sourcePosition,
  targetPosition,
  label,
  style = {},
}: EdgeProps) {
  const [isHovered, setIsHovered] = useState(false);
  const { setNodes, setEdges } = useReactFlow();

  const [edgePath, labelX, labelY] = getBezierPath({
    sourceX,
    sourceY,
    targetX,
    targetY,
    sourcePosition,
    targetPosition,
  });

  const handleAddNode = useCallback(() => {
    const newNodeId = `placeholder-${Date.now()}`;
    const midX = (sourceX + targetX) / 2 - 100;
    const midY = (sourceY + targetY) / 2 - 30;

    setNodes((nds) => [
      ...nds,
      {
        id: newNodeId,
        type: 'placeholderNode',
        position: { x: midX, y: midY },
        data: {
          label: 'New step',
          variant: 'placeholder',
          notes: '',
        },
      },
    ]);

    setEdges((eds) => {
      const originalEdge = eds.find((e) => e.id === id);
      if (!originalEdge) return eds;

      return [
        ...eds.filter((e) => e.id !== id),
        {
          id: `${id}-a`,
          source: originalEdge.source,
          target: newNodeId,
          type: 'flowEdge',
        },
        {
          id: `${id}-b`,
          source: newNodeId,
          target: originalEdge.target,
          type: 'flowEdge',
          label: originalEdge.label,
        },
      ];
    });
  }, [id, sourceX, sourceY, targetX, targetY, setNodes, setEdges]);

  return (
    <>
      <path
        d={edgePath}
        fill="none"
        className="stroke-muted-foreground/40"
        strokeWidth={2}
        style={style}
      />
      {/* Invisible wider path for hover detection */}
      <path
        d={edgePath}
        fill="none"
        stroke="transparent"
        strokeWidth={20}
        onMouseEnter={() => setIsHovered(true)}
        onMouseLeave={() => setIsHovered(false)}
      />
      <EdgeLabelRenderer>
        {label && (
          <div
            className="absolute pointer-events-none text-[10px] font-medium text-muted-foreground bg-background px-1.5 py-0.5 rounded border border-border"
            style={{
              transform: `translate(-50%, -50%) translate(${labelX}px, ${labelY - 14}px)`,
            }}
          >
            {label}
          </div>
        )}
        {isHovered && (
          <div
            className="absolute"
            style={{
              transform: `translate(-50%, -50%) translate(${labelX}px, ${labelY + (label ? 10 : 0)}px)`,
            }}
            onMouseEnter={() => setIsHovered(true)}
            onMouseLeave={() => setIsHovered(false)}
          >
            <button
              onClick={handleAddNode}
              className="w-6 h-6 rounded-full bg-foreground text-background flex items-center justify-center shadow-md hover:scale-110 transition-transform"
            >
              <Plus className="w-3.5 h-3.5" />
            </button>
          </div>
        )}
      </EdgeLabelRenderer>
    </>
  );
}

export default memo(FlowEdge);
```

### Adaptation Notes — FlowEdge

- **Already generic** — no project-specific imports
- **stroke class**: `stroke-muted-foreground/40` requires Tailwind + shadcn CSS variables. Without Tailwind, use `stroke="#9ca3af"` or similar.

---

## Route Page Pattern

### Next.js App Router

```tsx
"use client";

import dynamic from 'next/dynamic';
import { nodes, edges, componentMap } from '@/config/flows/checkout';

const FlowCanvas = dynamic(
  () => import('@/components/flow/FlowCanvas'),
  { ssr: false }
);

export default function CheckoutFlowPage() {
  return (
    <div className="w-screen h-screen">
      <FlowCanvas nodes={nodes} edges={edges} componentMap={componentMap} />
    </div>
  );
}
```

### Next.js Pages Router

```tsx
import dynamic from 'next/dynamic';
import { nodes, edges, componentMap } from '@/config/flows/checkout';

const FlowCanvas = dynamic(
  () => import('@/components/flow/FlowCanvas'),
  { ssr: false }
);

export default function CheckoutFlowPage() {
  return (
    <div style={{ width: '100vw', height: '100vh' }}>
      <FlowCanvas nodes={nodes} edges={edges} componentMap={componentMap} />
    </div>
  );
}
```

### Vite

```tsx
import { lazy, Suspense } from 'react';
import { nodes, edges, componentMap } from '@/config/flows/checkout';

const FlowCanvas = lazy(() => import('@/components/flow/FlowCanvas'));

export default function CheckoutFlowPage() {
  return (
    <div style={{ width: '100vw', height: '100vh' }}>
      <Suspense fallback={<div>Loading flow...</div>}>
        <FlowCanvas nodes={nodes} edges={edges} componentMap={componentMap} />
      </Suspense>
    </div>
  );
}
```

---

## ComponentMap Propagation

The FlowCanvas needs to make the `componentMap` available to `ComponentNode` without prop drilling through xyflow internals. Two patterns:

### Pattern A: Module-level setter (simpler)

FlowCanvas calls `setComponentMap(componentMap)` before render. ComponentNode reads from the module variable. This is the default pattern shown above.

```tsx
// In FlowCanvas, before returning JSX:
import { setComponentMap } from './ComponentNode';

// Inside FlowCanvasInner:
useMemo(() => setComponentMap(componentMap), [componentMap]);
```

### Pattern B: React Context (safer for concurrent rendering)

Create a `FlowContext` that wraps the ReactFlowProvider:

```tsx
const FlowContext = createContext<FlowComponentMap>({});

export function FlowCanvas({ nodes, edges, componentMap }: FlowCanvasProps) {
  return (
    <FlowContext.Provider value={componentMap}>
      <ReactFlowProvider>
        <FlowCanvasInner initialNodes={nodes} initialEdges={edges} />
      </ReactFlowProvider>
    </FlowContext.Provider>
  );
}

// In ComponentNode:
const componentMap = useContext(FlowContext);
```

Use Pattern A by default. Use Pattern B if the project uses React 18 concurrent features or renders multiple FlowCanvas instances on the same page.
