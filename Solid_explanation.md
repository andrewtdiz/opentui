# Using SolidJS TSX as a Bridge to Native UI Components

This document explains how OpenTUI's `@opentui/solid` package uses SolidJS's reactive primitives and custom rendering to bridge TSX/JavaScript UI logic to a native component system. The goal is to provide a blueprint for porting this architecture to a Zig-based native UI framework.

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│  TSX Layer (User Code)                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │  <box style={{ border: true, padding: 1 }}>                         │ │
│  │    <text>Hello World</text>                                         │ │
│  │  </box>                                                             │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────┬─────────────────────────────────┘
                                         │ JSX Transform
                                         ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  Reconciler (Custom SolidJS Renderer)                                    │
│  - createElement(tagName) → Instantiates native Renderable               │
│  - setProperty(node, name, value) → Sets properties on Renderable        │
│  - insertNode(parent, child) → Manages parent-child tree                 │
│  - removeNode(parent, child) → Removes from tree                         │
└────────────────────────────────────────┬─────────────────────────────────┘
                                         │ Direct Method Calls
                                         ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  Native Component Layer (Renderables)                                    │
│  - BoxRenderable, TextRenderable, InputRenderable, etc.                  │
│  - Yoga layout engine integration                                        │
│  - Event handling (mouse, keyboard, focus)                               │
│  - Render to buffer                                                      │
└────────────────────────────────────────┬─────────────────────────────────┘
                                         │
                                         ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  Render Context / Engine                                                 │
│  - Composites all renderables into final output                          │
│  - Manages render loop, dirty tracking, layout passes                    │
└──────────────────────────────────────────────────────────────────────────┘
```

## Core Concepts

### 1. The Custom Renderer

SolidJS allows creating custom renderers through its Universal Renderer API. The key file is `src/renderer/universal.js`, which is a DOM-agnostic reconciler that handles:

- **Expression insertion**: Tracking reactive values and updating nodes
- **Array reconciliation**: Efficient diffing for lists
- **Property spreading**: Applying props to nodes
- **Lifecycle management**: Creation/destruction of nodes

The reconciler is configured with callbacks that define HOW to manipulate your native components:

```typescript
createRenderer<DomNode>({
  // Create a native component instance by tag name
  createElement(tagName: string): DomNode { ... },
  
  // Create a text node
  createTextNode(value: string): DomNode { ... },
  
  // Set a property on a node
  setProperty(node: DomNode, name: string, value: any, prev: any): void { ... },
  
  // Insert child into parent (optionally before an anchor)
  insertNode(parent: DomNode, node: DomNode, anchor?: DomNode): void { ... },
  
  // Remove child from parent
  removeNode(parent: DomNode, node: DomNode): void { ... },
  
  // Tree traversal helpers
  getParentNode(node: DomNode): DomNode | undefined { ... },
  getFirstChild(node: DomNode): DomNode | undefined { ... },
  getNextSibling(node: DomNode): DomNode | undefined { ... },
})
```

### 2. Component Catalogue

Components are registered in a catalogue that maps tag names to renderable constructors:

```typescript
// src/elements/index.ts
export const baseComponents = {
  box: BoxRenderable,
  text: TextRenderable,
  input: InputRenderable,
  select: SelectRenderable,
  // ... more components
}

// Users can extend with custom components:
export function extend<T extends ComponentCatalogue>(objects: T): void {
  Object.assign(componentCatalogue, objects)
}
```

When `createElement("box")` is called, it looks up `componentCatalogue["box"]` and instantiates that class.

### 3. Renderable Base Class

Every native component extends `Renderable` (or `BaseRenderable`). Key responsibilities:

```typescript
abstract class Renderable extends BaseRenderable {
  // Identity
  protected _id: string
  public parent: Renderable | null = null
  
  // Layout (Yoga integration)
  protected yogaNode: YogaNode
  protected _x, _y, _width, _height: number
  
  // Visibility/State
  protected _visible: boolean
  protected _focused: boolean
  
  // Child management
  public add(child: Renderable, index?: number): number
  public remove(id: string): void
  public getChildren(): Renderable[]
  
  // Layout integration
  public getLayoutNode(): YogaNode
  public updateFromLayout(): void
  
  // Rendering
  protected renderSelf(buffer: OptimizedBuffer, deltaTime: number): void
  public render(buffer: OptimizedBuffer, deltaTime: number): void
  
  // Events
  public focus(): void
  public blur(): void
  public processMouseEvent(event: MouseEvent): void
  
  // Lifecycle
  public destroy(): void
  public requestRender(): void
}
```

### 4. Property Setting

The reconciler's `setProperty` function is the bridge between TSX props and native component properties:

```typescript
setProperty(node: DomNode, name: string, value: any, prev: any): void {
  // Handle events (on:click, onChange, etc.)
  if (name.startsWith("on:")) {
    const eventName = name.slice(3)
    if (value) node.on(eventName, value)
    if (prev) node.off(eventName, prev)
    return
  }

  // Handle style prop (spread style object to individual properties)
  if (name === "style") {
    for (const prop in value) {
      node[prop] = value[prop]
    }
    return
  }

  // Handle special props
  switch (name) {
    case "focused":
      value ? node.focus() : node.blur()
      break
    default:
      // Direct property assignment
      node[name] = value
  }
}
```

### 5. Render Entry Point

The main `render` function sets up the reactive root:

```typescript
export const render = async (node: () => JSX.Element, config = {}) => {
  // 1. Create the native renderer (manages render loop, input, etc.)
  const renderer = await createCliRenderer(config)
  
  // 2. Start the render engine
  engine.attach(renderer)
  
  // 3. Render the component tree into the native root
  const dispose = renderInternal(
    () => createComponent(RendererContext.Provider, {
      value: renderer,  // Provide context for useRenderer()
      children: createComponent(node, {})
    }),
    renderer.root  // The native root renderable
  )
}
```

## Key Implementation Details

### Tree Operations

```typescript
// INSERT: Add child to parent
function insertNode(parent: DomNode, node: DomNode, anchor?: DomNode): void {
  if (!anchor) {
    parent.add(node)  // Append at end
  } else {
    const anchorIndex = parent.getChildren().findIndex(el => el.id === anchor.id)
    parent.add(node, anchorIndex)  // Insert before anchor
  }
}

// REMOVE: Remove child from parent
function removeNode(parent: DomNode, node: DomNode): void {
  parent.remove(node.id)
  // Cleanup destroyed nodes
  process.nextTick(() => {
    if (!node.parent) {
      node.destroyRecursively()
    }
  })
}
```

### Reactive Updates

SolidJS's fine-grained reactivity means only changed properties trigger updates:

```tsx
const [count, setCount] = createSignal(0)

// The `text` property only updates when count() changes
<text content={`Count: ${count()}`} />
```

The reconciler wraps reactive expressions in `createRenderEffect`:

```typescript
// From universal.js
createRenderEffect(() => {
  setProperty(node, prop, props[prop], prevProps[prop])
  prevProps[prop] = value
})
```

### Hooks for Native Access

```typescript
// Provide native renderer via context
export const RendererContext = createContext<CliRenderer>()

// Access native renderer in components
export const useRenderer = () => {
  return useContext(RendererContext)
}

// Hook into keyboard events
export const useKeyboard = (callback: (key: KeyEvent) => void) => {
  const renderer = useRenderer()
  onMount(() => renderer.keyInput.on("keypress", callback))
  onCleanup(() => renderer.keyInput.off("keypress", callback))
}
```

## Porting to Zig HWND Framework

To port this architecture to a Zig-based native UI framework:

### 1. Define Your Native Component Interface

```zig
// Equivalent to BaseRenderable
pub const NativeComponent = struct {
    id: []const u8,
    parent: ?*NativeComponent = null,
    children: std.ArrayList(*NativeComponent),
    
    // Properties that TSX can set
    x: i32 = 0,
    y: i32 = 0,
    width: i32 = 0,
    height: i32 = 0,
    visible: bool = true,
    
    // HWND handle
    hwnd: ?HWND = null,
    
    // Virtual methods
    vtable: *const VTable,
    
    pub const VTable = struct {
        render: *const fn(*NativeComponent) void,
        destroy: *const fn(*NativeComponent) void,
        setProperty: *const fn(*NativeComponent, []const u8, Value) void,
    };
    
    pub fn add(self: *NativeComponent, child: *NativeComponent, index: ?usize) void { ... }
    pub fn remove(self: *NativeComponent, id: []const u8) void { ... }
};
```

### 2. Create a TypeScript FFI Bridge

```typescript
// bridge.ts - Interface to Zig via FFI (Bun.ffi, Node N-API, or WASM)

const zig = loadZigLibrary("./libui.so")

// Map tag names to Zig component types
const componentTypes = {
  window: 0,
  button: 1,
  text: 2,
  // ...
}

export function createElement(tagName: string, id: string): number {
  const typeId = componentTypes[tagName]
  return zig.create_component(typeId, id)
}

export function setProperty(handle: number, name: string, value: any): void {
  zig.set_property(handle, name, serializeValue(value))
}

export function insertChild(parentHandle: number, childHandle: number, index?: number): void {
  zig.insert_child(parentHandle, childHandle, index ?? -1)
}

export function removeChild(parentHandle: number, childHandle: number): void {
  zig.remove_child(parentHandle, childHandle)
}
```

### 3. Configure SolidJS Custom Renderer

```typescript
// reconciler.ts
import { createRenderer } from "./renderer/universal.js"
import * as bridge from "./bridge"

type DomNode = { handle: number; id: string }

export const { render, createElement, ... } = createRenderer<DomNode>({
  createElement(tagName: string): DomNode {
    const id = generateId(tagName)
    const handle = bridge.createElement(tagName, id)
    return { handle, id }
  },

  createTextNode(value: string): DomNode {
    const id = generateId("text")
    const handle = bridge.createElement("text", id)
    bridge.setProperty(handle, "content", value)
    return { handle, id }
  },

  setProperty(node: DomNode, name: string, value: any, prev: any): void {
    bridge.setProperty(node.handle, name, value)
  },

  insertNode(parent: DomNode, node: DomNode, anchor?: DomNode): void {
    const anchorIndex = anchor ? bridge.getChildIndex(parent.handle, anchor.handle) : undefined
    bridge.insertChild(parent.handle, node.handle, anchorIndex)
  },

  removeNode(parent: DomNode, node: DomNode): void {
    bridge.removeChild(parent.handle, node.handle)
  },

  getParentNode(node: DomNode): DomNode | undefined {
    const parentHandle = bridge.getParent(node.handle)
    return parentHandle ? { handle: parentHandle, id: bridge.getId(parentHandle) } : undefined
  },

  getFirstChild(node: DomNode): DomNode | undefined {
    const childHandle = bridge.getFirstChild(node.handle)
    return childHandle ? { handle: childHandle, id: bridge.getId(childHandle) } : undefined
  },

  getNextSibling(node: DomNode): DomNode | undefined {
    const siblingHandle = bridge.getNextSibling(node.handle)
    return siblingHandle ? { handle: siblingHandle, id: bridge.getId(siblingHandle) } : undefined
  },
})
```

### 4. Implement Zig Side

```zig
// libui.zig
export fn create_component(type_id: u32, id_ptr: [*]const u8, id_len: usize) usize {
    const id = id_ptr[0..id_len];
    const component = switch (type_id) {
        0 => WindowComponent.create(id),
        1 => ButtonComponent.create(id),
        2 => TextComponent.create(id),
        else => return 0,
    };
    return @intFromPtr(component);
}

export fn set_property(handle: usize, name_ptr: [*]const u8, name_len: usize, value_ptr: [*]const u8, value_len: usize) void {
    const component = @ptrFromInt(*NativeComponent, handle);
    const name = name_ptr[0..name_len];
    const value = deserialize(value_ptr[0..value_len]);
    component.vtable.setProperty(component, name, value);
}

export fn insert_child(parent_handle: usize, child_handle: usize, index: i32) void {
    const parent = @ptrFromInt(*NativeComponent, parent_handle);
    const child = @ptrFromInt(*NativeComponent, child_handle);
    parent.add(child, if (index < 0) null else @intCast(index));
}
```

## Summary

The key insight is that **SolidJS doesn't care what the underlying nodes are**. The custom renderer just needs to implement a few operations:

| Operation | Purpose |
|-----------|---------|
| `createElement(tag)` | Instantiate a native component |
| `createTextNode(text)` | Create text content |
| `setProperty(node, name, value)` | Update component properties |
| `insertNode(parent, child, anchor?)` | Add child to parent |
| `removeNode(parent, child)` | Remove child from parent |
| `getParentNode(node)` | Traverse up the tree |
| `getFirstChild(node)` | Traverse down the tree |
| `getNextSibling(node)` | Traverse horizontally |

Everything else—reactivity, component composition, control flow (`<Show>`, `<For>`)—is handled by SolidJS automatically. Your Zig layer only needs to:

1. **Maintain a component tree** (parent/child relationships)
2. **Expose property setters** (width, height, color, etc.)
3. **Handle events** (pass native events back to JS callbacks)
4. **Render** (draw to HWND/buffer when dirty)

The FFI bridge (Bun.ffi, N-API, or WASM) is the connection layer that translates between the JavaScript reconciler and your Zig component system.

