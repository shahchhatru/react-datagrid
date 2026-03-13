# `@your-org/datagrid` — Product DataGrid Component Library

## Roadmap & Architecture README

**Timeline:** 6 Months (26 Weeks)
**Stack:** React 18+, TanStack Table v8, React DnD, TypeScript
**Goal:** A composable, headless-first DataGrid library that ships table, card, and kanban views with column-level filtering, export, aggregation, and drag-and-drop — usable across any product domain.

---

## 1. The Core Idea

Most table libraries give you a table. This library gives you a **data surface** — the same dataset can render as a sortable table, a product card grid, or a kanban board, and the user switches between them without losing filter/sort/search state. TanStack Table handles the data engine (sorting, filtering, pagination, grouping). We build the **view layer**, **interaction layer**, and **plugin system** on top.

---

## 2. What Lives in the Library vs. What Stays App-Specific

This is the most important architectural decision. The rule is simple: **if it's about data shape or interaction pattern, it's library. If it's about domain meaning or visual branding, it's app-specific.**

### Library Scope (the npm package)

| Feature | Description | Rationale |
|---|---|---|
| `<DataGrid>` wrapper | Orchestrates TanStack Table instance, manages view state, passes context to child views | Core shell — every consumer needs this |
| **Table View** | Classic row/column table with sticky headers, virtual scrolling, resize handles | Baseline view, domain-agnostic |
| **Card View** | Grid of cards rendered via a `renderCard(row)` slot — consumer provides JSX, library handles layout and responsive columns | The *slot pattern* is library; the card *template* is app code |
| **Kanban View** | Columns derived from a specified field. Cards are draggable between columns. Calls `onFieldChange(rowId, field, newValue)` on drop | The board mechanics are generic; what "status" means is app-specific |
| **Column Filter Popup** | Click column header → popup with text search + unique-value checklist + filter type selector (contains, equals, gt/lt for numbers) | AG Grid parity feature — works for any column type |
| **Global Search** | Single input that fuzzy-matches across all (or specified) columns with debounce | Universal, no domain knowledge needed |
| **Column Visibility Panel** | Drawer/popover listing all columns with toggles to show/hide | Generic metadata operation |
| **Column Reorder** | Drag column headers to reposition via React DnD | Pure positional operation |
| **Sort Controls** | Click header to cycle asc/desc/none, multi-sort with shift-click | TanStack built-in, we surface the UI |
| **Aggregation Footer** | Configurable per-column: sum, avg, count, min, max, countDistinct. Renders in a sticky footer row | Engine is generic; *which* columns to aggregate is consumer config |
| **Export Engine** | `exportCSV(data, columns)`, `exportJSON(data)`, `exportPDF(data, columns, options)` — respects current filters/sort | Utility functions, domain-agnostic |
| **Row Selection** | Checkbox column, select-all, `onSelectionChange` callback | Standard pattern |
| **Pagination** | Page-based or infinite scroll, configurable page sizes | TanStack built-in, we build the UI controls |
| **Empty/Loading/Error States** | Slot-based: `renderEmpty`, `renderLoading`, `renderError` | Pattern is library; content is app-specific |
| **Density Toggle** | Compact / Default / Comfortable row heights | Generic UI preference |
| **Keyboard Navigation** | Arrow keys between cells, Enter to expand, Escape to close popups | Accessibility primitive |
| **Theming** | CSS custom properties for all colors/spacing/radii, shipped with a dark and light preset | Tokens are library; brand override is app-specific |

### App-Specific Code (consumer provides)

| Concern | Why It's App-Specific |
|---|---|
| **Product Card Template** | The JSX for rendering a product card (image, name, price, badges) depends on your data model and brand |
| **Column Definitions** | Which fields exist, their headers, their widths, their cell formatters — this is your schema |
| **Kanban Lane Field** | Which column drives kanban grouping (e.g., `status`, `category`) and what the valid values are |
| **`onFieldChange` Handler** | The mutation logic when a kanban card is dropped — calling your API, optimistic update strategy |
| **Filter Validation** | Domain rules like "price must be > 0" or "date must be in range" |
| **Export Branding** | PDF header/footer with your logo, custom column formatting in Excel |
| **Aggregation Selection** | Deciding that `price` gets `sum` and `rating` gets `avg` — that's domain knowledge |
| **Row Click / Action Handlers** | What happens when someone clicks a row or hits an action button |
| **Data Fetching** | Server-side pagination, API calls, caching — bring your own React Query / SWR |

### The Gray Zone (Library Hooks, App Composition)

Some features sit at the boundary. We expose these as **hooks** that the library ships but the app composes:

```
useDataGridExport(tableInstance)     → returns { exportCSV, exportPDF, exportJSON }
useColumnFilter(column)              → returns { filterValue, setFilter, uniqueValues, filterType }
useKanbanDrop(config)                → returns { onDragEnd, lanes, moveCard }
useAggregation(data, config)         → returns { aggregations, setAggFn }
useViewSwitcher(views)               → returns { currentView, setView, availableViews }
```

This way, if a consumer only needs the table + export, they don't ship kanban DnD code. Tree-shaking friendly.

---

## 3. Architecture Layers

```
┌─────────────────────────────────────────────────────────┐
│                    APP LAYER (Consumer)                  │
│  Column defs, Card template, Kanban config, Handlers    │
├─────────────────────────────────────────────────────────┤
│                   VIEW LAYER (Library)                   │
│  <TableView>  <CardView>  <KanbanView>                  │
│  <ColumnFilterPopup>  <ColumnVisibilityPanel>           │
│  <GlobalSearchInput>  <AggregationFooter>               │
│  <ExportMenu>  <DensityToggle>  <ViewSwitcher>          │
├─────────────────────────────────────────────────────────┤
│                 INTERACTION LAYER (Library)              │
│  useKanbanDrop    useColumnReorder   useRowSelection     │
│  useDataGridExport  useAggregation   useViewSwitcher     │
├─────────────────────────────────────────────────────────┤
│                  DATA ENGINE (TanStack Table)            │
│  Sorting, Filtering, Pagination, Grouping, Expanding    │
├─────────────────────────────────────────────────────────┤
│                  DnD ENGINE (React DnD / dnd-kit)       │
│  Column reorder, Kanban card drag, Row reorder          │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Component API Design (The Contract)

### `<DataGrid>` — Root Component

```tsx
<DataGrid
  data={products}
  columns={columnDefs}
  defaultView="table"                         // "table" | "card" | "kanban"
  views={["table", "card", "kanban"]}         // which views to enable
  globalSearchEnabled={true}
  globalSearchPlaceholder="Search products..."
  globalSearchColumns={["name", "sku", "category"]}

  // Card View
  renderCard={(row) => <ProductCard product={row.original} />}
  cardColumns={{ sm: 1, md: 2, lg: 3, xl: 4 }}

  // Kanban View
  kanbanField="status"
  kanbanLanes={["Draft", "Active", "Archived"]}
  onKanbanDrop={(rowId, newValue) => updateProduct(rowId, { status: newValue })}
  renderKanbanCard={(row) => <MiniProductCard product={row.original} />}

  // Aggregation
  aggregations={{ price: "sum", rating: "avg", stock: "sum" }}
  renderAggregation={(colId, { fn, value }) => `${fn}: ${value.toFixed(2)}`}

  // Export
  exportFilename="products-export"
  onExportCSV={true}
  onExportPDF={true}
  onExportJSON={true}

  // Events
  onRowClick={(row) => navigate(`/products/${row.original.id}`)}
  onSelectionChange={(selectedRows) => setBulkSelection(selectedRows)}

  // Slots
  renderEmpty={() => <EmptyState />}
  renderLoading={() => <Skeleton />}
  renderToolbar={(props) => <CustomToolbar {...props} />}

  // Theming
  density="default"                           // "compact" | "default" | "comfortable"
  theme="dark"
/>
```

### Column Definition (Extended TanStack)

```tsx
const columns = [
  {
    accessorKey: "name",
    header: "Product Name",
    size: 280,
    enableColumnFilter: true,
    filterFn: "includesString",               // TanStack built-in or custom
    enableSorting: true,
    enableHiding: false,                      // always visible
    meta: {
      filterType: "text",                     // "text" | "number" | "select" | "date" | "boolean"
      exportFormatter: (val) => val,          // custom export formatting
      aggregatable: false,                    // not meaningful to aggregate
    },
    cell: ({ row }) => (                      // custom cell renderer
      <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
        <img src={row.original.image} width={32} height={32} />
        <span>{row.original.name}</span>
      </div>
    ),
  },
  {
    accessorKey: "price",
    header: "Price",
    size: 120,
    enableColumnFilter: true,
    meta: { filterType: "number", aggregatable: true },
    cell: ({ getValue }) => `$${getValue().toFixed(2)}`,
  },
  // ... more columns
];
```

---

## 5. Key Technical Decisions

### Why TanStack Table (not AG Grid)

AG Grid is a full widget — you get everything but own nothing. TanStack Table is **headless** — it gives you the engine and you build the UI. For a component library meant to be themed, slotted, and extended, headless is the only sane choice. AG Grid's styling overrides are a maintenance nightmare at scale.

### Why React DnD (or dnd-kit)

**Recommendation: Start with `@dnd-kit/core`** over React DnD. dnd-kit is lighter, has better touch support, better accessibility (keyboard DnD built-in), and a cleaner API. React DnD works but carries legacy HTML5 backend complexity. Either works for this library — dnd-kit is the 2025+ default.

### Why CSS Custom Properties (not Tailwind for the library)

Tailwind is excellent for apps. It's terrible for libraries. A library consumer may not use Tailwind, or may use a different config. CSS custom properties (`--dg-color-bg`, `--dg-radius-md`) let any consumer override the theme from plain CSS, Tailwind `@layer`, CSS-in-JS, or a `<style>` tag. Ship a `.css` file with sensible defaults. Consumers override with their own variables.

### State Management: Controlled vs. Uncontrolled

Every stateful feature follows the React pattern: uncontrolled by default, optionally controlled.

```tsx
// Uncontrolled — library manages sort state internally
<DataGrid data={data} columns={columns} />

// Controlled — app owns the state
const [sorting, setSorting] = useState([]);
<DataGrid data={data} columns={columns} sorting={sorting} onSortingChange={setSorting} />
```

This applies to: sorting, filtering, column visibility, column order, pagination, selected rows, current view, density, and global search value.

---

## 6. The 6-Month Roadmap

### Month 1 — Foundation & Table View (Weeks 1–4)

**Goal:** Ship a working `<DataGrid>` with table view, sorting, and pagination.

| Week | Deliverable |
|---|---|
| 1 | Project scaffold: monorepo (Turborepo), TypeScript config, Storybook, Vitest, CSS token file, CI pipeline (lint + type-check + test on PR) |
| 2 | `<DataGrid>` shell: TanStack Table integration, column definitions, basic table rendering with sticky header |
| 3 | Sorting (click headers, multi-sort), pagination (page controls + page size selector), row selection (checkbox column) |
| 4 | Theming system: CSS custom properties, dark/light presets, density toggle (compact/default/comfortable). First Storybook stories. |

**Exit Criteria:** A consumer can `npm install`, pass data + columns, and get a sortable, paginated, themed table.

### Month 2 — Filtering & Search (Weeks 5–8)

**Goal:** Every column is individually searchable, plus global search works.

| Week | Deliverable |
|---|---|
| 5 | Column filter infrastructure: click header → popup appears. Text filter type working. |
| 6 | Number filters (gt, lt, eq, between), select filters (unique value checklist), date filters (date range picker) |
| 7 | Global search: debounced input, fuzzy match across configured columns, highlight matches in cells |
| 8 | Filter state serialization (URL params or JSON for persistence), filter presets (save/load named filter configs). Polish & edge cases. |

**Exit Criteria:** Click any column header → popup with relevant filter type. Global search works. All filter state is controllable from the app.

### Month 3 — Card View, Column Management & Export (Weeks 9–13)

**Goal:** Card view rendering, column show/hide/reorder, CSV/JSON/PDF export.

| Week | Deliverable |
|---|---|
| 9 | `<CardView>` component: `renderCard` slot, responsive CSS grid (container queries), shared filter/sort state with table view |
| 10 | View switcher UI: toggle between table and card. State persists across views. Animation on switch. |
| 11 | Column visibility panel (drawer with toggles), column reorder via dnd-kit (drag header handles) |
| 12 | Export engine: `exportCSV` and `exportJSON` utilities that respect current filters/sort/column visibility |
| 13 | PDF export via `jsPDF` + `jspdf-autotable`. Export menu UI. Aggregation footer (sum/avg/count/min/max per column). |

**Exit Criteria:** Three working views (table, card, + placeholder kanban). Columns can be hidden, reordered, exported. Aggregation row visible.

### Month 4 — Kanban View & DnD (Weeks 14–17)

**Goal:** Full kanban board with drag-and-drop that calls back to the app.

| Week | Deliverable |
|---|---|
| 14 | `<KanbanView>` layout: lanes derived from `kanbanField`, cards rendered via `renderKanbanCard` slot |
| 15 | Drag-and-drop between lanes using dnd-kit. On drop → call `onKanbanDrop(rowId, newLaneValue)`. Optimistic UI update. |
| 16 | Kanban enhancements: lane sorting, lane collapse, card count per lane, WIP limits (optional), drag handles |
| 17 | Cross-view consistency: filters/search apply to kanban. Sort order within lanes. Empty lane states. |

**Exit Criteria:** A consumer can set `kanbanField="status"` and get a working board. Dragging a card between lanes fires the callback. Filters from table view carry over.

### Month 5 — Polish, Accessibility & Performance (Weeks 18–22)

**Goal:** Production-grade quality. WCAG 2.1 AA. 10k+ rows without jank.

| Week | Deliverable |
|---|---|
| 18 | Accessibility audit: ARIA roles on table (`role="grid"`, `role="row"`, `role="gridcell"`), column headers as `role="columnheader"` with `aria-sort`. Focus management in filter popups. |
| 19 | Keyboard navigation: arrow keys between cells, Enter to activate, Escape to close popups, Tab to move between interactive elements. Kanban: arrow keys between lanes, space to pick up/drop card. |
| 20 | Virtual scrolling for table view (`@tanstack/react-virtual`). Benchmark with 10k rows — target 60fps scroll. Memoization audit. |
| 21 | Animation polish: view switch transitions (Framer Motion or CSS), filter popup open/close, card enter/exit in kanban. Loading skeleton for card view. |
| 22 | Error boundaries per view. Edge cases: empty data, single column, all columns hidden, very long cell content, RTL support investigation. |

**Exit Criteria:** Lighthouse accessibility score 95+. 10k rows renders under 100ms. Full keyboard navigation. No runtime errors with adversarial inputs.

### Month 6 — Documentation, Distribution & Dogfooding (Weeks 23–26)

**Goal:** Ship v1.0.0 on npm. Full docs. At least one internal app consuming it.

| Week | Deliverable |
|---|---|
| 23 | Storybook documentation: every component has stories for each state. Interactive props panel. Copy-paste code snippets. |
| 24 | README, API reference (auto-generated from TSDoc), migration guide from AG Grid, theming guide with examples |
| 25 | npm publish pipeline: semantic versioning, changelog generation, tree-shaking verification (check bundle size per feature), peer dependency audit |
| 26 | Dogfood: integrate into one internal app (the product catalog). Collect feedback. Fix paper cuts. Tag v1.0.0. |

**Exit Criteria:** `npm install @your-org/datagrid` works. Storybook is deployed. Bundle < 40kB gzipped (core). One real app is using it.

---

## 7. Dependency Map

| Package | Purpose | Required From |
|---|---|---|
| `@tanstack/react-table` | Data engine: sort, filter, paginate, group | Month 1 |
| `@tanstack/react-virtual` | Virtual scrolling for large datasets | Month 5 |
| `@dnd-kit/core` + `@dnd-kit/sortable` | Drag-and-drop for kanban + column reorder | Month 3 |
| `jspdf` + `jspdf-autotable` | PDF export | Month 3 |
| `fuse.js` | Fuzzy search for global search | Month 2 |
| `framer-motion` (optional) | View transition animations | Month 5 |

**No other runtime dependencies.** Everything else is CSS custom properties, vanilla React, and TypeScript.

---

## 8. Folder Structure

```
packages/
  datagrid/
    src/
      core/
        DataGrid.tsx              # Root orchestrator component
        DataGridContext.tsx        # React context for shared state
        types.ts                  # All TypeScript interfaces & types
        tokens.css                # CSS custom property definitions
      views/
        TableView.tsx             # Classic table rendering
        CardView.tsx              # Grid of renderCard() slots
        KanbanView.tsx            # Lane-based board with DnD
      features/
        filtering/
          ColumnFilterPopup.tsx   # Per-column filter UI
          GlobalSearch.tsx        # Cross-column search input
          filterFns.ts            # Custom filter functions
        sorting/
          SortIndicator.tsx       # Header sort icon
        columns/
          ColumnVisibility.tsx    # Show/hide panel
          ColumnReorder.tsx       # DnD reorder logic
        export/
          exportCSV.ts
          exportJSON.ts
          exportPDF.ts
          ExportMenu.tsx          # UI dropdown for export actions
        aggregation/
          aggregationFns.ts       # sum, avg, count, min, max
          AggregationFooter.tsx   # Sticky footer row
        selection/
          RowSelection.tsx        # Checkbox column + select all
        pagination/
          PaginationControls.tsx
      hooks/
        useDataGridExport.ts
        useColumnFilter.ts
        useKanbanDrop.ts
        useAggregation.ts
        useViewSwitcher.ts
        useDebounce.ts
      themes/
        dark.css
        light.css
      index.ts                    # Public API — named exports only
    stories/                      # Storybook files
    tests/                        # Vitest unit + integration tests
    package.json
    tsconfig.json
    README.md

apps/
  product-catalog/                # Dogfood app — your product table
    src/
      components/
        ProductCard.tsx           # App-specific card template
        MiniKanbanCard.tsx        # App-specific kanban card
      config/
        columns.tsx               # App-specific column definitions
        kanbanConfig.ts           # Which field, lane order, drop handler
      pages/
        ProductsPage.tsx          # Composes <DataGrid> with app config
```

---

## 9. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Kanban DnD performance with 500+ cards | Medium | High | Virtual lanes, limit visible cards per lane, paginate within lanes |
| PDF export formatting edge cases | High | Medium | Start with CSV/JSON (simpler), add PDF last. Use jspdf-autotable which handles wrapping. Accept "good enough" PDF. |
| TanStack Table API changes in minor versions | Low | High | Pin exact version. Wrap TanStack API in our own hook layer so consumers aren't coupled to TanStack directly. |
| Consumer wants server-side filtering/sorting | High | Medium | Design filter/sort state to be serializable from day 1. Provide `manualFiltering` / `manualSorting` mode where the library sends state changes but doesn't apply them — consumer fetches new data. |
| Bundle size creep from DnD + PDF + fuzzy search | Medium | Medium | Feature-level code splitting. Core bundle = table + sort + filter. Kanban, export, fuzzy search are separate entry points. |
| Accessibility regressions as features ship | Medium | High | Add axe-core to Storybook. Every PR runs accessibility linting. Keyboard nav tests in Vitest (jsdom). |

---

## 10. Success Metrics for v1.0.0

- Bundle size: core < 25kB gzipped, full < 45kB gzipped
- Storybook: 100% component coverage with all variant stories
- Tests: 80%+ line coverage on core + features
- Accessibility: WCAG 2.1 AA, Lighthouse 95+
- Performance: 10,000 rows renders in < 100ms, 60fps scroll
- Adoption: at least 1 internal app fully migrated from AG Grid or custom table
- Developer experience: new developer can render a working table in < 5 minutes from reading docs

---

## 11. Open Questions for CTO

Before starting implementation, these need your input:

1. **Server-side vs. Client-side first?** — Should the default mode assume all data is in memory, or should we design for server-side pagination/filtering from day 1? (Recommendation: client-side first, server-side as a `manual` mode in Month 2.)

2. **PDF export fidelity** — Is "good enough" PDF (basic table layout) acceptable, or do you need pixel-perfect branded PDFs? The latter requires significantly more effort (custom PDF templates, font embedding).

3. **Kanban persistence** — When a card is dragged to a new lane, should the library handle optimistic updates + rollback on API failure, or just fire the callback and let the app manage it? (Recommendation: fire callback only — optimistic update is app concern.)

4. **Inline editing** — Is cell-level inline editing a v1 requirement or v1.1? It's a significant feature that touches every view. (Recommendation: v1.1 — ship read-only first.)

5. **Monorepo tooling** — Turborepo vs Nx? (Recommendation: Turborepo for simplicity. Nx if you have other packages to manage.)

6. **Target consumers** — Is this library internal-only, or are we planning to open-source? This affects documentation depth, API stability guarantees, and semver discipline.
