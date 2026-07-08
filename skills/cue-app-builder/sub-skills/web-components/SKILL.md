---
name: cue-web-components
description: >
  Reference for the @qaecy/cue-ui web component library. Documents available
  components, installation, and usage patterns. Load when building any UI with
  Cue components.
---

# Cue Web Components (`@qaecy/cue-ui`)

Full interactive component documentation: **https://slides.qaecy.com/components**

QAECY ships a library of ready-made web components via the `@qaecy/cue-ui`
package. Always include at least `<cue-by-cue-logo>` in every app you build.

> **⚠️ Critical rules:**
> 1. Use `<cue-entity-list>` and `<cue-entity-viewer>` directly — do not build
>    custom entity UIs.
> 2. **The `cue` SDK instance must be passed as a JS property**, not an HTML
>    attribute. Assign it only after the user is authenticated.

---

## Installation / import

```bash
# npm-based projects
npm install @qaecy/cue-ui
```

```html
<!--
  CDN — single loader script. Only the components you request are downloaded
  as separate lazy chunks.

  • Components already in the static HTML are detected automatically.
  • List components created dynamically in data-components (comma- or
    space-separated).
-->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@qaecy/cue-ui/styles.css" />
<script
  src="https://cdn.jsdelivr.net/npm/@qaecy/cue-ui/index.js"
  type="module"
  data-components="cue-entity-viewer"
></script>
```

> Always add `type="module"` to the script tag.
> Use a single `index.js` — do **not** add separate per-component script tags.
> Static tags already in the HTML are loaded automatically without needing
> `data-components`.

---

## Available components

### `<cue-by-cue-logo>`

"Built with Cue" attribution mark. **Include in every app**, typically in a
header, footer, or splash screen.

```html
<cue-by-cue-logo></cue-by-cue-logo>
```

---

### `<cue-entity-list>`

Displays a lazy-loaded list of Cue entities. Use this instead of building a
custom entity list UI. Pass the `sdkState` and `iris` as JS properties after
auth.

| Property             | Type                  | Required | Description                                                            |
| -------------------- | --------------------- | -------- | ---------------------------------------------------------------------- |
| `iris`               | `string[]`            | ✅       | Entity IRIs to display.                                                |
| `sdkState`           | `{ cue?; entities? }` | ✅       | SDK context. Provide at least `entities` (a `CueProjectEntities`).     |
| `mapboxToken`        | `string`              | ❌       | Mapbox token; only needed to show entity location data on a map.       |
| `paginationPageSize` | `number`              | ❌       | Rows per page in the grid paginator. Default `10`.                     |

Events: `rowClicked` (`EntityRow`) and `clickedOpen` (`EntityRow`).

```js
await customElements.whenDefined('cue-entity-list');

const list = document.createElement('cue-entity-list');
list.sdkState = { entities: cue.project('your-project-id').entities }; // ← JS property
list.iris = ['https://example.com/entity/uuid-1', 'https://example.com/entity/uuid-2'];
list.paginationPageSize = 25;

list.addEventListener('rowClicked', (e) => console.log('Row clicked:', e.detail));
list.addEventListener('clickedOpen', (e) => console.log('Open entity:', e.detail));

document.getElementById('list-host').appendChild(list);
```

See **https://slides.qaecy.com/components** for the full prop reference.

---

### `<cue-document-list>`

Renders a lazy-loaded document table. Pass a `projectId`, `sdkState`, and either
explicit `uuids` or one of the filter inputs below — the component fetches
metadata and categories internally.

| Property                         | Type                                                              | Required | Description                                                                                          |
| -------------------------------- | ----------------------------------------------------------------- | -------- | ---------------------------------------------------------------------------------------------------- |
| `projectId`                      | `string`                                                          | ✅       | Cue project (space) ID.                                                                              |
| `sdkState`                       | `{ cue?; view?; documents?; language?; availableContentCategories? }` | ✅   | SDK context. Provide at least one of `cue`, `view`, or `documents`.                                  |
| `uuids`                          | `string[]`                                                        | ✅\*     | Document UUIDs to display (\*or supply a filter input below).                                        |
| `suffixes`                       | `string[]`                                                        | ❌       | File extensions to filter by (e.g. `['.pdf', '.ifc']`). Auto-fetches UUIDs. Highest precedence.      |
| `mimeTypes`                      | `string[]`                                                        | ❌       | MIME types to filter by (e.g. `['application/pdf']`). Lower precedence than `suffixes`.              |
| `contentCategories`              | `string[]`                                                        | ❌       | Content-category IRIs to filter by. Lower precedence than `suffixes` / `mimeTypes`.                  |
| `simple`                         | `boolean`                                                         | ❌       | Compact list + paginator (no ag-grid / column settings). Default `false`.                            |
| `pageSize`                       | `number`                                                          | ❌       | Page size for both modes. Default `10`.                                                              |
| `prefetchPages`                  | `number`                                                          | ❌       | Lazy-load buffer multiplier (`pageSize * prefetchPages`). Default `3`.                               |
| `showOpenInDocumentViewerAction` | `boolean`                                                         | ❌       | Show "Open in document viewer" menu item. Default `false`.                                           |
| `showOpenInFileManagerAction`    | `boolean`                                                         | ❌       | Show "Open in file manager" menu item. Default `false`.                                              |
| `customMenuItems`                | `Array<{ label; icon?; action }>`                                 | ❌       | Custom context-menu entries appended after the standard menu items.                                  |

By default only the `Download` row action is shown. Events match the internal
list: `clickedDocument`, `clickedDownloadDocument`, `clickedOpen`,
`clickedOpenInDir`.

```js
await customElements.whenDefined('cue-document-list');

const list = document.createElement('cue-document-list');
list.projectId = 'your-project-id';
list.sdkState = { cue };          // ← JS property, set after auth
list.suffixes = ['.pdf', '.ifc']; // or set uuids / mimeTypes / contentCategories

list.customMenuItems = [
  { label: 'Copy UUID', icon: 'copy', action: (doc) => navigator.clipboard.writeText(doc.id) },
];

document.getElementById('list-host').appendChild(list);
```

The filter inputs make `<cue-document-list>` pair directly with
`<cue-document-charts>` — map the chart's `selected` event onto `list.suffixes`
or `list.contentCategories` (see the charts section below).

See **https://slides.qaecy.com/components** for the full prop reference.

---

### `<cue-entity-viewer>`

Renders a Cue entity in detail — relationships, documents, metadata, and
spatial data depending on the entity type. The `cue` SDK instance must be
passed as a JS property after auth.

See **https://slides.qaecy.com/components** for the full prop reference.

---

### `<cue-document-charts>` _(new in 0.0.39)_

Interactive pie / doughnut charts giving an overview of the documents in a
project. Connect it directly to the SDK and it fetches the overview itself.

Two charts are available (`mode`: `'both'` | `'suffix'` | `'content-category'`):

- **suffix** — documents grouped by MIME category (PDF, CAD, BIM, …); click a
  category to drill down to individual file extensions.
- **content-category** — documents grouped by their semantic content category.

Pass the SDK context via the `sdkState` JS property (`{ cue }`, or reuse an
existing `{ view }` / `{ documents }`) plus the `projectId`.

```js
await customElements.whenDefined('cue-document-charts');

const charts = document.createElement('cue-document-charts');
charts.projectId = 'your-project-id';
charts.sdkState = { cue };        // ← JS property, set after auth
charts.mode = 'both';             // 'both' | 'suffix' | 'content-category'
charts.stackCharts = false;       // true = vertical layout

document.getElementById('charts-host').appendChild(charts);
```

Events:

| Event              | Payload                                          | Description                          |
| ------------------ | ------------------------------------------------ | ------------------------------------ |
| `selected`         | `{ filterBy; values: string[]; label: string }`  | User clicked a chart segment.        |
| `selectionCleared` | `{ chart: 'suffix' \| 'content-category' }`      | User cleared the selection.          |

`filterBy` is `'suffix'` (dot-extensions, e.g. `['.pdf', '.xlsx']`) or
`'content-category'` (category IRIs). Wire it to a `<cue-document-list>` to
filter the list to the clicked segment:

```js
charts.addEventListener('selected', (e) => {
  const { filterBy, values } = e.detail;
  if (filterBy === 'suffix') {
    list.suffixes = values;
    list.contentCategories = undefined;
  } else {
    list.contentCategories = values;
    list.suffixes = undefined;
  }
});

charts.addEventListener('selectionCleared', () => {
  list.suffixes = undefined;
  list.contentCategories = undefined;
});
```

See **https://slides.qaecy.com/components** for the full prop reference.

---

### `<cue-sparql-table>` _(new in 0.0.39)_

A fully declarative, column-configurable table backed by SPARQL. Each column is
a SPARQL `SELECT` query whose bindings are reduced to a single cell value by a
configurable **strategy** — no imperative formatter functions needed. This makes
it flexible enough to display almost any information from the graph (e.g. a
contract table listing involved parties and expiry dates).

Required props (all JS properties):

| Property        | Type                                          | Description                                                                 |
| --------------- | --------------------------------------------- | --------------------------------------------------------------------------- |
| `sdkState`      | `{ cue: Cue; projectId: string; graphType? }` | Authenticated SDK instance + project ID.                                    |
| `columns`       | `ColumnDefSPARQL[]`                           | Column definitions. Two-way bindable — edits in the settings panel sync back.|
| `rowDefinition` | `string` _(optional)_                         | SPARQL `SELECT ?subject` returning row IRIs; enables pagination. `''` = all. |
| `pageSize`      | `number` _(optional)_                         | Rows per page when a row definition is set. Default `10`.                    |
| `graphType`     | `'fuseki' \| 'qlever'` _(optional)_           | SPARQL engine to query. Default `'fuseki'`.                                  |

Each column's query must bind `?subject` (the row IRI) and `?val` (the cell
value). Multiple `?val` bindings for the same `?subject` are collapsed via the
column **strategy**:

| Strategy | Behaviour                                              |
| -------- | ------------------------------------------------------ |
| `first`  | First binding value (default).                         |
| `join`   | Concatenate all values with a configurable `separator`.|
| `count`  | Number of distinct bindings.                           |
| `sum`    | Numeric sum of all bindings.                           |
| `min`    | Minimum value.                                         |
| `max`    | Maximum value.                                         |

Columns may also declare an optional `postProcess` step for lightweight value
transformations after the strategy is applied, e.g.
`{ type: 'formula', expression: 'ROUND(?val, 2) * 1000' }`.

The built-in **column-settings panel** (gear icon) lets users add, remove,
reorder, and edit any column's query, strategy, and post-processing at runtime.

**Example — contract table** (one row per contract; the involved parties are
joined into a single cell and the latest mentioned date is used as the expiry):

```js
await customElements.whenDefined('cue-sparql-table');

const table = document.createElement('cue-sparql-table');
table.sdkState = { cue, projectId: 'your-project-id' };   // ← JS property

table.columns = [
  {
    field: 'contract',
    headerName: 'Contract',
    query: `PREFIX qcy: <https://dev.qaecy.com/ont#>
SELECT ?subject ?val WHERE { ?subject qcy:value ?val }`,
    strategy: { type: 'first' },
  },
  {
    field: 'parties',
    headerName: 'Involved parties',
    query: `PREFIX qcy: <https://dev.qaecy.com/ont#>
PREFIX qcy-e: <https://dev.qaecy.com/enum#>
SELECT ?subject ?val WHERE {
  ?subject qcy:mentions/qcy:resolvesTo ?org .
  ?org qcy:hasEntityCategory qcy-e:Organization ; qcy:value ?val .
}`,
    strategy: { type: 'join', separator: ', ' },
  },
  {
    field: 'expires',
    headerName: 'Expires',
    query: `PREFIX qcy: <https://dev.qaecy.com/ont#>
PREFIX qcy-e: <https://dev.qaecy.com/enum#>
SELECT ?subject ?val WHERE {
  ?subject qcy:mentions/qcy:resolvesTo ?date .
  ?date qcy:hasEntityCategory qcy-e:Date ; qcy:value ?val .
}`,
    strategy: { type: 'max' },
  },
];

// One row per contract — enables pagination
table.rowDefinition = `PREFIX qcy: <https://dev.qaecy.com/ont#>
PREFIX qcy-e: <https://dev.qaecy.com/enum#>
SELECT ?subject WHERE { ?subject qcy:hasContentCategory qcy-e:Contract }`;
table.pageSize = 20;

document.getElementById('table-host').appendChild(table);
```

Events:

| Event          | Payload    | Description                                          |
| -------------- | ---------- | ---------------------------------------------------- |
| `visibleRows`  | `any[]`    | Fires whenever the set of visible rows changes.      |
| `columnDefsAG` | `ColDef[]` | Fires when the grid column definitions are computed. |

See **https://slides.qaecy.com/components** for the full prop reference.

---

## Base UI primitives (`cue-base-*`)

Beyond the SDK-wrapped components above, `@qaecy/cue-ui` also ships Cue's own
low-level UI kit — cards, buttons, form inputs, typography, tables, layout —
under the `cue-base-` prefix (deliberately different from each component's
internal Angular selector, e.g. `cue-card` → `cue-base-card`, so registering
them as custom elements never collides with tags used inside other
components).

> **Use these for all general app chrome** — cards, buttons, form fields,
> headings/body text, tables — instead of hand-rolled `<div>`/`<button>`/CSS.
> This is what keeps every Studio app visually consistent with Cue's design
> language. Reach for custom HTML/CSS only for layout/spacing glue that these
> primitives don't cover.

| Tag | Wraps | Key inputs (JS properties unless noted) | Notes |
|---|---|---|---|
| `cue-base-card` | `Card` | `variant`, `shadow`, `padded`, `scrollable`, `maxHeight` | Generic content container — the default surface for grouping content. |
| `cue-base-flexcontainer` | `FlexContainer` | `direction`, `gap`, `align`, `justify`, `wrap` | Zero-dependency flex layout primitive — prefer this over custom flex CSS. |
| `cue-base-typography` | `Typography` | `as`, `size`, `weight`, `align`, `italic`, `innerHTML` | All headings/body text — don't hardcode font sizes/weights in CSS. |
| `cue-base-button` | `Button` | `variant`, `size`, `disabled`, `loading`, `toggleButton` | Content is projected — compose with the three helpers below. |
| `cue-base-button-label` | `ButtonLabel` | `fontSize`, `italic` | Wraps a button's text in `cue-typography` automatically. |
| `cue-base-button-icon` | `ButtonIcon` | `icon` (`IconName`), `slot` | Icon glyph inside a button. |
| `cue-base-button-padder` | `ButtonPadder` | `justify`, `size` | Spacer between icon/label inside a button. |
| `cue-base-status-badge` | `StatusBadge` | `tone`, `dot` | Small status/tag pill — content projected. |
| `cue-base-table` | `Table` | `data` (`any[]`), `columnDefs` (`ColumnDef[]`, two-way), `settings` (`TableSettings`) | Generic client-side data table — distinct from `cue-sparql-table`. |
| `cue-base-input` | `Input` | `label`, `placeholder`, `type`, `required`, `error`, `icon` | Single-line text/password/email/number field. |
| `cue-base-input-multiline` | `InputMultiline` | `label`, `placeholder`, `required`, `error` | Textarea. |
| `cue-base-input-date` | `InputDate` | `label`, `placeholder`, `required`, `error` | Date picker input. |
| `cue-base-inline-input` | `InlineInput` | `placeholder`, `autofocus` | Click-to-edit inline text (emits `save`/`cancel`). |
| `cue-base-chip-input` | `ChipInput` | `label`, `placeholder`, `allowAdd`, `allowDelete` | Tag/chip list editor. |
| `cue-base-checkbox` | `Checkbox` | `label`, `color` | — |
| `cue-base-input-switch` | `Switch` | `size`, `icon`, `iconChecked` | Toggle switch. |
| `cue-base-darkmode-switch` | `DarkmodeSwitch` | `size`, `darkClass` | Toggles a dark-mode class on demand. |
| `cue-base-select` | `Select` | `options`, `labelMap`, `placeholder`, `showFilter` | Single-value dropdown. |
| `cue-base-multi-select` | `MultiSelect` | `options`, `labelMap`, `fixedValues` | Multi-value dropdown. |
| `cue-base-nested-select` | `NestedSelect` | `items` (`NestedSelectItem[]`), `multiSelect` | Hierarchical/tree dropdown. |
| `cue-base-grid-size-select` | `GridSizeSelect` | `maxCols`, `maxRows` | Spreadsheet-style row/column picker. |

Full interactive reference for every prop/event: **https://slides.qaecy.com/components**.

### Card + typography + flex layout

```html
<cue-base-flexcontainer id="host" direction="column" gap="m"></cue-base-flexcontainer>

<script type="module">
  await customElements.whenDefined('cue-base-card');
  const card = document.createElement('cue-base-card');
  card.variant = 'primary';

  const heading = document.createElement('cue-base-typography');
  heading.size = 'l';
  heading.weight = 'medium';
  heading.innerHTML = 'Section title';

  card.appendChild(heading);
  document.getElementById('host').appendChild(card);
</script>
```

### Button composition

A button's icon/label are separate elements projected as children — assemble
them explicitly rather than relying on plain text:

```html
<cue-base-button id="save-btn" variant="primary">
  <cue-base-button-padder>
    <cue-base-button-icon icon="check"></cue-base-button-icon>
  </cue-base-button-padder>
  <cue-base-button-padder>
    <cue-base-button-label>Save</cue-base-button-label>
  </cue-base-button-padder>
</cue-base-button>

<script type="module">
  document.getElementById('save-btn').addEventListener('click', () => { /* ... */ });
</script>
```

### Table

`data` and `columnDefs` are JS properties, not attributes:

```js
await customElements.whenDefined('cue-base-table');
const table = document.createElement('cue-base-table');
table.data = rows; // any[]
table.columnDefs = [
  { key: 'name', label: 'Name' },
  { key: 'status', label: 'Status' },
];
table.clickableRows = true;
table.addEventListener('clickedRow', (e) => console.log(e.detail));
document.getElementById('table-host').appendChild(table);
```

---

## General prop-passing pattern

All interactive components require the authenticated SDK instance as a JS
property:

```html
<cue-entity-viewer id="viewer"></cue-entity-viewer>

<script type="module">
  import { Cue } from '@qaecy/cue-sdk';

  const cue = new Cue();
  await cue.auth.signIn('google');

  const viewer = document.getElementById('viewer');
  viewer.cue = cue;               // ← must be set after auth
  viewer.projectId = 'proj-abc';
  // set additional props per the component docs
</script>
```
