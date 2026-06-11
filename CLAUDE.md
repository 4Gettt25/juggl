# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev      # watch mode with inline sourcemaps, copies output to docs/.obsidian/plugins/juggl
npm run build    # production build (minified via terser)
npm run release  # version bump via standard-version
```

There is no test suite. To test changes, load the plugin inside Obsidian by pointing the vault at the `docs/` folder (it contains a preconfigured `.obsidian/plugins/juggl` that `dev`/`build` copy output into).

Linting: `src/.eslintrc.js` configures ESLint with `@typescript-eslint` and `eslint-config-google` rules.

## Architecture

Juggl is an **Obsidian plugin** that renders interactive graph views using [Cytoscape.js](https://js.cytoscape.org). The entry point is `src/main.ts` (`JugglPlugin extends Plugin`), which wires up all Obsidian lifecycle hooks.

### Core object model

| Class | File | Role |
|---|---|---|
| `JugglPlugin` | `src/main.ts` | Obsidian plugin root. Owns plugin-level state, registers views/commands, exposes the public API. |
| `Juggl` | `src/viz/visualization.ts` | The graph instance. Wraps a `cytoscape.Core` and owns all graph mutation logic (expand, merge, filter, layout, stylesheet). Implements `IJuggl` from `juggl-api`. |
| `JugglView` | `src/viz/juggl-view.ts` | Obsidian `ItemView` wrapper around a `Juggl` instance. One per open leaf. |
| `ObsidianStore` | `src/obsidian-store.ts` | The built-in `ICoreDataStore`. Translates Obsidian `MetadataCache`/`Vault` events into Cytoscape node/edge definitions. Reacts to vault changes (rename, delete, metadata update) and refreshes live graphs. |
| `WorkspaceMode` | `src/viz/workspaces/workspace-mode.ts` | `IAGMode` for "workspace" graphs — persistent, user-curated graphs with save/load. Uses a radial context menu (cytoscape-cxtmenu) and a Svelte `Toolbar`. |
| `LocalMode` | `src/viz/local-mode.ts` | `IAGMode` for "local" graphs — shows the neighbourhood of a single note, depth-limited. Uses a simpler `ToolbarLocal` Svelte component. |
| `WorkspaceManager` | `src/viz/workspaces/workspace-manager.ts` | Persists named workspaces as JSON under `.obsidian/plugins/juggl/<name>/`. |
| `GraphStyleSheet` | `src/viz/stylesheet.ts` | Generates the Cytoscape stylesheet by merging a base sheet, YAML-driven per-node overrides, and a user-editable `graph.css` at `.obsidian/plugins/juggl/graph.css`. |

### Data flow

1. `JugglPlugin` holds a `coreStores` map (`{ Obsidian: ObsidianStore }`) and a `stores: IDataStore[]` array for third-party plugin stores (via the public API).
2. When a graph opens, a `Juggl` instance is constructed with an `IJugglStores` snapshot — `{ coreStore, dataStores[] }`.
3. `Juggl.expand()` calls `store.getNeighbourhood()` on each store and merges returned `NodeDefinition[]` into the Cytoscape graph, then calls `store.connectNodes()` to build edges.
4. The `mode` (`LocalMode` | `WorkspaceMode`) is a `Component` child of `Juggl` and owns all interaction logic (keyboard shortcuts, context menus, toolbar, auto-add behaviour). Modes communicate back to `Juggl` via its event emitter (`Juggl.trigger()`).
5. `ObsidianStore` listens to `MetadataCache.on('changed')` and `Vault.on('rename'|'delete')` and calls `refreshNode()` on all active graphs.

### Public API surface

External Obsidian plugins extend Juggl via `juggl-api` (a separate npm package at `github:HEmile/juggl-api`). The key contracts are:

- `IJugglPlugin.registerStore(store: IDataStore)` — add extra data sources
- `IJugglPlugin.createJuggl(el, settings?, datastores?, initialNodes?)` — embed a graph programmatically
- `IJugglPlugin.registerEvents(handler: IJugglEvents)` — lifecycle hooks (`onJugglCreated`, `onJugglDestroyed`)
- `VizId` (from `juggl-api`) — canonical node identity: `{ id: string, storeId: string }`. The built-in store uses `storeId = 'core'`.

### Styling pipeline

Node appearance flows from three layered sources (last wins):
1. Programmatic base styles in `GraphStyleSheet.getStylesheet()`
2. YAML frontmatter fields (`color`, `shape`, `image`, `width`, `height`, `title`) — applied via the `YAML_MODIFY_SHEET` selector block
3. User's `graph.css` (`.obsidian/plugins/juggl/graph.css`), hot-reloaded via a `vault.on('raw')` watcher

Style groups (from Settings or the Style Pane) assign CSS classes `global-N` / `local-N` to nodes matched by a filter expression, which the stylesheet uses to set colors/shapes/icons.

### UI components

Svelte is used for the Settings tab appearance panel (`AppearanceSettings.svelte`), the toolbar in workspace mode (`Toolbar.svelte`), and the toolbar in local mode (`ToolbarLocal.svelte`). These are bundled by `rollup-plugin-svelte` — no separate Svelte compilation step.

The two side panes (Nodes Pane, Style Pane) live in `src/pane/view.ts` as standard Obsidian `ItemView`s.

### Build output

Rollup bundles everything into a single `main.js` (CJS format, `obsidian` kept external) in the repo root. `rollup-plugin-copy` then copies `main.js` and `styles.css` into `docs/.obsidian/plugins/juggl/` for in-vault testing.
