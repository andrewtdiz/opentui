# OpenTUI Solid renderer: how TSX maps to native nodes

This package wires Solid's JSX runtime into the OpenTUI render engine so TSX components instantiate and mutate native renderable objects (the things that ultimately draw in your Zig window). The pieces fit together like this:

- `packages/solid/index.ts` exposes `render`/`testRender`. `render` creates a CLI renderer from `@opentui/core`, attaches it to the global `engine`, then calls the Solid reconciler with the renderer's root. Everything is wrapped in `RendererContext.Provider` so any hook can grab the live renderer instance.
- `packages/solid/src/reconciler.ts` builds a Solid custom renderer by passing platform-specific methods into `createRenderer` (from `src/renderer/universal.js`). These methods are the bridge points:
  - `createElement(tag)` looks up the tag in a component catalogue and instantiates the matching `Renderable` class (e.g., `BoxRenderable`, `TextRenderable`) with a generated id and the current renderer context.
  - `createTextNode` and `replaceText` produce `TextNodeRenderable` wrappers for string content.
  - `setProperty` maps JSX props onto renderable fields and event emitters (`on:` props attach listeners; `style` mutates renderable properties; focused/input/select handlers map to core event enums).
  - `insertNode`, `removeNode`, and the sibling/parent lookups manipulate the render tree the way Solid expects, with special handling for `SlotRenderable` placeholders.
  - `createSlotNode` produces layout/text slot placeholders so arrays/fragments can be reconciled cleanly.
- `packages/solid/src/renderer/universal.js` contains a DOMless Solid renderer adapted from Solid's runtime. It drives reactive inserts/updates by calling the functions above instead of touching the browser DOM. Solid signals trigger `insertExpression`, which diffs arrays, replaces text, and calls `setProperty` and `insertNode`/`removeNode` to keep the native tree in sync.
- `packages/solid/src/elements/index.ts` registers the available intrinsic elements. The catalogue maps string tag names to concrete `Renderable` constructors from `@opentui/core`. Text modifiers (`<b>`, `<i>`, `<u>`, `<br>`) are thin wrappers that toggle text attributes or insert newlines. You can extend the catalogue via `extend({...})` to expose new native controls.
- `packages/solid/src/elements/hooks.ts` exposes renderer-aware hooks (`useRenderer`, `onResize`, `useKeyboard`, `usePaste`, `useSelectionHandler`, `useTimeline`) so Solid components can interact with input/output streams without importing core directly.
- `packages/solid/src/elements/extras.ts` provides utility components:
  - `Portal` renders children into another mount point (defaults to the root) by manually creating a `box` node and inserting it elsewhere—useful for overlays/modals.
  - `Dynamic`/`createDynamic` pick a component at runtime and forward props using the same element creation and `spread` logic.
- `packages/solid/src/elements/slot.ts` defines `SlotRenderable`, a non-rendering placeholder that returns either a layout slot (`LayoutSlotRenderable`) or a text slot (`TextSlotRenderable`) depending on the parent. The reconciler uses slots when cleaning/patching children so Solid's fragment/array reconciliation has stable anchors.
- `packages/solid/jsx-runtime.d.ts` reshapes `JSX.IntrinsicElements` so TSX tags resolve to the renderable catalogue and return `DomNode` instead of DOM nodes.
- Utility glue:
  - `src/utils/id-counter.ts` generates stable ids per element type for debugging.
  - `src/utils/log.ts` gates reconciler logs behind `DEBUG`.

### How to apply this pattern to a Zig-backed UI

1) **Provide renderable primitives**: Implement classes analogous to `BoxRenderable`, `TextRenderable`, etc., that proxy into your Zig hwnd-layer (create native widgets, update layout, destroy). Expose a constructor signature `(ctx, options)` so `createElement` can instantiate them with an id and renderer context.

2) **Build a component catalogue**: Register string tag names to those renderable constructors (see `componentCatalogue` in `elements/index.ts`) and export an `extend` helper so apps can add more native controls.

3) **Implement the reconciler hooks**: Supply `createElement`, `createTextNode`, `replaceText`, `setProperty`, `insertNode`, `removeNode`, and the sibling/parent getters so Solid's universal renderer can drive your native tree. Mirror the prop/event mapping in `setProperty`, translating TSX props into Zig-side property setters and event subscriptions.

4) **Wire the renderer lifecycle**: Provide a `render` entry that creates your Zig renderer (akin to `createCliRenderer`), attaches it to whatever scheduling/engine loop you have, and calls the Solid renderer's `render` with the renderer's root node. Wrap the app in a context provider to pass the renderer to hooks.

5) **Handle slots/portals**: Keep the slot placeholder pattern; it lets Solid reconcile arrays/fragments without real nodes. Portals can be reimplemented by creating a container renderable and inserting it at an alternate mount point in the native tree.

6) **TSX typing**: Define a `jsx-runtime.d.ts` that maps intrinsic elements to your catalogue types and points `JSX.Element` at your native node type so editors and the compiler understand the bridge.

Following this structure, Solid's reactive TSX logic remains unchanged, while your renderer functions translate every creation, prop change, and insertion/removal into native Zig-backed component operations.
