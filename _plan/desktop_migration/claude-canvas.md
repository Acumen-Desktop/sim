# Canvas Implementation for Tauri Desktop

## Current Implementation: ReactFlow
- Heavy React-based flow library
- Complex state management with React hooks
- Bundle size overhead
- Framework coupling

## Proposed Implementation: Vanilla JavaScript Canvas

### Core Canvas Features Needed

1. **Infinite Scrollable Canvas**
2. **Drag and Drop Blocks**
3. **Connection System** (edges between blocks)
4. **Pan and Zoom**
5. **Block Selection and Editing**
6. **Visual Feedback** (hover, selection states)

### Implementation Strategy: SVG-based Approach

#### Why SVG over HTML5 Canvas?
- **DOM Integration**: Easy event handling on individual elements
- **Styling**: CSS-based styling for blocks and connections
- **Accessibility**: Screen reader compatible
- **Debugging**: Inspectable in dev tools
- **Scalability**: Vector-based, crisp at any zoom level

### Core Canvas Structure

```html
<!-- Canvas container -->
<div id="canvas-container">
    <svg id="workflow-canvas" width="100%" height="100%">
        <!-- Background grid -->
        <defs>
            <pattern id="grid" width="20" height="20" patternUnits="userSpaceOnUse">
                <path d="M 20 0 L 0 0 0 20" fill="none" stroke="#e0e0e0" stroke-width="1"/>
            </pattern>
        </defs>
        <rect width="100%" height="100%" fill="url(#grid)" />

        <!-- Canvas content group (for pan/zoom transform) -->
        <g id="canvas-content">
            <!-- Connections layer -->
            <g id="connections-layer"></g>

            <!-- Blocks layer -->
            <g id="blocks-layer"></g>
        </g>
    </svg>
</div>
```

### Block Implementation

```javascript
class WorkflowBlock {
    constructor(id, type, position) {
        this.id = id;
        this.type = type;
        this.position = position;
        this.size = { width: 200, height: 100 };
        this.element = null;
        this.isDragging = false;
    }

    render() {
        const group = document.createElementNS('http://www.w3.org/2000/svg', 'g');
        group.setAttribute('class', 'workflow-block');
        group.setAttribute('transform', `translate(${this.position.x}, ${this.position.y})`);
        group.setAttribute('data-block-id', this.id);

        // Block background
        const rect = document.createElementNS('http://www.w3.org/2000/svg', 'rect');
        rect.setAttribute('width', this.size.width);
        rect.setAttribute('height', this.size.height);
        rect.setAttribute('rx', '8');
        rect.setAttribute('class', 'block-background');

        // Block title
        const title = document.createElementNS('http://www.w3.org/2000/svg', 'text');
        title.setAttribute('x', '10');
        title.setAttribute('y', '25');
        title.setAttribute('class', 'block-title');
        title.textContent = this.type;

        // Connection points
        const inputPort = this.createPort('input', 0, this.size.height / 2);
        const outputPort = this.createPort('output', this.size.width, this.size.height / 2);

        group.appendChild(rect);
        group.appendChild(title);
        group.appendChild(inputPort);
        group.appendChild(outputPort);

        this.element = group;
        this.attachEventListeners();

        return group;
    }

    createPort(type, x, y) {
        const circle = document.createElementNS('http://www.w3.org/2000/svg', 'circle');
        circle.setAttribute('cx', x);
        circle.setAttribute('cy', y);
        circle.setAttribute('r', '6');
        circle.setAttribute('class', `port port-${type}`);
        circle.setAttribute('data-port-type', type);
        return circle;
    }

    attachEventListeners() {
        this.element.addEventListener('mousedown', this.onMouseDown.bind(this));
        this.element.addEventListener('click', this.onClick.bind(this));
    }

    onMouseDown(event) {
        if (event.target.classList.contains('port')) return; // Port clicks handled separately

        this.isDragging = true;
        this.dragStart = {
            x: event.clientX - this.position.x,
            y: event.clientY - this.position.y
        };

        document.addEventListener('mousemove', this.onMouseMove.bind(this));
        document.addEventListener('mouseup', this.onMouseUp.bind(this));

        event.preventDefault();
    }

    onMouseMove(event) {
        if (!this.isDragging) return;

        this.position.x = event.clientX - this.dragStart.x;
        this.position.y = event.clientY - this.dragStart.y;

        this.element.setAttribute('transform', `translate(${this.position.x}, ${this.position.y})`);

        // Update any connected edges
        CanvasManager.updateConnections(this.id);
    }

    onMouseUp() {
        this.isDragging = false;
        document.removeEventListener('mousemove', this.onMouseMove);
        document.removeEventListener('mouseup', this.onMouseUp);
    }

    onClick(event) {
        if (!this.isDragging) {
            // Handle block selection/configuration
            CanvasManager.selectBlock(this.id);
        }
    }
}
```

### Connection System

```javascript
class Connection {
    constructor(fromBlockId, fromPort, toBlockId, toPort) {
        this.id = `${fromBlockId}-${toBlockId}`;
        this.fromBlockId = fromBlockId;
        this.fromPort = fromPort;
        this.toBlockId = toBlockId;
        this.toPort = toPort;
        this.element = null;
    }

    render() {
        const fromBlock = CanvasManager.getBlock(this.fromBlockId);
        const toBlock = CanvasManager.getBlock(this.toBlockId);

        if (!fromBlock || !toBlock) return null;

        const fromPoint = this.getPortPosition(fromBlock, this.fromPort);
        const toPoint = this.getPortPosition(toBlock, this.toPort);

        const path = document.createElementNS('http://www.w3.org/2000/svg', 'path');
        const pathData = this.createBezierPath(fromPoint, toPoint);

        path.setAttribute('d', pathData);
        path.setAttribute('class', 'connection');
        path.setAttribute('data-connection-id', this.id);

        this.element = path;
        return path;
    }

    getPortPosition(block, portType) {
        const x = portType === 'output' ?
            block.position.x + block.size.width :
            block.position.x;
        const y = block.position.y + block.size.height / 2;
        return { x, y };
    }

    createBezierPath(from, to) {
        const dx = to.x - from.x;
        const controlOffset = Math.max(50, Math.abs(dx) * 0.5);

        const controlX1 = from.x + controlOffset;
        const controlX2 = to.x - controlOffset;

        return `M ${from.x} ${from.y} C ${controlX1} ${from.y} ${controlX2} ${to.y} ${to.x} ${to.y}`;
    }
}
```

### Canvas Manager

```javascript
class CanvasManager {
    constructor() {
        this.canvas = document.getElementById('workflow-canvas');
        this.canvasContent = document.getElementById('canvas-content');
        this.blocksLayer = document.getElementById('blocks-layer');
        this.connectionsLayer = document.getElementById('connections-layer');

        this.blocks = new Map();
        this.connections = new Map();
        this.selectedBlock = null;

        this.transform = { x: 0, y: 0, scale: 1 };
        this.isPanning = false;

        this.initializeEventListeners();
    }

    initializeEventListeners() {
        // Pan and zoom
        this.canvas.addEventListener('wheel', this.onWheel.bind(this));
        this.canvas.addEventListener('mousedown', this.onCanvasMouseDown.bind(this));

        // Connection creation
        this.canvas.addEventListener('click', this.onPortClick.bind(this));
    }

    onWheel(event) {
        event.preventDefault();

        const delta = event.deltaY > 0 ? 0.9 : 1.1;
        const newScale = Math.max(0.1, Math.min(3, this.transform.scale * delta));

        // Zoom towards mouse position
        const rect = this.canvas.getBoundingClientRect();
        const mouseX = event.clientX - rect.left;
        const mouseY = event.clientY - rect.top;

        const scaleChange = newScale / this.transform.scale;
        this.transform.x = mouseX - (mouseX - this.transform.x) * scaleChange;
        this.transform.y = mouseY - (mouseY - this.transform.y) * scaleChange;
        this.transform.scale = newScale;

        this.updateTransform();
    }

    onCanvasMouseDown(event) {
        if (event.target === this.canvas || event.target.tagName === 'rect') {
            this.isPanning = true;
            this.panStart = { x: event.clientX, y: event.clientY };

            document.addEventListener('mousemove', this.onPanMove.bind(this));
            document.addEventListener('mouseup', this.onPanEnd.bind(this));
        }
    }

    onPanMove(event) {
        if (!this.isPanning) return;

        const dx = event.clientX - this.panStart.x;
        const dy = event.clientY - this.panStart.y;

        this.transform.x += dx;
        this.transform.y += dy;

        this.panStart = { x: event.clientX, y: event.clientY };
        this.updateTransform();
    }

    onPanEnd() {
        this.isPanning = false;
        document.removeEventListener('mousemove', this.onPanMove);
        document.removeEventListener('mouseup', this.onPanEnd);
    }

    updateTransform() {
        const transform = `translate(${this.transform.x}, ${this.transform.y}) scale(${this.transform.scale})`;
        this.canvasContent.setAttribute('transform', transform);
    }

    addBlock(blockData) {
        const block = new WorkflowBlock(blockData.id, blockData.type, blockData.position);
        const element = block.render();

        this.blocks.set(blockData.id, block);
        this.blocksLayer.appendChild(element);

        return block;
    }

    addConnection(fromBlockId, toBlockId) {
        const connection = new Connection(fromBlockId, 'output', toBlockId, 'input');
        const element = connection.render();

        if (element) {
            this.connections.set(connection.id, connection);
            this.connectionsLayer.appendChild(element);
        }
    }

    updateConnections(blockId) {
        this.connections.forEach(connection => {
            if (connection.fromBlockId === blockId || connection.toBlockId === blockId) {
                const newElement = connection.render();
                if (newElement && connection.element) {
                    this.connectionsLayer.replaceChild(newElement, connection.element);
                }
            }
        });
    }

    selectBlock(blockId) {
        // Remove previous selection
        if (this.selectedBlock) {
            this.selectedBlock.element.classList.remove('selected');
        }

        const block = this.blocks.get(blockId);
        if (block) {
            block.element.classList.add('selected');
            this.selectedBlock = block;

            // Show block configuration panel
            this.showBlockConfig(block);
        }
    }

    showBlockConfig(block) {
        // Trigger Tauri command to show block configuration
        // This will open a sidebar or modal for editing block parameters
    }
}
```

### CSS Styling

```css
/* Canvas styles */
#canvas-container {
    width: 100%;
    height: 100vh;
    overflow: hidden;
    background: #f8f9fa;
    cursor: grab;
}

#canvas-container.panning {
    cursor: grabbing;
}

/* Block styles */
.workflow-block {
    cursor: move;
    transition: filter 0.2s;
}

.workflow-block:hover {
    filter: brightness(1.05);
}

.workflow-block.selected .block-background {
    stroke: #007bff;
    stroke-width: 2;
}

.block-background {
    fill: white;
    stroke: #dee2e6;
    stroke-width: 1;
    filter: drop-shadow(0 2px 4px rgba(0,0,0,0.1));
}

.block-title {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    font-size: 14px;
    font-weight: 600;
    fill: #495057;
}

/* Port styles */
.port {
    fill: #6c757d;
    stroke: white;
    stroke-width: 2;
    cursor: crosshair;
}

.port:hover {
    fill: #007bff;
    r: 8;
}

/* Connection styles */
.connection {
    fill: none;
    stroke: #6c757d;
    stroke-width: 2;
    pointer-events: stroke;
}

.connection:hover {
    stroke: #007bff;
    stroke-width: 3;
}
```

### Integration with Tauri Backend

```javascript
// Save canvas state to backend
async function saveWorkflow() {
    const workflowData = {
        blocks: Array.from(CanvasManager.blocks.values()).map(block => ({
            id: block.id,
            type: block.type,
            position: block.position,
            config: block.config
        })),
        connections: Array.from(CanvasManager.connections.values()).map(conn => ({
            from: conn.fromBlockId,
            to: conn.toBlockId
        }))
    };

    const { invoke } = window.__TAURI__.tauri;
    await invoke('save_workflow', { workflow: workflowData });
}

// Load canvas state from backend
async function loadWorkflow(workflowId) {
    const { invoke } = window.__TAURI__.tauri;
    const workflow = await invoke('load_workflow', { workflowId });

    // Clear current canvas
    CanvasManager.clear();

    // Add blocks
    workflow.blocks.forEach(blockData => {
        CanvasManager.addBlock(blockData);
    });

    // Add connections
    workflow.connections.forEach(connData => {
        CanvasManager.addConnection(connData.from, connData.to);
    });
}
```

## Benefits of This Approach

1. **Zero Dependencies**: No external libraries needed
2. **Lightweight**: Minimal code footprint
3. **Customizable**: Full control over behavior and styling
4. **Performance**: Direct DOM manipulation, no virtual DOM overhead
5. **Debuggable**: Standard web technologies, inspectable in dev tools

## Implementation Priority

1. **Phase 1**: Basic canvas with pan/zoom
2. **Phase 2**: Block rendering and dragging
3. **Phase 3**: Connection system
4. **Phase 4**: Selection and configuration
5. **Phase 5**: Keyboard shortcuts and accessibility

This vanilla implementation provides all the core functionality of ReactFlow while eliminating the framework dependency and maintaining the simplicity needed for the desktop MVP.