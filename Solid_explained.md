# TSX as a Bridge to Native UI: How OpenTUI's Solid Package Works

This document explains how OpenTUI's Solid package uses TSX/JSX as a declarative bridge between UI logic and an underlying native component framework. The goal is to help you port this architecture to render native UI components in a Zig-based application (e.g., rendering to an HWND).

## Architecture Overview

The architecture has three layers:

```
┌─────────────────────────────────────────────────────────────┐
│  TSX/JSX Layer (Declarative UI)                             │
│  - Components written in TSX                                │
│  - Reactive state management via Solid.js signals           │
│  - Familiar HTML-like syntax: <box>, <text>, <input>        │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  Reconciler (Bridge Layer)                                  │
│  - Translates TSX operations into native node operations    │
│  - createElement, insertNode, removeNode, setProperty       │
│  - Handles reactivity: effects re-run when signals change   │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  Native Renderable Layer (Your Zig/HWND Components)         │
│  - Actual UI components: Box, Text, Input, ScrollBox, etc.  │
│  - Layout engine (Yoga/Flexbox)                             │
│  - Rendering to target (terminal buffer, HWND, etc.)        │
└─────────────────────────────────────────────────────────────┘
```

## Key Components

### 1. The Reconciler (`src/reconciler.ts`)

The reconciler is the heart of the bridge. It implements a set of primitive operations that Solid.js uses to manage the UI tree:

```typescript
createRenderer<DomNode>({
  // Create a native element by tag name
  createElement(tagName: string): DomNode {
    const elements = getComponentCatalogue()
    return new elements[tagName](renderContext, { id })
  },

  // Create a text node
  createTextNode(value: string): TextNode {
    return TextNode.fromString(value, { id })
  },

  // Insert a node into the tree
  insertNode(parent: DomNode, node: DomNode, anchor?: DomNode): void {
    if (!anchor) {
      parent.add(node)
    } else {
      parent.add(node, anchorIndex)
    }
  },

  // Remove a node from the tree
  removeNode(parent: DomNode, node: DomNode): void {
    parent.remove(node.id)
    node.destroyRecursively()
  },

  // Set a property on a node
  setProperty(node: DomNode, name: string, value: any, prev: any): void {
    // Handle events (on:click, onChange, etc.)
    if (name.startsWith("on:")) {
      node.on(eventName, value)
      return
    }
    // Handle style prop
    if (name === "style") {
      for (const prop in value) {
        node[prop] = value[prop]
      }
      return
    }
    // Direct property assignment
    node[name] = value
  },

  // Tree traversal
  getParentNode(node): DomNode | undefined { ... },
  getFirstChild(node): DomNode | undefined { ... },
  getNextSibling(node): DomNode | undefined { ... },
})
```

### 2. Component Catalogue (`src/elements/index.ts`)

Maps JSX tag names to native renderable classes:

```typescript
export const baseComponents = {
  box: BoxRenderable,
  text: TextRenderable,
  input: InputRenderable,
  select: SelectRenderable,
  scrollbox: ScrollBoxRenderable,
  // ... etc
}

// Extend with custom components
export function extend<T>(objects: T): void {
  Object.assign(componentCatalogue, objects)
}
```

### 3. The Universal Renderer (`src/renderer/universal.js`)

This is Solid's core rendering logic (borrowed from `solid-js/universal`). Key functions:

- **`insert(parent, accessor, marker)`**: Inserts content into a parent. If `accessor` is a function (reactive), wraps it in `createRenderEffect` so it re-runs when dependencies change.
- **`reconcileArrays(parent, oldArray, newArray)`**: Efficiently diffs and patches arrays of children.
- **`spreadExpression(node, props)`**: Applies reactive props to a node.

### 4. Entry Point (`index.ts`)

```typescript
export const render = async (node: () => JSX.Element, config = {}) => {
  // 1. Create the native renderer (your Zig backend)
  const renderer = await createCliRenderer(config)

  // 2. Attach to the render engine
  engine.attach(renderer)

  // 3. Render the TSX tree into the native root
  const dispose = renderInternal(
    () => createComponent(RendererContext.Provider, {
      value: renderer,
      children: createComponent(node, {})
    }),
    renderer.root  // The native root node
  )
}
```

## How Reactivity Works

Solid.js uses **signals** for reactivity. When you write:

```tsx
const [count, setCount] = createSignal(0)

return <text>Count: {count()}</text>
```

Here's what happens:

1. **Initial Render**: The reconciler calls `createElement("text")` and `createTextNode("Count: 0")`.
2. **Reactive Binding**: The `{count()}` expression is wrapped in a render effect.
3. **Signal Update**: When `setCount(1)` is called, Solid automatically re-runs the effect.
4. **DOM Update**: The reconciler's `replaceText()` is called to update the text node.

The key insight: **Solid tracks which signals are read during an effect, then re-runs that effect when those signals change.**

## Porting to Zig: What You Need to Implement

### 1. Native Node Interface

Your Zig components need to implement these operations:

```zig
const Node = struct {
    id: []const u8,
    parent: ?*Node,
    children: std.ArrayList(*Node),

    // Core operations the reconciler will call
    pub fn add(self: *Node, child: *Node, index: ?usize) void { ... }
    pub fn remove(self: *Node, id: []const u8) void { ... }
    pub fn getChildren(self: *Node) []*Node { ... }
    pub fn destroy(self: *Node) void { ... }

    // Property setters (called by setProperty)
    pub fn setWidth(self: *Node, value: i32) void { ... }
    pub fn setHeight(self: *Node, value: i32) void { ... }
    pub fn setBackgroundColor(self: *Node, color: u32) void { ... }
    // ... etc
};
```

### 2. Component Registry

Map tag names to constructors:

```zig
const ComponentRegistry = struct {
    pub fn create(tag: []const u8, ctx: *RenderContext, opts: NodeOptions) *Node {
        if (std.mem.eql(u8, tag, "box")) return BoxNode.create(ctx, opts);
        if (std.mem.eql(u8, tag, "text")) return TextNode.create(ctx, opts);
        if (std.mem.eql(u8, tag, "button")) return ButtonNode.create(ctx, opts);
        // ...
    }
};
```

### 3. FFI Bridge (TypeScript → Zig)

You need a way for the TypeScript reconciler to call into Zig:

```typescript
// TypeScript side
const zigBridge = {
  createElement(tag: string, id: string): number {
    return zigFFI.createElement(tag, id)
  },
  insertNode(parentHandle: number, childHandle: number, anchorHandle?: number) {
    zigFFI.insertNode(parentHandle, childHandle, anchorHandle ?? -1)
  },
  removeNode(parentHandle: number, childHandle: number) {
    zigFFI.removeNode(parentHandle, childHandle)
  },
  setProperty(handle: number, name: string, value: any) {
    zigFFI.setProperty(handle, name, serializeValue(value))
  }
}
```

```zig
// Zig side (exported functions)
export fn createElement(tag_ptr: [*]const u8, tag_len: usize, id_ptr: [*]const u8, id_len: usize) i32 {
    const tag = tag_ptr[0..tag_len];
    const id = id_ptr[0..id_len];
    const node = ComponentRegistry.create(tag, global_ctx, .{ .id = id });
    return node_handles.insert(node);
}

export fn insertNode(parent_handle: i32, child_handle: i32, anchor_handle: i32) void {
    const parent = node_handles.get(parent_handle);
    const child = node_handles.get(child_handle);
    const anchor = if (anchor_handle >= 0) node_handles.get(anchor_handle) else null;
    parent.add(child, anchor);
}
```

### 4. Render Context

The context provides access to the renderer and global state:

```zig
const RenderContext = struct {
    hwnd: HWND,
    root: *Node,
    width: i32,
    height: i32,

    pub fn requestRender(self: *RenderContext) void {
        // Trigger a repaint of the HWND
        InvalidateRect(self.hwnd, null, false);
    }

    pub fn focusNode(self: *RenderContext, node: *Node) void {
        // Handle focus management
    }
};
```

## Minimal Reconciler Implementation

Here's the minimal TypeScript reconciler you need:

```typescript
import { createRenderer } from "solid-js/universal"

type ZigHandle = number

export const { render, createComponent, insert } = createRenderer<ZigHandle>({
  createElement(tag: string): ZigHandle {
    return zigBridge.createElement(tag, generateId())
  },

  createTextNode(value: string): ZigHandle {
    return zigBridge.createTextNode(value, generateId())
  },

  insertNode(parent: ZigHandle, node: ZigHandle, anchor?: ZigHandle): void {
    zigBridge.insertNode(parent, node, anchor)
  },

  removeNode(parent: ZigHandle, node: ZigHandle): void {
    zigBridge.removeNode(parent, node)
  },

  setProperty(node: ZigHandle, name: string, value: any, prev: any): void {
    zigBridge.setProperty(node, name, value)
  },

  replaceText(node: ZigHandle, value: string): void {
    zigBridge.replaceText(node, value)
  },

  isTextNode(node: ZigHandle): boolean {
    return zigBridge.isTextNode(node)
  },

  getParentNode(node: ZigHandle): ZigHandle | undefined {
    return zigBridge.getParentNode(node)
  },

  getFirstChild(node: ZigHandle): ZigHandle | undefined {
    return zigBridge.getFirstChild(node)
  },

  getNextSibling(node: ZigHandle): ZigHandle | undefined {
    return zigBridge.getNextSibling(node)
  },
})
```

## JSX Type Definitions

Define what JSX elements are valid:

```typescript
// jsx-runtime.d.ts
declare namespace JSX {
  type Element = ZigHandle | string | number | boolean | null | undefined

  interface IntrinsicElements {
    box: BoxProps
    text: TextProps
    button: ButtonProps
    input: InputProps
    // ... your Zig components
  }
}

interface BoxProps {
  style?: {
    width?: number | string
    height?: number | string
    backgroundColor?: string
    flexDirection?: "row" | "column"
    // ... layout props
  }
  children?: JSX.Element
}
```

## Event Handling

Events flow from native → TypeScript:

```zig
// Zig: When a button is clicked
pub fn handleClick(node: *Node) void {
    if (node.onClick) |callback| {
        // Call back into TypeScript
        js_callback(node.handle, "click", .{});
    }
}
```

```typescript
// TypeScript: In setProperty
setProperty(node, name, value, prev) {
  if (name.startsWith("on")) {
    const eventName = name.slice(2).toLowerCase()
    zigBridge.addEventListener(node, eventName, value)
  }
}
```

## Summary: The Bridge Pattern

1. **TSX** is the declarative syntax for describing UI
2. **Solid.js** provides reactivity (signals) and the rendering algorithm
3. **The Reconciler** translates Solid's operations into your native API
4. **Native Components** (Zig) handle actual rendering and layout

The beauty of this pattern is that your UI logic (components, state, effects) lives entirely in TypeScript/TSX, while the heavy lifting of rendering happens in Zig. Changes to the UI tree are communicated through a minimal set of operations:

- `createElement(tag)` → Create a native node
- `insertNode(parent, child, anchor)` → Add to tree
- `removeNode(parent, child)` → Remove from tree
- `setProperty(node, name, value)` → Update properties
- `replaceText(node, value)` → Update text content

This is the same pattern native mobile frameworks use to bridge declarative code to platform views.
