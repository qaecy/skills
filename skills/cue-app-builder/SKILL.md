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
- Auth model: is this a Studio app meant to run embedded inside the Cue
  portal (the default — use the `postMessage` handshake in Step 3), a
  standalone app with interactive user login (SSO / email-password), or a
  server-side service account using `CUE_API_KEY`?

---

## Step 2 — SDK installation & import

**Load `context/sdk-api.md` now** to have the full API at hand.

### Browser / vanilla JS (no build step)

Import directly from jsdelivr's dedicated `/browser.js` entry point — the
same pattern the first-party `doc-search`/`sparql-client` Studio apps use in
production (`apps/frontend/cue-portal/public/studio-apps/`). Omit the version
to always get the latest release — don't guess a version number.

```html
<script type="module">
  import { Cue } from 'https://cdn.jsdelivr.net/npm/@qaecy/cue-sdk/browser.js';
  const cue = new Cue();   // uses built-in default SDK config
  // ...
</script>
```

> **Do NOT** use an importmap + esm.sh bare-specifier setup
> (`"@qaecy/cue-sdk": "https://esm.sh/@qaecy/cue-sdk"` remapped alongside a
> `"firebase/": "https://esm.sh/firebase@12/"` entry for the peer
> dependency). That combination is unreliable — it 404s at runtime, which
> silently kills the whole module (including the `cue:ready` postMessage
> in Step 3) since it's a static `import`, leaving the app stuck on its
> "Connecting to Cue…" overlay forever. The `/browser.js` entry above has no
> peer-dependency resolution problem and needs no importmap at all.

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

**First decide which of these two modes applies — see "Auth model" in Step 1.**
Studio apps (anything built via the Cue App Builder, or otherwise intended to
run embedded inside the Cue portal) **must** use the portal handshake below.
Never show an interactive login screen (`cue.auth.signIn()`) in a Studio app
— the user is already signed in to the portal; showing a second login is a
bug, not a fallback. The "Interactive"/"Redirect" flows further down are only
for apps that run standalone, outside the portal (their own domain, a
desktop app, etc.).

### Studio apps embedded in the Cue portal (default for anything built here)

The portal hosts every Studio app — shelf apps and apps built with this
skill alike — inside an iframe and performs a `postMessage` handshake to
hand it an authenticated session, instead of making the app show its own
login UI:

```js
// 1. Signal readiness as soon as the script runs.
window.parent.postMessage({ type: 'cue:ready' }, '*');

// 2. Wait for the portal's response.
window.addEventListener('message', (event) => {
  if (event.data?.type !== 'cue:init') return;
  bootstrap(event.data);
});

async function bootstrap({ firebaseConfig, projectId, appId, customToken }) {
  const cue = new Cue(firebaseConfig);

  // The preview/runtime iframe is sandboxed (opaque origin, no shared
  // storage), so sign in with the short-lived custom token the portal
  // minted for the current user rather than waiting on a restored session.
  await cue.auth.signInWithCustomToken(customToken);

  // `projectId` is always provided — use it for every SDK call that takes one.

  // `appId` is only present once the app has been saved to the Studio
  // registry (the platform assigns and injects it — never hardcode your
  // own UUID here, unlike the standalone-app appData pattern in Step 9).
  // While still iterating on a fresh, unsaved build, `appId` is undefined —
  // skip `cue.api.appData` persistence gracefully in that case instead of
  // throwing.

  // ...render the app, reveal it, hide any loading/connecting overlay...
}
```

Notes:
- Do this unconditionally at startup — don't gate it behind a "sign in"
  button or check `cue.auth.currentUser` first.
- `doc-search`/`sparql-client` (checked into
  `apps/frontend/cue-portal/public/studio-apps/`) show a related but distinct
  variant of this same handshake: they're hand-authored, first-party files
  served *same-origin* with the portal, so they skip the token and instead
  wait on `cue.auth.onAuthStateChanged` to pick up the session Firebase
  already persisted in shared storage. Apps built by this skill always run
  in a sandboxed, opaque-origin iframe (both while previewing in the builder
  and once saved and run from the Studio gallery), so the `customToken`
  path above is the one that applies — read those two files for the rest of
  the bootstrap shape (search-bar/list/viewer wiring, the auth-overlay
  loading pattern), not for their auth mechanism specifically. If reachable,
  the same two apps are also live at `https://cue.qaecy.com/studio-apps/doc-search/index.html`
  and `.../sparql-client/index.html` — useful to skim for UI/UX conventions,
  but the checked-in source is the source of truth if that URL is ever
  unreachable or out of date.
- Keep the `#auth-overlay` / connecting-spinner pattern from those files
  while waiting for `cue:init` and for `signInWithCustomToken` to resolve.

### Interactive (user-facing app, standalone only)

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

## Step 9 — Persisting per-user app data

If your app needs to remember something for the current user across sessions —
settings, a language/theme preference, a list of saved/favorited items — use
`cue.api.appData` instead of inventing your own storage. It's small-blob
storage only (**100KB cap, enforced server-side**), not for documents, datasets,
or anything project-scoped; use `cue.projects.documents`/sync for bulk data.

**Studio apps embedded in the portal (Step 3 default):** don't hardcode a
UUID — use the `appId` the portal handshake hands you in `cue:init`. It's
`undefined` until the app is saved to the Studio registry (the platform
assigns it then), so skip `appData` calls gracefully until it's present:

```ts
if (appId) {
  const settings = await cue.api.appData.get<MySettings>(appId); // null if nothing saved yet
  await cue.api.appData.set(appId, { ...settings, favorites: [...] });
}
```

**Standalone apps (not embedded in the portal):** generate a UUID4 **once**
and hardcode it as a constant in your app — this namespaces your app's data
from every other Cue app sharing the platform. Never change it after
release, or you'll orphan any data already saved.

```ts
const APP_ID = '3fa2b1c0-58cc-4372-a567-0e02b2c3d479'; // generate once, hardcode, never change

const settings = await cue.api.appData.get<MySettings>(APP_ID); // null if nothing saved yet
await cue.api.appData.set(APP_ID, { ...settings, favorites: [...] });
```

See `context/sdk-api.md` for the full reference.

---

## Step 10 — App structure guidance

### Vanilla JS SPA (recommended starting point)

- One `index.html` + `app.js` + `styles.css`.
- Use the `importmap` pattern (Step 2) to avoid a build step.
- For Studio apps, complete the Step 3 `cue:ready`/`cue:init` handshake
  before rendering anything; `onAuthStateChanged` still fires afterwards and
  can drive further UI transitions, but never gate initial render on an
  interactive sign-in screen.
- Keep a single `activeProject` state variable; reload data on project change.
- **Mandatory: include `<cue-by-cue-logo>` in every app** (header, footer, or splash).
- **Use built-in Cue components** — do not build custom entity list or viewer
  UIs. Always use `<cue-entity-list>` and `<cue-entity-viewer>`. Load
  `sub-skills/web-components/SKILL.md` for full usage details.
- **Build general app chrome from the `cue-base-*` primitives** — cards
  (`cue-base-card`), buttons (`cue-base-button` + its label/icon/padder
  helpers), form fields (`cue-base-input`, `cue-base-select`, …), typography
  (`cue-base-typography`), tables (`cue-base-table`), and layout
  (`cue-base-flexcontainer`) — instead of hand-rolled `<div>`/`<button>`/CSS.
  Load `sub-skills/web-components/SKILL.md` for the full list and usage
  snippets. This is what keeps every Studio app visually consistent with Cue.
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

