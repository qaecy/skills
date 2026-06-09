---
name: cue-app-builder
description: >
  Expert guide for building apps on the QAECY Cue platform using the Cue SDK.
  Use when a developer wants to create a new Cue-connected app, add Cue
  authentication or data access to an existing app, design SPARQL queries
  against a Cue knowledge graph, or work with Cue entities, documents, or GIS
  data. Triggers: Cue app, Cue SDK, QAECY, CUE_API_KEY, Cue auth, Cue SPARQL,
  build on Cue, knowledge graph app, Cue SPA.
argument-hint: '<description of the app or feature to build>'
---

# Cue App Builder

Guides an agent to build a complete app—or add a feature—using the QAECY Cue
SDK. Covers auth, project selection, SPARQL queries, entity lookups, document
access, and GIS, for any stack (vanilla JS, React, Node.js, etc.).

---

## Reference files

Load these files with `read_file` when you need them:

| File | When to load |
|---|---|
| `context/sdk-api.md` | Full SDK method reference |
| `context/ontology.md` | QAECY ontology, prefixes, and Turtle modeling examples |
| `context/sparql-patterns.md` | Ready-to-use SPARQL query patterns |
| `sub-skills/list-projects/SKILL.md` | Listing the user's projects and finding project IDs |
| `sub-skills/project-summary/SKILL.md` | Fetching the entity schema for an unfamiliar project |
| `sub-skills/query-tester/SKILL.md` | Testing a SPARQL query with `CUE_API_KEY` |
| `sub-skills/document-metadata/SKILL.md` | Document model: FileLocation vs FileContent, alternative representations (DWG→DXF, IFC→ThatOpen), and metadata SPARQL patterns |
| `sub-skills/web-components/SKILL.md` | Available `@qaecy/cue-ui` components (`<cue-by-cue-logo>`, `<cue-entity-list>`, `<cue-entity-viewer>`), installation, and usage patterns |

When building any UI, load `sub-skills/web-components/SKILL.md` for the
available `@qaecy/cue-ui` components. All components are loaded via a
**single `index.js`** bundle with the `data-components` attribute for
dynamic elements.

Always load `context/sdk-api.md` before writing any SDK code. Load the others
as they become relevant to the task.

**Recommended opening sequence for any new project:**
1. Load `sub-skills/list-projects/SKILL.md` → run it to identify the target `projectId`.
2. Load `sub-skills/project-summary/SKILL.md` → run it to understand what entity types and relationships exist in that specific project.
3. Then proceed with the steps below.

---

## Step 1 — Understand the requirements

Before asking the user anything, orient yourself against the actual project data:

1. If no `projectId` is known, invoke `sub-skills/list-projects/SKILL.md` to
   discover available projects.
2. Once a `projectId` is identified, invoke `sub-skills/project-summary/SKILL.md`
   to fetch the entity category graph. This reveals the custom schema — which
   entity types exist, how they relate, and which relationships are most
   prominent. Use this to inform query design and UI decisions.

Then clarify the remaining requirements if still ambiguous:

- What data should the app display? (entities, documents, GIS, summaries?)
- What user interaction is expected? (search, filter, click-to-detail, map?)
- What stack? (vanilla JS SPA, React/Vue/Svelte, Node.js CLI/script, other?)
- Auth model: interactive user login (SSO / email-password), or a server-side
  service account using `CUE_API_KEY`?

---

## Step 2 — SDK installation & import

**Load `context/sdk-api.md` now** to have the full API at hand.

### Browser / vanilla JS (importmap pattern)

The SDK has firebase as a peer dependency, so firebase must be available as a
bere-specifier mapping. Use the trailing-slash prefix to cover all subpaths
(`firebase/app`, `firebase/auth`, `firebase/storage`, etc.) in one entry:

```html
<script type="importmap">
{
  "imports": {
    "@qaecy/cue-sdk": "https://esm.sh/@qaecy/cue-sdk@0.0.17",
    "firebase/":      "https://esm.sh/firebase@12/"
  }
}
</script>
<script type="module">
  import { Cue } from '@qaecy/cue-sdk';
  const cue = new Cue();   // uses built-in default SDK config
  // ...
</script>
```

> **Note:** Do NOT use `?external=firebase` on the esm.sh SDK URL, and do not
> list individual `firebase/app`, `firebase/auth`, … entries. The single
> trailing-slash rule `"firebase/": "…"` handles every subpath.

### npm-based (React, Vue, Svelte, Node.js)

```bash
npm install @qaecy/cue-sdk
```

```ts
// Browser / framework
import { Cue } from '@qaecy/cue-sdk';
const cue = new Cue();

// Node.js (includes sync / file-upload capabilities)
import { CueNode } from '@qaecy/cue-sdk/node';
const cue = new CueNode();
```

No constructor argument is needed for end-user apps targeting the default QAECY
environment — the SDK ships with a built-in Firebase config. Pass a custom
`CueSdkConfig` only when deploying to a dedicated tenant.

---

## Step 3 — Authentication

### Interactive (user-facing app)

```ts
// SSO (Google or Microsoft)
await cue.auth.signIn('google');
await cue.auth.signIn('microsoft');

// Email + password
await cue.auth.signIn('password', { email: 'user@example.com', password: '…' });

// React to auth state changes
cue.auth.onAuthStateChanged(user => {
  if (user) { /* load data */ }
  else      { /* show login */ }
});

// Sign out
await cue.auth.signOut();
```

### Redirect flow (mobile / iframe contexts where popups are blocked)

```ts
await cue.auth.signInWithRedirect('google');
// On the next page load:
const user = await cue.auth.checkRedirectResult();
```

### Service / developer auth (non-interactive)

```ts
// Requires CUE_API_KEY — exchanged for a Firebase custom token internally
await cue.auth.signInWithApiKey(process.env.CUE_API_KEY, projectId);
```

Use `signInWithApiKey` in Node.js scripts, CI pipelines, or during development
when you want to avoid the browser popup. Never commit the API key to source
control — use environment variables or a secret manager.

---

## Step 4 — List and select projects

```ts
const projects = await cue.projects.listProjects();
// [{ id: 'proj-abc', name: 'My Project', … }, …]

const project = await cue.projects.getProject('proj-abc');
```

---

## Step 5 — Query the knowledge graph

Load `context/sparql-patterns.md` for copy-paste-ready queries.

### SPARQL (primary query mechanism)

```ts
const PREFIXES = `
PREFIX rdf:   <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs:  <http://www.w3.org/2000/01/rdf-schema#>
PREFIX qcy:   <https://dev.qaecy.com/ont#>
PREFIX qcy-e: <https://dev.qaecy.com/enum#>
`;

const result = await cue.api.sparql(PREFIXES + `
  SELECT ?entity ?name (COUNT(DISTINCT ?mention) AS ?count) WHERE {
    ?entity a qcy:CanonicalEntity ;
            qcy:hasEntityCategory qcy-e:Organization ;
            qcy:value ?name .
    OPTIONAL { ?mention qcy:resolvesTo ?entity }
  }
  GROUP BY ?entity ?name
  ORDER BY DESC(?count)
  LIMIT 5
`, projectId);

// Flatten the SPARQL JSON response
const rows = result.results.bindings.map(b =>
  Object.fromEntries(Object.entries(b).map(([k, v]) => [k, v.value]))
);
```

### Natural-language search

```ts
const response = await cue.api.search({
  term: 'fire safety requirements for building A',
  projectId,
  categories: ['Building'],   // optional category filter
});
// response.response — AI-generated answer
// response.sources  — document snippets used as evidence
```

### Entity summary graph

```ts
// Get a markdown table of entity-category relationships in the project
const md = await cue.projects.entities(projectId).buildSummaryGraph('md');
```

### Testing queries during development

When you need to run a SPARQL query against a real project to check results,
invoke the query-tester sub-skill:
> Load `sub-skills/query-tester/SKILL.md` and follow its procedure.

---

## Step 6 — Entity detail

Load `context/ontology.md` to understand how entities relate to documents,
fragments, and selectors.

```ts
const entities = cue.projects.entities(projectId);

// Batch-fetch entity labels + categories
entities.requestEntityData(['uuid-1', 'uuid-2'], /* includeMentionCount */ true);
const info = entities.entityInfoMap.get()['uuid-1'];
// { value: 'Acme Corp', categories: ['…Organization'], mentionCount: 42 }

// Fetch relationships
const rels = await entities.fetchEntityRelationships(entityIri);
// { incoming: […], outgoing: […] }

// Fetch which documents mention this entity
const docIds = await entities.fetchEntityDocuments(entityIri);

// Construct an entity IRI from a UUID
const iri = entities.entityIri('my-uuid');
// 'https://cue.qaecy.com/r/{projectId}/my-uuid'
```

---

## Step 7 — Documents

```ts
const docs = cue.projects.documents(projectId);

// Lazy-fetch document metadata
docs.requestDocumentData(['doc-uuid-1']);
const docInfo = docs.documentInfoMap.get()['doc-uuid-1'];
// { id, path, suffix, size, tags, categories, subject, summary, providerId }

// Project-level overview (counts by file type, content category, duplicates)
await docs.fetchOverview();
const overview = docs.projectDocumentsData.get();
```

---

## Step 8 — GIS

```ts
cue.gis.setProjectId(projectId);

// Subscribe to available feature categories for the current bbox
cue.gis.onAvailableCategories(cats => { /* render layer toggles */ });

// Set map viewport
cue.gis.setBbox([west, south, east, north]);

// Request features for selected categories
cue.gis.setSelectedCategories(new Set(['cadastre', 'building']));

// Subscribe to feature data (GeoJSON-like)
cue.gis.onFeaturesChange(featuresMap => {
  for (const [category, features] of featuresMap) { /* add to map */ }
});
```

---

## Step 9 — App structure guidance

### Vanilla JS SPA (recommended starting point)

- One `index.html` + `app.js` + `styles.css`.
- Use the `importmap` pattern (Step 2) to avoid a build step.
- Auth state drives all UI transitions via `onAuthStateChanged`.
- Keep a single `activeProject` state variable; reload data on project change.
- **Mandatory: include `<cue-by-cue-logo>` in every app** (header, footer, or splash).
- **Use built-in Cue components** — do not build custom entity list or viewer
  UIs. Always use `<cue-entity-list>` and `<cue-entity-viewer>`. Load
  `sub-skills/web-components/SKILL.md` for full usage details.
- Unless the user explicitly requests a different design system/theme, use the
  default Cue visual language: default Cue colors and Cue component styling.

### React / Vue / Svelte

- Install via npm. Wrap `onAuthStateChanged` in a context provider or store.
- Use the reactive `cue.auth.user` signal directly if your framework supports
  fine-grained reactivity.

### Node.js script / CLI

- Use `CueNode` from `@qaecy/cue-sdk/node`.
- Auth via `signInWithApiKey`.
- Ideal for data pipelines, one-off queries, and CI jobs.

---

## Cue web components (`@qaecy/cue-ui`)

Load `sub-skills/web-components/SKILL.md` for full component documentation.
Full interactive reference: **https://slides.qaecy.com/components**

Available components:

| Component | Purpose |
|---|---|
| `<cue-by-cue-logo>` | "Built with Cue" attribution mark — include in every app |
| `<cue-entity-list>` | List of Cue entities with metadata |
| `<cue-entity-viewer>` | Detailed entity view (relationships, documents, spatial data) |

