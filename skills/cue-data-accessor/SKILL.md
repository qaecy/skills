---
name: cue-data-accessor
description: >
  Expert guide for accessing and querying data in the QAECY Cue knowledge graph.
  Use when a developer wants to understand a project's schema, write SPARQL queries,
  retrieve entities and documents, or extract insights from the dataset without
  building a full application. Covers service-account authentication, schema discovery,
  entity lookup, document retrieval, and natural-language search.
  Triggers: Cue query, Cue data, SPARQL, Cue dataset, knowledge graph query,
  entity lookup, Cue document retrieval, CUE_API_KEY.
argument-hint: '<description of the data you want to access or query>'
---

# Cue Data Accessor

Guides an agent to discover, query, and extract data from the QAECY Cue knowledge
graph using service accounts and the Cue SDK. Focuses on understanding the dataset
and building queries rather than building applications.

---

## Reference files

Load these files with `read_file` when you need them:

| File | When to load |
|---|---|
| `../cue-app-builder/context/sdk-api.md` | Full SDK method reference |
| `../cue-app-builder/context/ontology.md` | QAECY ontology, prefixes, and entity/document structure |
| `../cue-app-builder/context/sparql-patterns.md` | Ready-to-use SPARQL query patterns |
| `../cue-app-builder/sub-skills/list-projects/SKILL.md` | Listing the user's projects and finding project IDs |
| `../cue-app-builder/sub-skills/project-summary/SKILL.md` | Fetching the entity schema for an unfamiliar project |
| `../cue-app-builder/sub-skills/query-tester/SKILL.md` | Testing a SPARQL query with `CUE_API_KEY` |
| `../cue-app-builder/sub-skills/document-metadata/SKILL.md` | Document model and metadata SPARQL patterns |

Always load `../cue-app-builder/context/sdk-api.md` before writing any SDK code.

**Recommended opening sequence:**
1. Load `../cue-app-builder/sub-skills/list-projects/SKILL.md` → run it to identify the target `projectId`.
2. Load `../cue-app-builder/sub-skills/project-summary/SKILL.md` → run it to understand entity types and relationships in the project.
3. Load `../cue-app-builder/context/sparql-patterns.md` → choose or adapt a query pattern.
4. Load `../cue-app-builder/sub-skills/query-tester/SKILL.md` → test your query against real data.

---

## Step 1 — Clarify the data access goal

Before asking the user anything, understand what they're trying to retrieve:

- **Exploration**: Understand what entity types and relationships exist in a project?
- **Entity lookup**: Find specific entities, their properties, or relationships?
- **Document retrieval**: Fetch document metadata or search document content?
- **Batch export**: Extract a large dataset for analysis or integration?
- **Search**: Find relevant information using natural language?
- **Validation**: Test a query or understand why results differ from expectations?

Then clarify:

- Which project? (If unknown, use `list-projects/SKILL.md` to discover)
- What entities or documents? (If exploratory, use `project-summary/SKILL.md`)
- What format? (JSON, markdown table, CSV, Python dict, other?)
- Any filters or constraints? (date range, category, relationship type, etc.)

---

## Step 2 — Authenticate with a service account

**Load `../cue-app-builder/context/sdk-api.md` now.**

Service-account authentication is the standard for data access workflows — no user
login popup, no browser required.

### Node.js (scripts, batch jobs, CI/CD)

```bash
npm install @qaecy/cue-sdk
```

```ts
import { CueNode } from '@qaecy/cue-sdk/node';

const apiKey = process.env.CUE_API_KEY;
const projectId = 'proj-abc';  // or leave undefined to access all projects

const cue = new CueNode();
await cue.auth.signInWithApiKey(apiKey, projectId);

// Now ready to query
const result = await cue.api.sparql(query, projectId);
```

### Python (via fetch or external API calls)

If integrating with Python data tools, use authenticated fetch to the Cue API:

```python
import os
import requests

api_key = os.getenv('CUE_API_KEY')
project_id = 'proj-abc'

# Exchange API key for a Firebase custom token (SDK does this internally)
# Then use the token in subsequent requests.
# For simplicity, call the Cue SDK from Node.js or use the REST endpoints directly.
```

### Browser / SPA (if needed)

Even in a SPA, you can use service-account auth if you route SPARQL queries through
a backend server:

```ts
// Frontend (never exposes API key)
const result = await fetch('/api/sparql', {
  method: 'POST',
  body: JSON.stringify({ query, projectId })
});
```

```ts
// Node.js backend
import { CueNode } from '@qaecy/cue-sdk/node';

const cue = new CueNode();
await cue.auth.signInWithApiKey(process.env.CUE_API_KEY);

app.post('/api/sparql', async (req, res) => {
  const { query, projectId } = req.body;
  const result = await cue.api.sparql(query, projectId);
  res.json(result);
});
```

---

## Step 3 — Discover projects and schemas

### List available projects

Invoke `../cue-app-builder/sub-skills/list-projects/SKILL.md` to:
- See all projects the API key has access to
- Identify the target `projectId`

### Understand the schema

Once you have a `projectId`, invoke
`../cue-app-builder/sub-skills/project-summary/SKILL.md` to:
- Fetch a markdown table of all entity categories and their relationships
- Identify which entity types are most common
- Understand the data structure before writing queries

Example output (entity-category graph):
```
| Entity Type | Relationship | Related Type |
|---|---|---|
| Building | contains | Room |
| Building | isOwnedBy | Organization |
| Room | hasEquipment | Equipment |
```

---

## Step 4 — Query the knowledge graph

Load `../cue-app-builder/context/sparql-patterns.md` for copy-paste-ready queries.

### SPARQL (recommended for structured data extraction)

```ts
const PREFIXES = `
PREFIX rdf:   <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs:  <http://www.w3.org/2000/01/rdf-schema#>
PREFIX qcy:   <https://dev.qaecy.com/ont#>
PREFIX qcy-e: <https://dev.qaecy.com/enum#>
`;

// Example: Find all organizations sorted by mention count
const query = PREFIXES + `
  SELECT ?entity ?name (COUNT(DISTINCT ?mention) AS ?mentionCount) WHERE {
    ?entity a qcy:CanonicalEntity ;
            qcy:hasEntityCategory qcy-e:Organization ;
            qcy:value ?name .
    OPTIONAL { ?mention qcy:resolvesTo ?entity }
  }
  GROUP BY ?entity ?name
  ORDER BY DESC(?mentionCount)
  LIMIT 20
`;

const result = await cue.api.sparql(query, projectId);

// Flatten the result
const rows = result.results.bindings.map(b =>
  Object.fromEntries(Object.entries(b).map(([k, v]) => [k, v.value]))
);

console.table(rows);
```

**Key SPARQL patterns:**
- **Top-N entities by mention count**: Count how many documents mention each entity.
- **Entities by category**: Filter to a specific entity type (Organization, Person, Location, etc.).
- **Entity relationships**: Find outgoing or incoming links from an entity.
- **Coordinates**: Extract latitude/longitude for spatial queries.
- **Document associations**: Find which documents mention a specific entity.

### Natural-language search

When precise SPARQL is overkill or schema is unfamiliar:

```ts
const response = await cue.api.search({
  term: 'fire safety requirements for commercial buildings',
  projectId,
  categories: ['Building'],   // optional
});

// response.response    — AI-generated answer
// response.sources     — document snippets with relevance scores
```

### Hybrid approach (validate with test queries)

Before running a large batch job, always test your query:

Invoke `../cue-app-builder/sub-skills/query-tester/SKILL.md` to:
- Run your SPARQL query against real data
- See sample results
- Validate your assumptions about entity categories and relationships

---

## Step 5 — Retrieve entities and entity data

### Batch-fetch entity information

```ts
const entities = cue.projects.entities(projectId);

// Request data for multiple entities at once
await entities.requestEntityData(
  ['uuid-1', 'uuid-2', 'uuid-3'],
  /* includeMentionCount */ true
);

// Access the cached data
const info = entities.entityInfoMap.get();
// {
//   'uuid-1': { value: 'Acme Corp', categories: ['Organization'], mentionCount: 42 },
//   'uuid-2': { value: 'John Smith', categories: ['Person'], mentionCount: 15 },
//   …
// }
```

### Get entity relationships

```ts
// Given an entity IRI from your SPARQL result
const iri = 'https://cue.qaecy.com/r/proj-abc/uuid-1';

const rels = await entities.fetchEntityRelationships(iri);
// {
//   incoming: [
//     { from: '…', predicate: 'qcy:employedBy', label: 'Employed by' }
//   ],
//   outgoing: [
//     { to: '…', predicate: 'qcy:employs', label: 'Employs' }
//   ]
// }
```

### Find which documents mention an entity

```ts
const docIris = await entities.fetchEntityDocuments(iri);
// ['https://…/doc-uuid-1', 'https://…/doc-uuid-2', …]
```

### Construct an entity IRI from a UUID

```ts
const iri = entities.entityIri('my-uuid');
// 'https://cue.qaecy.com/r/proj-abc/my-uuid'
```

---

## Step 6 — Retrieve documents

Load `../cue-app-builder/sub-skills/document-metadata/SKILL.md` for detailed
document structure, file-location vs. file-content, and format conversions.

### Fetch document metadata

```ts
const docs = cue.projects.documents(projectId);

// Request metadata for specific documents
await docs.requestDocumentData(['doc-uuid-1', 'doc-uuid-2']);

const info = docs.documentInfoMap.get();
// {
//   'doc-uuid-1': {
//     id: 'doc-uuid-1',
//     path: 'documents/report.pdf',
//     suffix: 'pdf',
//     size: 2048576,
//     tags: ['structural', 'inspection'],
//     categories: ['BuildingDocumentation'],
//     subject: 'Annual Building Inspection 2023',
//     summary: 'Comprehensive structural assessment…',
//     providerId: 'my-provider'
//   },
//   …
// }
```

### Project-level document overview

```ts
await docs.fetchOverview();
const overview = docs.projectDocumentsData.get();
// {
//   fileTypeCounts: { pdf: 150, docx: 45, xlsx: 12 },
//   categoryBreakdown: { BuildingDocumentation: 120, … },
//   duplicates: { 'doc-uuid-1': 'doc-uuid-2', … }
// }
```

### Search document content

Use natural-language search (Step 4) to find relevant documents:

```ts
const response = await cue.api.search({
  term: 'structural defects',
  projectId,
});
// response.sources includes document snippets and file paths
```

---

## Step 7 — Export and transform data

### Export as JSON

```ts
const rows = result.results.bindings.map(b =>
  Object.fromEntries(Object.entries(b).map(([k, v]) => [k, v.value]))
);
console.log(JSON.stringify(rows, null, 2));
```

### Export as CSV (Node.js)

```ts
import { stringify } from 'csv-stringify/sync';

const csv = stringify(rows, { header: true });
fs.writeFileSync('export.csv', csv);
```

### Export as markdown table

```ts
const md = rows
  .slice(0, 20)
  .reduce((acc, row) => {
    const cells = Object.values(row).map(v => String(v).slice(0, 30));
    return acc + '| ' + cells.join(' | ') + ' |\n';
  }, '| ' + Object.keys(rows[0]).join(' | ') + ' |\n|---|---|\n');

console.log(md);
```

### Transform for Python / pandas

```ts
// From Node.js → Python:
// 1. Export as JSON or CSV
// 2. Load into pandas:
//    pd.read_json('entities.json')
//    pd.read_csv('entities.csv')
```

---

## Step 8 — Validate and debug

### Test a query before running at scale

Always invoke `../cue-app-builder/sub-skills/query-tester/SKILL.md` when:
- Writing a new SPARQL query
- Unsure if an entity category or property name is correct
- Results seem unexpected or incomplete
- Troubleshooting performance (empty results, timeout, etc.)

### Common issues and solutions

| Issue | Diagnosis | Solution |
|---|---|---|
| Empty results | Entity category name wrong? | Run query-tester with a broader query to discover available categories |
| Unexpected entity count | Missing OPTIONAL clauses? | Check if mentions are required or optional in the schema |
| Slow query | Too many joins or large result set? | Add LIMIT, use FILTER, or narrow the entity category |
| URIs don't resolve | Stale or malformed IRIs? | Verify IRI format against `entityIri()` output |

### Inspect the ontology

When results are unexpected, load `../cue-app-builder/context/ontology.md` to:
- Understand the full namespace (prefixes, enum values)
- See which properties are defined for each entity type
- Learn how documents and entities are connected

---

## Workflow summary

1. **Authenticate** with `signInWithApiKey(apiKey, projectId)` (Step 2)
2. **Discover** available projects (Step 3)
3. **Understand** the project schema with project-summary (Step 3)
4. **Choose** an access pattern: SPARQL (exact), natural-language search (flexible), or entity lookup (by UUID) (Step 4–5)
5. **Test** your query with query-tester before running at scale (Step 8)
6. **Execute** the query and iterate
7. **Export** results in your target format (Step 7)
8. **Validate** results against expectations (Step 8)

---

## Examples

### Example 1: Find all buildings and their locations

```ts
const query = PREFIXES + `
  SELECT ?building ?name ?lat ?lng WHERE {
    ?building a qcy:CanonicalEntity ;
              qcy:hasEntityCategory qcy-e:Building ;
              qcy:value ?name .
    OPTIONAL {
      ?building qcy:latitude  ?lat ;
                qcy:longitude ?lng .
    }
  }
  LIMIT 100
`;

const result = await cue.api.sparql(query, projectId);
const buildings = result.results.bindings.map(b =>
  Object.fromEntries(Object.entries(b).map(([k, v]) => [k, v.value]))
);

console.table(buildings);
```

### Example 2: Find the top 10 organizations by mention frequency

```ts
const query = PREFIXES + `
  SELECT ?org ?name (COUNT(DISTINCT ?mention) AS ?mentions) WHERE {
    ?org a qcy:CanonicalEntity ;
         qcy:hasEntityCategory qcy-e:Organization ;
         qcy:value ?name .
    OPTIONAL { ?mention qcy:resolvesTo ?org }
  }
  GROUP BY ?org ?name
  ORDER BY DESC(?mentions)
  LIMIT 10
`;

const result = await cue.api.sparql(query, projectId);
// … process and export
```

### Example 3: Natural-language search for specific information

```ts
const response = await cue.api.search({
  term: 'electrical system failures reported in 2023',
  projectId,
});

console.log('Answer:', response.response);
console.log('Sources:');
response.sources.forEach(src => {
  console.log(`  - ${src.document} (${src.relevance.toFixed(2)})`);
});
```

---

## When to use Cue Data Accessor vs. Cue App Builder

| Goal | Use Data Accessor | Use App Builder |
|---|---|---|
| Understand a project's schema | ✓ | |
| Write and test SPARQL queries | ✓ | |
| Extract entities for analysis | ✓ | |
| Build a search UI | | ✓ |
| Build a dashboard or map view | | ✓ |
| Batch export data | ✓ | |
| Create an interactive web app | | ✓ |
| Validate assumptions before app-building | ✓ | |

---

## Resources

- **SDK**: `@qaecy/cue-sdk` on npm
- **API Key**: Request via QAECY platform
- **Questions?** Reference the app-builder's context files and sub-skills for deeper dives into ontology, SDK methods, and query patterns.
