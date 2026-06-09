---
name: cue-query-tester
description: >
  Test a SPARQL query against a live Cue project graph using a CUE_API_KEY
  environment variable. Use when iterating on SPARQL queries during Cue app
  development. Triggers: test SPARQL, run query against Cue, check query
  results, CUE_API_KEY query, validate SPARQL.
argument-hint: '-s <projectId> -q "<sparql-query>"'

---

# Cue Query Tester

Runs a SPARQL query against a live Cue project using the developer's
`CUE_API_KEY` and prints the raw SPARQL JSON response.

---

## Prerequisites

- Node.js 18+ with ESM support
- `CUE_API_KEY` environment variable set (obtain from the QAECY portal)
- No local install needed — npx handles the CLI automatically

---

## Procedure

### Step 1 — Resolve the project ID

If the user has not provided a project ID, load `context/sdk-api.md` and use
`cue.projects.listProjects()` — or ask the user to check their QAECY portal.

### Step 2 — Prepare the SPARQL query

If the query is more than a few lines, write it to a temporary `.sparql` file
so it can be passed as a file path argument. Include standard prefixes:

```sparql
PREFIX rdf:   <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs:  <http://www.w3.org/2000/01/rdf-schema#>
PREFIX qcy:   <https://dev.qaecy.com/ont#>
PREFIX qcy-e: <https://dev.qaecy.com/enum#>

SELECT ?entity ?name WHERE {
  ?entity a qcy:CanonicalEntity ;
          qcy:value ?name .
}
LIMIT 10
```

### Step 3 — Run the test command

Run with `run_in_terminal`:

```bash
CUE_API_KEY=<key> npx @qaecy/cue-cli app-builder-tools sparql -s <projectId> -q "SELECT ?s ?p ?o WHERE { ?s ?p ?o } LIMIT 5"
```

The `CUE_API_KEY` must be set as an environment variable — never pass it as a
positional argument or write it to a file.

### Step 4 — Interpret results

The CLI prints a diagnostic line (`Executing SPARQL query…`) followed by the
raw SPARQL JSON response. Bindings are in `results.bindings`; each value has
a `type` (`uri` or `literal`) and a `value`. Check `meta.result-size-total`
for the total match count (may exceed the `LIMIT`).

Common issues:

| Symptom | Likely cause |
|---|---|
| `0 results` | Wrong category IRI — use `qcy-e:Organization` not `"Organization"` |
| `Error: permission denied` | API key does not have access to this project |
| `Error: invalid query` | Syntax error — check prefixes and variable binding |
| Large result set | Add `LIMIT` and `OFFSET` to paginate |

### Step 5 — Iterate

Refine the query based on results and re-run. Once the query returns the
expected data, paste it back into the app code.

---

## Security notes

- `CUE_API_KEY` is a bearer credential. Never commit it to source control.
- Use `.env` files (gitignored) or shell environment variables.
- The script does **not** log or persist the key.
