# Cue SDK — API Quick Reference

Version: `@qaecy/cue-sdk@0.0.16`
Exports: `@qaecy/cue-sdk` (browser / framework) · `@qaecy/cue-sdk/node` (Node.js)

---

## Initialization

```ts
import { Cue } from '@qaecy/cue-sdk';
const cue = new Cue();          // default QAECY environment

import { CueNode } from '@qaecy/cue-sdk/node';
const cue = new CueNode();      // Node.js — adds file-sync capabilities
```

---

## `cue.auth` — CueAuth

| Method | Signature | Notes |
|---|---|---|
| `signIn` | `(provider: 'google' \| 'microsoft') → Promise<User>` | Browser popup |
| `signIn` | `(provider: 'password', { email, password }) → Promise<User>` | Email + password |
| `signInWithApiKey` | `(apiKey: string, projectId?: string) → Promise<User>` | Non-interactive; exchanges API key for custom token |
| `signInWithRedirect` | `(provider: 'google' \| 'microsoft') → Promise<void>` | Mobile / iframe |
| `checkRedirectResult` | `() → Promise<User \| null>` | Call on page load after redirect |
| `signOut` | `() → Promise<void>` | |
| `signUp` | `(name, email) → Promise<{ uid, orgName }>` | Sends set-password email |
| `getToken` | `(forceRefresh?) → Promise<string \| null>` | Firebase ID token |
| `authenticatedFetch` | `(url, init?) → Promise<Response>` | Fetch with Bearer; auto-refreshes on 401 |
| `onAuthStateChanged` | `(listener) → Unsubscribe` | Reactive auth state |
| `checkSuperAdmin` | `() → Promise<boolean>` | One-shot superadmin check |

**Key properties:**

| Property | Type | Description |
|---|---|---|
| `currentUser` | `User \| null` | Currently signed-in user |
| `user` | `ReadonlySignal<User \| null>` | Reactive auth state |
| `token` | `ReadonlySignal<string \| null>` | Reactive ID token |
| `isSuperAdmin` | `ReadonlySignal<boolean>` | Reactive superadmin flag |

---

## `cue.projects` — CueProjects

| Method | Signature | Notes |
|---|---|---|
| `listProjects` | `() → Promise<ProjectData[]>` | All projects the user is a member of |
| `getProject` | `(projectId) → Promise<ProjectData \| null>` | Single project by ID |
| `createProject` | `(options: CreateProjectOptions) → Promise<ProjectData>` | See options below |
| `inviteUserToProject` | `(email, projectId, role) → Promise<{ uid, name? }>` | Roles: `admin \| syncer \| member` |
| `changeUserRoleOnProject` | `(uid, projectId, role) → Promise<void>` | |
| `removeUserFromProject` | `(uid, projectId) → Promise<void>` | |

**`CreateProjectOptions`:**
```ts
{
  organizationID: string;
  name: string;
  id?: string;            // defaults to UUID
  graphType?: 'fuseki' | 'qlever';
  tier?: 's' | 'm' | 'l';
}
```

---

## `cue.api` — CueApi

| Method | Signature | Notes |
|---|---|---|
| `sparql` | `(query: string, projectId: string) → Promise<SparqlJsonResult>` | Standard SPARQL 1.1 SELECT/CONSTRUCT/ASK |
| `search` | `(request: SearchRequest) → Promise<SearchResponse>` | Natural-language AI search |
| `shacl` | `(shape: string, projectId, options?) → Promise<ShaclReport \| string>` | SHACL validation |
| `setLanguage` | `(lang: string) → void` | E.g. `'en'`, `'da'` |
| `getAuthHeaders` | `() → Promise<Record<string, string>>` | Bearer headers for direct REST calls |
| `getConsumption` | `(projectId) → Promise<UnitsConsumedDto>` | Credit usage stats |

**`SearchRequest`:**
```ts
{ term: string; projectId: string; categories?: string[] }
```

**`SearchResponse`:**
```ts
{
  id: string;
  question: string;
  questionHTML: string;
  response: string;       // AI-generated answer
  sources: SearchSource[];
  rankedSources: SearchSource[];
}
```

**SPARQL result helper** (flatten bindings to plain objects):
```ts
const rows = result.results.bindings.map(b =>
  Object.fromEntries(Object.entries(b).map(([k, v]) => [k, v.value]))
);
```

---

## `CueProjectEntities` — accessed via SPARQL or future `cue.projects.entities(id)`

| Method | Signature | Notes |
|---|---|---|
| `requestEntityData` | `(uuids: string[], includeMentionCount?) → void` | Lazy batch-fetch labels + categories |
| `requestEntityLocations` | `(uuids: string[]) → Promise<void>` | Lazy fetch OSM geometry |
| `fetchEntityRelationships` | `(iri: string) → Promise<EntityRelationships>` | `{ incoming, outgoing }` |
| `fetchEntityDocuments` | `(iri: string) → Promise<string[]>` | UUIDs of referencing documents |
| `contentCategoriesInProject` | `(orderByOccurrences?) → Promise<{ iri, label }[]>` | All entity categories in project |
| `buildSummaryGraph` | `(format: 'graph' \| 'md') → Promise<SummaryGraphData \| string>` | Category relationship overview |
| `entityIri` | `(uuid: string) → string` | Constructs `https://cue.qaecy.com/r/{projectId}/{uuid}` |
| `reset` | `() → void` | Call when active project changes |

**`EntityDetailedData` shape:**
```ts
{
  value: string;                 // Entity label
  categories: string[];          // Category IRIs
  mentionCount?: number;
  documentRefs?: string[];
  relationshipData?: EntityRelationships;
  directMapGeometries?: MapGeometry[];
  indirectMapGeometries?: { … }[];
}
```

---

## `CueProjectDocuments` — accessed via SPARQL or document API

| Method | Signature | Notes |
|---|---|---|
| `requestDocumentData` | `(uuids: string[]) → void` | Lazy batch-fetch document metadata |
| `fetchOverview` | `() → Promise<void>` | Counts by suffix, content category, duplicates |
| `randomFilePath` | `() → Promise<string \| null>` | Useful for dev/debug |
| `setLanguage` | `(lang) → void` | Clears cache so language-sensitive fields are re-fetched |
| `reset` | `() → void` | Call when active project changes |

**`DocumentInfo` shape:**
```ts
{
  id: string;
  contentIRI: string;
  path: string;
  suffix: string;
  size: number;
  tags: string[];
  categories: string[];
  subject?: string;
  summary?: string;
  providerId?: string;
}
```

---

## `cue.gis` — CueGis

| Method | Signature | Notes |
|---|---|---|
| `setProjectId` | `(projectId: string \| null) → void` | Required before any GIS operation |
| `setBbox` | `(bbox: [west, south, east, north]) → void` | Map viewport; triggers category query |
| `setSelectedCategories` | `(categories: Set<FeatureCategory>) → void` | Starts loading selected categories |
| `onAvailableCategories` | `(cb: (cats: GisCategoryDescriptor[]) => void) → Unsubscribe` | Replays immediately |
| `onFeaturesChange` | `(cb: (map: GisFeaturesMap) => void) → Unsubscribe` | `Map<category, GisFeature[]>` |
| `onLoadingChange` | `(cb: (loading: boolean) => void) → Unsubscribe` | |
| `destroy` | `() → void` | Clean up all subscriptions |

---

## `CueSyncApi` — Node.js only (`cue.api.sync`)

| Method | Signature | Notes |
|---|---|---|
| `previewSync` | `(localFiles, options) → Promise<SyncPreview>` | Cost estimate without uploading |
| `sync` | `(localFiles, options) → Promise<SyncResult>` | Upload + write RDF metadata |

**`SyncOptions`:**
```ts
{
  spaceId: string;       // project ID
  providerId: string;    // e.g. 'local', 's3'
  userId: string;
  verbose?: boolean;
  onProgress?: (progress: SyncProgress) => void;
}
```

---

## RDF IRI patterns

| Pattern | Example IRI |
|---|---|
| Entity IRI | `https://cue.qaecy.com/r/{projectId}/{uuid}` |
| Ontology class | `https://dev.qaecy.com/ont#Building` |
| Enum value | `https://dev.qaecy.com/enum#Building` |
