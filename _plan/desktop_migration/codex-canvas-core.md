# Canvas Core Plan

## Goals
- Replace ReactFlow with lightweight SVG-based renderer.
- Support panning/zooming, drag-drop node creation, connection drawing, selection box, and basic alignment guides.
- Store canvas state as plain objects compatible with existing serializer shapes (nodes/edges, positions, handles).

## Data Model
```ts
interface CanvasNode {
  id: string
  type: string
  x: number
  y: number
  width: number
  height: number
  data: Record<string, any>
}

interface CanvasEdge {
  id: string
  source: string
  target: string
  sourceHandle?: string
  targetHandle?: string
}

interface CanvasState {
  nodes: Map<string, CanvasNode>
  edges: Map<string, CanvasEdge>
  selection: Set<string>
  viewport: { x: number; y: number; scale: number }
}
```

## Rendering Stack
- Use `SVG` for edges + connection previews, `HTML` absolutely-positioned `<div>`s for nodes (easier block content rendering).
- Maintain separate layers: background grid, edges, nodes, overlays.
- RequestAnimationFrame loop only when viewport or dragging changes; otherwise static DOM updates.

## Interaction Model
- Panning: track pointer events on background, update `viewport` then apply CSS transform to node/edge layers.
- Zooming: listen to wheel events, clamp scale (0.25–2.0), recalc transforms.
- Drag node: pointer down on `.node` header, capture pointer, update node coords, snap to 8px grid.
- Connect: pointer down on handle triggers ghost edge until pointer up; validate drop target before committing edge.
- Multi-select: `Shift` + drag creates selection rectangle; update `selection` set.

## Persistence Hooks
- `canvas.export()` returns arrays ready for serializer.
- `canvas.load(nodes, edges)` resets state and re-renders.
- Auto-save throttle: emit `canvas:changed` events debounced at 500ms so `app.js` can call `save_workflow`.

## Performance Considerations
- Keep DOM node count low (<200) by recycling connection previews and using `documentFragment` for batched updates.
- Avoid expensive layout thrash: apply transforms using CSS variables, not inline style recalculation per frame.
- Provide utility to freeze canvas during execution replay (render read-only state from logs).

## Testing Strategy
- Unit test geometry helpers (hit testing, edge routing) with Vitest-in-browser (later) or Rust integration tests for serializer invariants.
- Manual QA checklist: zoom stability, edge creation, undo/redo (future), keyboard nav.

## Stretch (Post-MVP)
- Mini-map overlay.
- Smart alignment lines across nodes.
- Subflow nesting collapsed into grouped nodes.
