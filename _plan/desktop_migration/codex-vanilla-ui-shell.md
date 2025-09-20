# Vanilla UI Shell Plan

## Layout Targets
- **Primary Split**: left sidebar (block palette + workflow list), central canvas, right inspector.
- **Global Header**: minimal (workspace title, run button, status indicator).
- **Modal Layer**: single overlay container managed via CSS classes (no portal abstraction needed).

## DOM Structure Skeleton
```html
<body>
  <header id="top-bar">
    <button id="run-workflow">Run</button>
    <span id="workflow-title"></span>
    <div id="status-indicator"></div>
  </header>
  <main id="layout">
    <aside id="sidebar">
      <section id="workflow-list"></section>
      <section id="block-palette"></section>
    </aside>
    <section id="canvas-container"></section>
    <aside id="inspector"></aside>
  </main>
  <div id="modal-root" hidden></div>
</body>
```

## State Handling
- Use a tiny event bus (Publish/Subscribe) implemented in `state.js` to coordinate modules; avoids global objects.
- Core state slices: `workflows` (metadata + active workflow), `canvasSelection`, `executionStatus`, `settings`.
- Persist UI preferences (theme, zoom, last opened workflow) to `localStorage`; business data lives in SQLite through commands.

## Module Responsibilities
- `app.js`: boot sequence (load workflows, render lists, register listeners, hydrate canvas module).
- `sidebar.js`: handles workflow list CRUD and block search. Emits `workflow:selected`, `block:add-request` events.
- `inspector.js`: renders form fields for selected block using declarative templates, writing updates back through `canvas` module.
- `status-bar.js` (optional small module): shows run state, token usage, last save timestamp.

## Styling Approach
- CSS custom properties for colors / spacing; responsive layout using CSS Grid for main area.
- Use `prefers-color-scheme` to auto-switch light/dark; allow manual override stored in settings slice.
- Reuse utility classes for spacing/typography to avoid reintroducing Tailwind.

## Accessibility & Keyboard
- Ensure focus management for sidebar + canvas (arrow keys to navigate list, `Delete` to remove nodes, `Ctrl+S` to save via IPC).
- Provide non-mouse canvas controls: arrow keys pan, `Ctrl +/-` zoom, `Space` for drag mode toggle.

## Asset Strategy
- Icons: inline SVG sprite file (`assets/icons.svg`) referenced with `<use>`. No icon fonts.
- Fonts: bundle system fonts first; optionally ship inter/jetbrains as static `.woff2` in `src/fonts`.

## Testing Hooks
- Add `data-testid` attributes on pivotal elements (run button, palette items, inspector forms) to facilitate later integration tests with Playwright (desktop automation).
