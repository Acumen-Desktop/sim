# V2 Canvas Implementation: Hybrid High-Performance Canvas

## Design Philosophy

This V2 canvas combines **Codex's hybrid rendering approach** (SVG edges + HTML nodes) with **Gemini's SVG-first philosophy** and **Claude's vanilla JavaScript principles**. The result is a high-performance, maintainable canvas that leverages the best of both vector and DOM rendering.

## Hybrid Rendering Strategy

### Why Hybrid Approach?

**SVG for Connections (Vector Graphics):**
- Crisp lines at any zoom level
- Efficient path rendering
- Built-in hit testing
- CSS-stylable strokes

**HTML Divs for Blocks (Rich Content):**
- Complex layouts with CSS Grid/Flexbox
- Easy text rendering and wrapping
- Form controls and inputs
- Better accessibility support
- Simpler event handling

### Canvas Architecture

```html
<div id="canvas-container">
    <!-- SVG Layer for connections and background -->
    <svg id="canvas-svg" class="canvas-layer">
        <defs>
            <!-- Grid pattern -->
            <pattern id="grid" width="20" height="20" patternUnits="userSpaceOnUse">
                <path d="M 20 0 L 0 0 0 20" fill="none" stroke="#e0e0e0" stroke-width="1"/>
            </pattern>

            <!-- Connection arrow markers -->
            <marker id="arrowhead" markerWidth="10" markerHeight="7"
                    refX="9" refY="3.5" orient="auto">
                <polygon points="0 0, 10 3.5, 0 7" fill="#666"/>
            </marker>
        </defs>

        <!-- Background grid -->
        <rect width="100%" height="100%" fill="url(#grid)"/>

        <!-- Connections group -->
        <g id="connections-layer" class="canvas-content"></g>

        <!-- Connection preview (while dragging) -->
        <g id="connection-preview"></g>
    </svg>

    <!-- HTML Layer for blocks -->
    <div id="blocks-layer" class="canvas-layer canvas-content">
        <!-- Blocks will be absolutely positioned here -->
    </div>

    <!-- UI overlays -->
    <div id="canvas-ui" class="canvas-layer">
        <!-- Selection box, minimap, etc. -->
    </div>
</div>
```

## Data Model: Compatible with Existing Format

**Adopting Codex's compatible data structure:**

```typescript
interface CanvasNode {
    id: string;
    type: string;
    x: number;
    y: number;
    width: number;
    height: number;
    data: Record<string, any>;

    // Runtime properties
    selected?: boolean;
    dragging?: boolean;
    element?: HTMLElement;
}

interface CanvasEdge {
    id: string;
    source: string;
    target: string;
    sourceHandle?: string;
    targetHandle?: string;

    // Runtime properties
    element?: SVGPathElement;
    selected?: boolean;
}

interface Viewport {
    x: number;
    y: number;
    scale: number;
}

interface CanvasState {
    nodes: Map<string, CanvasNode>;
    edges: Map<string, CanvasEdge>;
    selection: Set<string>;
    viewport: Viewport;

    // Interaction state
    dragState: DragState | null;
    connectionState: ConnectionState | null;
}
```

## Core Canvas Implementation

### Canvas Manager Class

```javascript
class WorkflowCanvas {
    constructor(containerId) {
        this.container = document.getElementById(containerId);
        this.svg = this.container.querySelector('#canvas-svg');
        this.connectionsLayer = this.container.querySelector('#connections-layer');
        this.blocksLayer = this.container.querySelector('#blocks-layer');
        this.uiLayer = this.container.querySelector('#canvas-ui');

        this.state = {
            nodes: new Map(),
            edges: new Map(),
            selection: new Set(),
            viewport: { x: 0, y: 0, scale: 1 },
            dragState: null,
            connectionState: null
        };

        this.blocks = new Map(); // Block instances
        this.connections = new Map(); // Connection instances

        this.initializeEventListeners();
        this.setupPerformanceOptimizations();
    }

    initializeEventListeners() {
        // Pan and zoom on SVG
        this.svg.addEventListener('wheel', this.handleWheel.bind(this));
        this.svg.addEventListener('mousedown', this.handleCanvasMouseDown.bind(this));

        // Global mouse events for dragging
        document.addEventListener('mousemove', this.handleMouseMove.bind(this));
        document.addEventListener('mouseup', this.handleMouseUp.bind(this));

        // Keyboard shortcuts
        document.addEventListener('keydown', this.handleKeyDown.bind(this));
    }

    setupPerformanceOptimizations() {
        // Use RAF for smooth updates
        this.rafId = null;
        this.pendingUpdate = false;

        // Throttle expensive operations
        this.updateTransform = this.throttle(this.updateTransform.bind(this), 16); // 60fps
        this.updateConnections = this.debounce(this.updateConnections.bind(this), 10);
    }
}
```

### Block Rendering (HTML)

```javascript
class WorkflowBlock {
    constructor(node, canvas) {
        this.node = node;
        this.canvas = canvas;
        this.element = null;
        this.ports = new Map();
    }

    render() {
        const block = document.createElement('div');
        block.className = 'workflow-block';
        block.dataset.blockId = this.node.id;
        block.dataset.blockType = this.node.type;

        // Position the block
        block.style.position = 'absolute';
        block.style.left = `${this.node.x}px`;
        block.style.top = `${this.node.y}px`;
        block.style.width = `${this.node.width}px`;
        block.style.minHeight = `${this.node.height}px`;

        // Block structure
        block.innerHTML = `
            <div class="block-header">
                <div class="block-icon">${this.getBlockIcon()}</div>
                <div class="block-title">${this.node.data.label || this.node.type}</div>
                <div class="block-menu">⋮</div>
            </div>
            <div class="block-content">
                ${this.renderBlockContent()}
            </div>
            <div class="block-ports">
                ${this.renderPorts()}
            </div>
        `;

        this.element = block;
        this.attachEventListeners();

        return block;
    }

    renderBlockContent() {
        // Render block-specific content based on type
        const descriptor = BlockRegistry.getDescriptor(this.node.type);

        let content = '<div class="block-inputs">';
        descriptor.inputs.forEach(input => {
            content += this.renderInput(input);
        });
        content += '</div>';

        content += '<div class="block-outputs">';
        descriptor.outputs.forEach(output => {
            content += this.renderOutput(output);
        });
        content += '</div>';

        return content;
    }

    renderPorts() {
        const descriptor = BlockRegistry.getDescriptor(this.node.type);
        let portsHtml = '';

        // Input ports (left side)
        descriptor.inputs.forEach((input, index) => {
            portsHtml += `
                <div class="port input-port"
                     data-port-id="${input.id}"
                     data-port-type="input"
                     style="top: ${this.calculatePortPosition('input', index)}px">
                </div>
            `;
        });

        // Output ports (right side)
        descriptor.outputs.forEach((output, index) => {
            portsHtml += `
                <div class="port output-port"
                     data-port-id="${output.id}"
                     data-port-type="output"
                     style="top: ${this.calculatePortPosition('output', index)}px; right: -6px;">
                </div>
            `;
        });

        return portsHtml;
    }

    attachEventListeners() {
        // Block dragging
        const header = this.element.querySelector('.block-header');
        header.addEventListener('mousedown', this.handleDragStart.bind(this));

        // Port connection
        const ports = this.element.querySelectorAll('.port');
        ports.forEach(port => {
            port.addEventListener('mousedown', this.handlePortMouseDown.bind(this));
        });

        // Block selection
        this.element.addEventListener('click', this.handleClick.bind(this));
    }
}
```

### Connection Rendering (SVG)

```javascript
class WorkflowConnection {
    constructor(edge, canvas) {
        this.edge = edge;
        this.canvas = canvas;
        this.element = null;
    }

    render() {
        const path = document.createElementNS('http://www.w3.org/2000/svg', 'path');
        path.setAttribute('class', 'workflow-connection');
        path.setAttribute('data-connection-id', this.edge.id);
        path.setAttribute('marker-end', 'url(#arrowhead)');

        this.updatePath();

        this.element = path;
        this.attachEventListeners();

        return path;
    }

    updatePath() {
        const sourceNode = this.canvas.state.nodes.get(this.edge.source);
        const targetNode = this.canvas.state.nodes.get(this.edge.target);

        if (!sourceNode || !targetNode) return;

        const sourcePoint = this.getPortPosition(sourceNode, this.edge.sourceHandle, 'output');
        const targetPoint = this.getPortPosition(targetNode, this.edge.targetHandle, 'input');

        const pathData = this.createBezierPath(sourcePoint, targetPoint);
        this.element.setAttribute('d', pathData);
    }

    createBezierPath(from, to) {
        const dx = to.x - from.x;
        const dy = to.y - from.y;

        // Dynamic control point offset based on distance
        const controlOffset = Math.max(50, Math.min(200, Math.abs(dx) * 0.5));

        const c1x = from.x + controlOffset;
        const c1y = from.y;
        const c2x = to.x - controlOffset;
        const c2y = to.y;

        return `M ${from.x} ${from.y} C ${c1x} ${c1y} ${c2x} ${c2y} ${to.x} ${to.y}`;
    }

    getPortPosition(node, portId, portType) {
        const blockElement = this.canvas.blocks.get(node.id)?.element;
        if (!blockElement) return { x: node.x, y: node.y };

        const port = blockElement.querySelector(`[data-port-id="${portId}"]`);
        if (!port) {
            // Fallback to block center
            return {
                x: node.x + (portType === 'output' ? node.width : 0),
                y: node.y + node.height / 2
            };
        }

        const portRect = port.getBoundingClientRect();
        const containerRect = this.canvas.container.getBoundingClientRect();

        return {
            x: portRect.left - containerRect.left + portRect.width / 2,
            y: portRect.top - containerRect.top + portRect.height / 2
        };
    }
}
```

## Interaction System

### Pan and Zoom

```javascript
handleWheel(event) {
    event.preventDefault();

    const delta = event.deltaY > 0 ? 0.9 : 1.1;
    const newScale = Math.max(0.1, Math.min(3, this.state.viewport.scale * delta));

    // Zoom towards mouse position
    const rect = this.svg.getBoundingClientRect();
    const mouseX = event.clientX - rect.left;
    const mouseY = event.clientY - rect.top;

    const scaleChange = newScale / this.state.viewport.scale;

    this.state.viewport.x = mouseX - (mouseX - this.state.viewport.x) * scaleChange;
    this.state.viewport.y = mouseY - (mouseY - this.state.viewport.y) * scaleChange;
    this.state.viewport.scale = newScale;

    this.scheduleUpdate();
}

handleCanvasMouseDown(event) {
    if (event.target === this.svg || event.target.tagName === 'rect') {
        this.state.dragState = {
            type: 'pan',
            startX: event.clientX,
            startY: event.clientY,
            startViewport: { ...this.state.viewport }
        };
    }
}

handleMouseMove(event) {
    if (!this.state.dragState) return;

    switch (this.state.dragState.type) {
        case 'pan':
            this.handlePan(event);
            break;
        case 'block':
            this.handleBlockDrag(event);
            break;
        case 'connection':
            this.handleConnectionDrag(event);
            break;
    }
}
```

### Block Dragging

```javascript
handleBlockDrag(event) {
    const { blockId, startX, startY, startPosition } = this.state.dragState;

    const dx = event.clientX - startX;
    const dy = event.clientY - startY;

    const newX = startPosition.x + dx / this.state.viewport.scale;
    const newY = startPosition.y + dy / this.state.viewport.scale;

    // Snap to grid
    const gridSize = 20;
    const snappedX = Math.round(newX / gridSize) * gridSize;
    const snappedY = Math.round(newY / gridSize) * gridSize;

    // Update node position
    const node = this.state.nodes.get(blockId);
    node.x = snappedX;
    node.y = snappedY;

    // Update block element
    const block = this.blocks.get(blockId);
    block.element.style.left = `${snappedX}px`;
    block.element.style.top = `${snappedY}px`;

    // Update connected edges
    this.updateConnections(blockId);
}
```

### Connection Creation

```javascript
handlePortMouseDown(event) {
    event.stopPropagation();

    const port = event.target;
    const portType = port.dataset.portType;
    const portId = port.dataset.portId;
    const blockId = port.closest('.workflow-block').dataset.blockId;

    if (portType === 'output') {
        // Start connection from output port
        this.state.connectionState = {
            type: 'creating',
            sourceBlock: blockId,
            sourcePort: portId,
            mouseX: event.clientX,
            mouseY: event.clientY
        };

        this.showConnectionPreview();
    }
}

updateConnectionPreview(mouseX, mouseY) {
    if (!this.state.connectionState) return;

    const sourceNode = this.state.nodes.get(this.state.connectionState.sourceBlock);
    const sourcePoint = this.getPortPosition(sourceNode, this.state.connectionState.sourcePort, 'output');

    const rect = this.container.getBoundingClientRect();
    const targetPoint = {
        x: mouseX - rect.left,
        y: mouseY - rect.top
    };

    const pathData = this.createBezierPath(sourcePoint, targetPoint);

    let preview = this.container.querySelector('#connection-preview path');
    if (!preview) {
        preview = document.createElementNS('http://www.w3.org/2000/svg', 'path');
        preview.setAttribute('class', 'connection-preview');
        this.container.querySelector('#connection-preview').appendChild(preview);
    }

    preview.setAttribute('d', pathData);
}
```

## Performance Optimizations

### Virtualization for Large Workflows

```javascript
class CanvasVirtualization {
    constructor(canvas) {
        this.canvas = canvas;
        this.visibleBounds = { x: 0, y: 0, width: 0, height: 0 };
        this.renderBuffer = 100; // pixels of buffer around visible area
    }

    updateVisibleBounds() {
        const { viewport } = this.canvas.state;
        const containerRect = this.canvas.container.getBoundingClientRect();

        this.visibleBounds = {
            x: -viewport.x / viewport.scale - this.renderBuffer,
            y: -viewport.y / viewport.scale - this.renderBuffer,
            width: containerRect.width / viewport.scale + 2 * this.renderBuffer,
            height: containerRect.height / viewport.scale + 2 * this.renderBuffer
        };
    }

    shouldRenderNode(node) {
        return (
            node.x + node.width >= this.visibleBounds.x &&
            node.x <= this.visibleBounds.x + this.visibleBounds.width &&
            node.y + node.height >= this.visibleBounds.y &&
            node.y <= this.visibleBounds.y + this.visibleBounds.height
        );
    }

    cullNodes() {
        this.canvas.state.nodes.forEach((node, id) => {
            const block = this.canvas.blocks.get(id);
            if (!block) return;

            const shouldRender = this.shouldRenderNode(node);
            const isCurrentlyVisible = block.element.style.display !== 'none';

            if (shouldRender && !isCurrentlyVisible) {
                block.element.style.display = 'block';
            } else if (!shouldRender && isCurrentlyVisible) {
                block.element.style.display = 'none';
            }
        });
    }
}
```

### Efficient Updates

```javascript
class CanvasUpdateManager {
    constructor(canvas) {
        this.canvas = canvas;
        this.pendingUpdates = new Set();
        this.rafId = null;
    }

    scheduleUpdate(type = 'full') {
        this.pendingUpdates.add(type);

        if (!this.rafId) {
            this.rafId = requestAnimationFrame(() => {
                this.processUpdates();
                this.rafId = null;
            });
        }
    }

    processUpdates() {
        if (this.pendingUpdates.has('viewport')) {
            this.updateViewport();
        }

        if (this.pendingUpdates.has('nodes')) {
            this.updateNodes();
        }

        if (this.pendingUpdates.has('connections')) {
            this.updateAllConnections();
        }

        if (this.pendingUpdates.has('full')) {
            this.updateViewport();
            this.updateNodes();
            this.updateAllConnections();
        }

        this.pendingUpdates.clear();
    }

    updateViewport() {
        const { x, y, scale } = this.canvas.state.viewport;
        const transform = `translate(${x}px, ${y}px) scale(${scale})`;

        // Update all canvas content layers
        this.canvas.container.querySelectorAll('.canvas-content').forEach(layer => {
            layer.style.transform = transform;
        });
    }
}
```

## CSS Styling

```css
/* Canvas container */
.canvas-container {
    position: relative;
    width: 100%;
    height: 100vh;
    overflow: hidden;
    background: #f8f9fa;
    cursor: grab;
}

.canvas-container.panning {
    cursor: grabbing;
}

/* Canvas layers */
.canvas-layer {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
}

.canvas-layer * {
    pointer-events: auto;
}

/* SVG elements */
#canvas-svg {
    pointer-events: all;
}

.workflow-connection {
    fill: none;
    stroke: #6c757d;
    stroke-width: 2;
    cursor: pointer;
}

.workflow-connection:hover {
    stroke: #007bff;
    stroke-width: 3;
}

.connection-preview {
    fill: none;
    stroke: #007bff;
    stroke-width: 2;
    stroke-dasharray: 5,5;
    opacity: 0.7;
}

/* Block elements */
.workflow-block {
    background: white;
    border: 1px solid #dee2e6;
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    cursor: move;
    min-width: 200px;
    user-select: none;
}

.workflow-block:hover {
    box-shadow: 0 4px 8px rgba(0,0,0,0.15);
}

.workflow-block.selected {
    border-color: #007bff;
    box-shadow: 0 0 0 2px rgba(0,123,255,0.25);
}

.block-header {
    display: flex;
    align-items: center;
    padding: 8px 12px;
    background: #f8f9fa;
    border-bottom: 1px solid #dee2e6;
    border-radius: 8px 8px 0 0;
}

.block-title {
    flex: 1;
    font-weight: 600;
    font-size: 14px;
}

.block-content {
    padding: 12px;
}

/* Ports */
.port {
    position: absolute;
    width: 12px;
    height: 12px;
    background: #6c757d;
    border: 2px solid white;
    border-radius: 50%;
    cursor: crosshair;
    z-index: 10;
}

.port:hover {
    background: #007bff;
    transform: scale(1.2);
}

.input-port {
    left: -6px;
}

.output-port {
    right: -6px;
}
```

## Integration with Backend

```javascript
// Canvas state persistence
class CanvasPersistence {
    constructor(canvas) {
        this.canvas = canvas;
        this.saveTimeout = null;
    }

    scheduleAutoSave() {
        if (this.saveTimeout) {
            clearTimeout(this.saveTimeout);
        }

        this.saveTimeout = setTimeout(() => {
            this.saveWorkflow();
        }, 2000); // Auto-save after 2 seconds of inactivity
    }

    async saveWorkflow() {
        const workflowData = this.exportWorkflow();

        try {
            await IPC.invoke('save_workflow', {
                id: this.canvas.workflowId,
                data: workflowData
            });
        } catch (error) {
            console.error('Failed to save workflow:', error);
            // Show user notification
        }
    }

    exportWorkflow() {
        const nodes = Array.from(this.canvas.state.nodes.values()).map(node => ({
            id: node.id,
            type: node.type,
            position: { x: node.x, y: node.y },
            data: node.data
        }));

        const edges = Array.from(this.canvas.state.edges.values()).map(edge => ({
            id: edge.id,
            source: edge.source,
            target: edge.target,
            sourceHandle: edge.sourceHandle,
            targetHandle: edge.targetHandle
        }));

        return {
            nodes,
            edges,
            viewport: this.canvas.state.viewport
        };
    }
}
```

## Benefits of V2 Canvas Design

### Performance Benefits
- **Hybrid Rendering**: Best performance for each element type
- **Virtualization**: Handles large workflows efficiently
- **RAF Updates**: Smooth 60fps interactions
- **Efficient DOM**: Minimal layout thrashing

### Maintainability Benefits
- **Clear Architecture**: Separate concerns for rendering and interaction
- **Vanilla JavaScript**: No framework dependencies
- **Modular Design**: Easy to test and extend
- **Standard APIs**: Uses familiar DOM and SVG APIs

### User Experience Benefits
- **Responsive**: Smooth interactions at any scale
- **Intuitive**: Familiar drag-and-drop patterns
- **Accessible**: Proper keyboard and screen reader support
- **Visual Feedback**: Clear interaction states and animations

This V2 canvas design provides a robust, performant foundation that combines the best aspects of all three original approaches while maintaining the simplicity needed for reliable desktop operation.