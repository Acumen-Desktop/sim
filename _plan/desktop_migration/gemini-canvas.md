# Gemini's Canvas Plan for Sim AI Desktop

## 1. Philosophy: Lightweight, Performant, and Maintainable

We will create a custom canvas implementation using vanilla JavaScript and SVG, as proposed by Claude. This approach avoids the overhead and complexity of a library like ReactFlow, giving us full control over performance and features. The data model will be compatible with the existing Sim AI serialization format, as suggested by Codex, to facilitate workflow import/export.

## 2. Core Technology: SVG

We will use SVG (Scalable Vector Graphics) for rendering the canvas. This is the ideal choice for this application because:

*   **DOM-Based:** Each block and connection is a DOM element, making event handling (clicks, drags, etc.) straightforward.
*   **CSS Stylable:** We can style the canvas elements using standard CSS, which is great for theming and visual feedback.
*   **Scalable:** As a vector format, the canvas will look crisp at any zoom level.
*   **Accessible:** SVG has better accessibility properties than a raster-based canvas.

## 3. Canvas Data Model

To ensure compatibility, we will adopt the data model suggested by Codex, which is based on the existing serialization format.

```typescript
interface CanvasNode {
  id: string;
  type: string;
  x: number;
  y: number;
  width: number;
  height: number;
  data: Record<string, any>;
}

interface CanvasEdge {
  id: string;
  source: string;
  target: string;
  sourceHandle?: string;
  targetHandle?: string;
}

interface CanvasState {
  nodes: Map<string, CanvasNode>;
  edges: Map<string, CanvasEdge>;
  selection: Set<string>;
  viewport: { x: number; y: number; scale: number };
}
```

This data structure will be the single source of truth for the canvas state, managed by a dedicated JavaScript module.

## 4. Key Features and Implementation

### HTML Structure

The canvas will be built on a simple HTML structure, as outlined by Claude.

```html
<div id="canvas-container">
    <svg id="workflow-canvas" width="100%" height="100%">
        <defs>
            <pattern id="grid" width="20" height="20" patternUnits="userSpaceOnUse">
                <path d="M 20 0 L 0 0 0 20" fill="none" stroke="#e0e0e0" stroke-width="1"/>
            </pattern>
        </defs>
        <rect width="100%" height="100%" fill="url(#grid)" />
        <g id="canvas-content">
            <g id="edges-layer"></g>
            <g id="nodes-layer"></g>
        </g>
    </svg>
</div>
```

### Pan and Zoom

*   Panning will be implemented by listening for mouse down/move/up events on the canvas background and updating the `transform` attribute of the `#canvas-content` group.
*   Zooming will be handled via the mouse wheel event, adjusting the `scale` of the `#canvas-content` group. The zoom will be centered on the mouse position for a natural feel.

### Blocks (Nodes)

*   Blocks will be rendered as `<g>` elements within the `#nodes-layer`.
*   Each block will be an instance of a `WorkflowBlock` class, responsible for its own rendering and event handling.
*   Dragging will be implemented by tracking mouse events on the block element.

### Connections (Edges)

*   Connections will be rendered as `<path>` elements within the `#edges-layer`.
*   The path data will be calculated as a Bezier curve between the source and target ports.
*   A `Connection` class will manage the rendering and updating of connections.

### Interaction

*   **Connecting Blocks:** A new connection will be initiated by clicking on a block's output port. A "ghost" connection will follow the mouse until the user clicks on a valid input port.
*   **Selection:** Single blocks can be selected by clicking on them. Multi-selection will be supported by holding `Shift` and clicking, or by drawing a selection box.
*   **Configuration:** Double-clicking a block or selecting it and viewing an inspector panel will allow the user to configure its properties.

## 5. Performance Considerations

*   **Virtualization:** For very large workflows, we will implement a simple form of virtualization, only rendering the blocks and connections that are currently in the viewport.
*   **Debouncing:** Canvas state changes that trigger backend saves will be debounced to avoid excessive IPC calls.
*   **Efficient DOM Updates:** We will use `requestAnimationFrame` for smooth animations and batch DOM updates where possible.

## 6. Integration with the Backend

The canvas module will communicate with the Rust backend via the `ipc.js` module.

*   `save_workflow`: When the canvas state changes, the `CanvasState` will be serialized to JSON and sent to the backend to be saved in the SQLite database.
*   `load_workflow`: The frontend will be able to request a workflow from the backend. The canvas module will then clear its current state and render the new workflow from the loaded data.

This plan provides a clear path to a performant and maintainable canvas, striking the right balance between simplicity and functionality.
