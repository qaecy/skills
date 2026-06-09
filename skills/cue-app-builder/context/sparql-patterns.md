# Cue — SPARQL Patterns

All patterns use these shared prefixes (include at the top of every query):

```sparql
PREFIX rdf:   <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs:  <http://www.w3.org/2000/01/rdf-schema#>
PREFIX qcy:   <https://dev.qaecy.com/ont#>
PREFIX qcy-e: <https://dev.qaecy.com/enum#>
PREFIX geo:   <http://www.w3.org/2003/01/geo/wgs84_pos#>
```

---

## Entity queries

### Top-N entities by mention count (any category)

```sparql
SELECT ?entity ?name (COUNT(DISTINCT ?mention) AS ?count) WHERE {
  ?entity a qcy:CanonicalEntity ;
          qcy:value ?name .
  OPTIONAL { ?mention qcy:resolvesTo ?entity }
}
GROUP BY ?entity ?name
ORDER BY DESC(?count)
LIMIT 10
```

### Top-N entities filtered by category

Replace `qcy-e:Organization` with any `qcy-e:` category.

```sparql
SELECT ?entity ?name (COUNT(DISTINCT ?mention) AS ?count) WHERE {
  ?entity a qcy:CanonicalEntity ;
          qcy:hasEntityCategory qcy-e:Organization ;
          qcy:value ?name .
  OPTIONAL { ?mention qcy:resolvesTo ?entity }
}
GROUP BY ?entity ?name
ORDER BY DESC(?count)
LIMIT 5
```

### All properties of a specific entity

```sparql
SELECT ?p ?o WHERE {
  <ENTITY_IRI> ?p ?o .
  FILTER(?p != rdf:type)
}
```

### Outgoing relationships to other canonical entities

```sparql
SELECT ?predicate ?relEntity ?relName WHERE {
  <ENTITY_IRI> ?predicate ?relEntity .
  ?relEntity a qcy:CanonicalEntity ;
             qcy:value ?relName .
}
LIMIT 20
```

### Coordinates stored on an entity

```sparql
SELECT ?lat ?lng WHERE {
  {
    <ENTITY_IRI> qcy:latitude  ?lat ;
                 qcy:longitude ?lng .
  } UNION {
    <ENTITY_IRI> geo:lat  ?lat ;
                 geo:long ?lng .
  }
}
LIMIT 1
```

---

## Document / mention queries

### Documents and pages where an entity is mentioned

```sparql
SELECT DISTINCT ?docName ?page ?summary WHERE {
  ?mention qcy:resolvesTo <ENTITY_IRI> .
  ?content qcy:mentions ?mention ;
           qcy:hasFileLocation ?loc .
  ?loc qcy:value ?docName .
  OPTIONAL {
    ?frag qcy:mentions ?mention .
    OPTIONAL { ?frag qcy:textSummary ?summary }
    OPTIONAL {
      ?sel qcy:selectorObject ?frag ;
           qcy:value ?page .
    }
  }
}
ORDER BY ?docName ?page
LIMIT 30
```

### All documents in a project (name, size, type)

```sparql
SELECT ?doc ?name ?suffix ?size ?category WHERE {
  ?doc a qcy:FileContent ;
       qcy:hasFileLocation ?loc .
  ?loc qcy:value ?name ;
       qcy:suffix ?suffix .
  OPTIONAL { ?doc qcy:sizeBytes ?size }
  OPTIONAL { ?doc qcy:hasContentCategory ?category }
}
ORDER BY ?name
```

### Documents containing a keyword in text

```sparql
SELECT DISTINCT ?docName ?page ?text WHERE {
  ?frag a qcy:DocumentPageFragment ;
        qcy:textRaw ?text .
  FILTER(CONTAINS(LCASE(?text), "keyword"))
  OPTIONAL {
    ?sel qcy:selectorObject ?frag ;
         qcy:selectorSubject ?content ;
         qcy:value ?page .
    ?content qcy:hasFileLocation ?loc .
    ?loc qcy:value ?docName .
  }
}
LIMIT 20
```

### Page summaries for a specific document

```sparql
SELECT ?page ?summary WHERE {
  ?sel qcy:selectorSubject <CONTENT_IRI> ;
       qcy:selectorObject  ?frag ;
       qcy:value ?page .
  ?frag qcy:textSummary ?summary .
}
ORDER BY xsd:integer(?page)
```

---

## Project-level overview queries

### Count entities by category

```sparql
SELECT ?category (COUNT(DISTINCT ?entity) AS ?count) WHERE {
  ?entity a qcy:CanonicalEntity ;
          qcy:hasEntityCategory ?category .
}
GROUP BY ?category
ORDER BY DESC(?count)
```

### Count documents by file suffix

```sparql
SELECT ?suffix (COUNT(DISTINCT ?doc) AS ?count) WHERE {
  ?doc a qcy:FileContent ;
       qcy:hasFileLocation ?loc .
  ?loc qcy:suffix ?suffix .
}
GROUP BY ?suffix
ORDER BY DESC(?count)
```

### Count documents by content category

```sparql
SELECT ?category (COUNT(DISTINCT ?doc) AS ?count) WHERE {
  ?doc a qcy:FileContent ;
       qcy:hasContentCategory ?category .
}
GROUP BY ?category
ORDER BY DESC(?count)
```

### Entities that co-occur (mentioned in the same document)

```sparql
SELECT ?nameA ?nameB (COUNT(DISTINCT ?content) AS ?coOccurrences) WHERE {
  ?content qcy:mentions ?mentionA ;
           qcy:mentions ?mentionB .
  ?mentionA qcy:resolvesTo ?entityA .
  ?mentionB qcy:resolvesTo ?entityB .
  ?entityA qcy:value ?nameA .
  ?entityB qcy:value ?nameB .
  FILTER(?entityA != ?entityB && STR(?entityA) < STR(?entityB))
}
GROUP BY ?nameA ?nameB
ORDER BY DESC(?coOccurrences)
LIMIT 20
```

---

## Tips

- Always `GROUP BY` on all non-aggregated variables in `SELECT`.
- Use `COUNT(DISTINCT …)` to avoid inflated counts from multi-valued paths.
- IRIs must be wrapped in `< >`. Use `STR(?var)` for string operations on IRI-valued variables.
- `qcy-e:` values are enum IRIs — do not quote them as strings.
- For large result sets, add `LIMIT` and `OFFSET` for pagination.
- To explore an unknown graph, start with `SELECT DISTINCT ?type WHERE { ?s a ?type } LIMIT 50` to see what classes exist.

### QLever: aggregation over IRIs requires STR()

QLever does **not** allow aggregate functions (`SAMPLE`, `GROUP_CONCAT`, `MIN`, `MAX`) to be applied directly to IRI-valued variables. Always wrap the variable in `STR()` first:

```sparql
# WRONG — QLever will error
SELECT (SAMPLE(?entity) AS ?sampleEntity) WHERE { ... }

# CORRECT
SELECT (SAMPLE(STR(?entity)) AS ?sampleEntity) WHERE { ... }
SELECT (GROUP_CONCAT(STR(?entity) ; separator=",") AS ?entities) WHERE { ... }
```

This applies to any variable whose values are IRIs (e.g. `?entity`, `?category`, `?doc`). Literal-valued variables (strings, numbers) do not need `STR()`.
