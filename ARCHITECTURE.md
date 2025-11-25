# @opentui/solid architecture notes

This package adapts Solid’s JSX/TSX runtime to OpenTUI’s renderables so UI logic can be written declaratively while targeting a non-DOM backend. The same pattern can be used to bridge TSX into a Zig UI layer that renders native components in a window.

## High-level flow
- `render()`/`testRender()` (`packages/solid/index.ts`) create a renderer from `@opentui/core` (CLI or test), attach it to the global `engine`, and call the Solid-compatible reconciler with a provider-wrapped component tree.
- `createRenderer()` (`src/renderer/universal.js`) implements Solid’s host renderer contract: create/insert/remove nodes, set props, reconcile arrays, and run effects. It is framework-agnostic and accepts backend-specific callbacks.
- `src/reconciler.ts` supplies those callbacks for OpenTUI renderables (subclasses of `BaseRenderable`). It handles element creation, text nodes, slots, property/event mapping, and traversal helpers (`getParentNode`, `getFirstChild`, `getNextSibling`).
- `src/elements/index.ts` maps JSX intrinsic tags to concrete renderable classes from `@opentui/core` (box, text, input, etc.). The catalogue is extensible via `extend()` so new tags can be bound to custom renderables.
- `src/elements/hooks.ts` exposes renderer-aware hooks (resize, keyboard, paste, selection, timelines) by pulling the active renderer from context.
- `src/elements/extras.ts` implements Solid helpers like `Portal`, `Dynamic`, and `createDynamic`, using slots and manual insertion to render children at arbitrary mount points.
- Types in `src/types/elements.ts` wire JSX props to renderable options, filter styleable properties, and support catalogue augmentation.

## How TSX turns into backend nodes
1. Solid compiles TSX to calls into the host renderer exported from `src/reconciler.ts` (`createElement`, `createTextNode`, `insert`, `spread`, `setProp`, etc.).
2. `createElement(tag)` looks up the tag in the component catalogue and instantiates the corresponding renderable with a generated id. This is the seam where you would instantiate Zig-backed renderables instead.
3. `insertNode(parent, child, anchor?)` and `removeNode()` manipulate the renderable tree, handling special `SlotRenderable`s that stand in for holes/markers.
4. Text is represented by `TextNode` (extends `TextNodeRenderable`) with helpers for replacement and style mapping (`createTextAttributes`, `parseColor`).
5. Props/events are applied in `setProperty`:
   - `on:*` is forwarded to the renderable’s event emitter.
   - Common events (`onChange`, `onInput`, `onSubmit`, `onSelect`) map to specific renderable event enums.
   - `style` merges into renderable properties (fg/bg for text nodes, direct assignment otherwise).
   - `text`/`content` coerce values to strings.
   - `focused` drives focus/blur on focusable renderables.
6. Tree traversal helpers power Solid’s diffing (`getParentNode`, `getFirstChild`, `getNextSibling`), including handling of root text nodes.

## Slots, portals, and dynamic rendering
- `SlotRenderable` provides invisible placeholder nodes used by `cleanChildren` in the universal renderer to manage insertion points; it chooses a text or layout slot based on the parent type.
- `Portal` renders children into a specified mount (defaults to renderer root) by creating a new element subtree and inserting it manually, mirroring Solid’s web portal behavior for out-of-flow UI like modals.
- `Dynamic`/`createDynamic` allow choosing a component/tag at runtime, calling functions for custom components or creating/spreading props onto intrinsic tags.

## Renderer context and lifecycle
- `RendererContext` supplies the active `CliRenderer` to components and hooks. `render()` wraps the root element with this provider.
- Hooks (`useRenderer`, `useKeyboard`, `usePaste`, `useSelectionHandler`, `onResize`, `useTerminalDimensions`) subscribe to renderer or input events and clean up on unmount.
- `useTimeline` integrates OpenTUI timelines with Solid’s lifecycle (autoplay on mount, unregister on cleanup).
- Disposal: `render()` registers `onDestroy` to detach from the engine and dispose the Solid root.

## Typing and extensibility
- `jsx-runtime.d.ts` declares `JSX.IntrinsicElements` for built-in tags and exports the `JSX` namespace consumed by TSX.
- `ExtendedIntrinsicElements` and `OpenTUIComponents` enable module augmentation: add renderable constructors to the catalogue and augment the JSX namespace so new tags are type-safe.
- `style` prop is constrained to renderable options minus non-style keys (`NonStyledProps` + component-specific exclusions).

## Porting guidance for a Zig backend
- Keep the host renderer shape from `src/renderer/universal.js`; implement the callback surface (`createElement`, `createTextNode`, `createSlotNode`, `setProperty`, `insertNode`, `removeNode`, traversal helpers) against your Zig-owned renderable layer.
- Mirror `componentCatalogue`: expose JSX tag names that construct Zig-backed renderable handles. Provide an `extend` hook so TSX can introduce new native widgets without changing the renderer core.
- Ensure renderables expose:
  - `add/remove` with optional index/anchor semantics,
  - child traversal (`getChildren`, `parent`, `getLayoutNode` equivalents if needed),
  - text nodes that can be replaced/merged,
  - event subscription (`on/off`) for keyboard, selection, change, input, submit, select, etc.,
  - focus control (`focus/blur`) and destroy semantics.
- Replicate the property mapping logic from `setProperty`; keep event name conventions (`on:*` and specific handlers) stable so TSX props stay ergonomic.
- Implement slot/marker nodes (even as no-op placeholders) so `cleanChildren` can diff arrays and portals can target alternate mounts.
- Keep the Solid lifecycle utilities (context provider, hooks, portals) unchanged; only the renderable constructors and property/event plumbing need to call into Zig.

## Minimal end-to-end path
1. Reuse `src/renderer/universal.js` unchanged.
2. Rewrite `src/reconciler.ts` to call into Zig-backed renderable constructors and property/event shims.
3. Define a catalogue of JSX tags → Zig renderable classes/functions; export `extend` for custom widgets.
4. Keep `render()` to set up the renderer instance, provide context, and run the reconciler; swap the renderer factory to your Zig window/rendering backend.
5. Verify with `testRender()` equivalents to drive headless renderables and assert tree state without a UI surface.
