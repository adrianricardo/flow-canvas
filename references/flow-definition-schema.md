# Flow Definition Schema

Reference for the data structures used to define flows. Claude should read this when generating flow definition files.

## FlowNodeData Interface

```typescript
import type { Node, Edge, type ComponentType } from '@xyflow/react';
import { lazy } from 'react';

export interface FlowNodeData {
  /** Display label shown above component nodes or inside placeholders */
  label: string;
  /** Unique identifier linking this node to a component in the component map */
  componentId?: string;
  /** Node visual variant */
  variant?: 'component' | 'decision' | 'placeholder';
  /** Editable notes shown inside placeholder/decision nodes */
  notes?: string;
  /** Required by xyflow — allows arbitrary extra data */
  [key: string]: unknown;
}

export interface FlowComponentEntry {
  /** The React component to render (use lazy() for code splitting) */
  component: ComponentType<any>;
  /** Props to pass to the component for preview */
  defaultProps?: Record<string, unknown>;
  /** Override the default node width (default: 280) */
  width?: number;
}

/** Map of componentId → component entry */
export type FlowComponentMap = Record<string, FlowComponentEntry>;
```

## Example Flow Definition

This is the pattern for a per-flow definition file (e.g., `src/config/flows/checkout.ts`):

```typescript
import type { Node, Edge } from '@xyflow/react';
import { lazy, type ComponentType } from 'react';

// ---- Types ----

export interface FlowNodeData {
  label: string;
  componentId?: string;
  variant?: 'component' | 'decision' | 'placeholder';
  notes?: string;
  [key: string]: unknown;
}

export interface FlowComponentEntry {
  component: ComponentType<any>;
  defaultProps?: Record<string, unknown>;
  width?: number;
}

export type FlowComponentMap = Record<string, FlowComponentEntry>;

// ---- Component Map ----

export const componentMap: FlowComponentMap = {
  cart: {
    component: lazy(() => import('@/components/CartPage')),
    defaultProps: { items: [] },
    width: 400,
  },
  shipping: {
    component: lazy(() => import('@/components/ShippingForm')),
  },
  payment: {
    component: lazy(() => import('@/components/PaymentForm')),
    defaultProps: { amount: 99.99 },
  },
  confirmation: {
    component: lazy(() => import('@/components/ConfirmationPage')),
    defaultProps: { orderId: 'DEMO-001' },
  },
};

// ---- Nodes ----

export const nodes: Node<FlowNodeData>[] = [
  {
    id: 'cart',
    type: 'componentNode',
    position: { x: 0, y: 0 }, // dagre will reposition
    data: {
      label: 'Cart',
      componentId: 'cart',
      variant: 'component',
    },
  },
  {
    id: 'shipping',
    type: 'componentNode',
    position: { x: 0, y: 0 },
    data: {
      label: 'Shipping',
      componentId: 'shipping',
      variant: 'component',
    },
  },
  {
    id: 'check-payment',
    type: 'placeholderNode',
    position: { x: 0, y: 0 },
    data: {
      label: 'Payment Valid?',
      variant: 'decision',
      notes: 'Validates card details with Stripe',
    },
  },
  {
    id: 'payment',
    type: 'componentNode',
    position: { x: 0, y: 0 },
    data: {
      label: 'Payment',
      componentId: 'payment',
      variant: 'component',
    },
  },
  {
    id: 'confirmation',
    type: 'componentNode',
    position: { x: 0, y: 0 },
    data: {
      label: 'Confirmation',
      componentId: 'confirmation',
      variant: 'component',
    },
  },
];

// ---- Edges ----

export const edges: Edge[] = [
  {
    id: 'e-cart-shipping',
    source: 'cart',
    target: 'shipping',
    type: 'flowEdge',
    label: 'Continue',
  },
  {
    id: 'e-shipping-payment',
    source: 'shipping',
    target: 'payment',
    type: 'flowEdge',
    label: 'Address confirmed',
  },
  {
    id: 'e-payment-check',
    source: 'payment',
    target: 'check-payment',
    type: 'flowEdge',
    label: 'Submit',
  },
  {
    id: 'e-check-confirm',
    source: 'check-payment',
    target: 'confirmation',
    type: 'flowEdge',
    label: 'Approved',
  },
  {
    id: 'e-check-retry',
    source: 'check-payment',
    target: 'payment',
    type: 'flowEdge',
    label: 'Declined',
  },
];
```

## Node ID Conventions

- Use kebab-case: `cart`, `check-payment`, `order-confirmation`
- Keep IDs short and descriptive
- Decision nodes: prefix with `check-` or `decide-` for clarity

## Edge ID Conventions

- Format: `e-{source}-{target}` (e.g., `e-cart-shipping`)
- If multiple edges between same nodes, append a suffix: `e-check-payment-yes`, `e-check-payment-no`

## Position Values

All `position` values can be `{ x: 0, y: 0 }` — dagre auto-layout will compute actual positions. The values in the definition are only used as fallbacks if dagre isn't applied.

## Nodes Without Components

Nodes that don't have a matching component in the `componentMap` will automatically render as `PlaceholderNode`. This is the expected behavior for:
- Decision nodes (always render as placeholders with the diamond style)
- Steps where the component hasn't been built yet
- Abstract steps that don't map to a single component
