# QAECY Ontology Reference

---

## Standard prefixes

```sparql
PREFIX rdf:   <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs:  <http://www.w3.org/2000/01/rdf-schema#>
PREFIX xsd:   <http://www.w3.org/2001/XMLSchema#>
PREFIX qcy:   <https://dev.qaecy.com/ont#>
PREFIX qcy-e: <https://dev.qaecy.com/enum#>
PREFIX skos:  <http://www.w3.org/2004/02/skos/core#>
PREFIX geo:   <http://www.w3.org/2003/01/geo/wgs84_pos#>
```

---

## Core node types

| Class | IRI | Description |
|---|---|---|
| `qcy:CanonicalEntity` | `qcy:CanonicalEntity` | Deduplicated, project-scoped entity (the primary entry point) |
| `qcy:FileLocation` | `qcy:FileLocation` | A file path / storage reference |
| `qcy:FileContent` | `qcy:FileContent` | A specific version / content of a file, linked to one `FileLocation` |
| `qcy:DocumentPageFragment` | `qcy:DocumentPageFragment` | A page or section of a document |
| `qcy:Mention` | `qcy:Mention` | An occurrence of an entity within a document |
| `qcy:PageSelector` | `qcy:PageSelector` | Maps a fragment to a page number within a content node |

---

## Key predicates

### On `qcy:CanonicalEntity`

| Predicate | Range | Description |
|---|---|---|
| `qcy:hasEntityCategory` | `qcy-e:*` | Entity category (e.g. `qcy-e:Building`, `qcy-e:Organization`) |
| `qcy:value` | `xsd:string` | Human-readable label / name |
| `qcy:latitude` / `qcy:longitude` | `xsd:decimal` | Direct WGS-84 coordinates (if available) |
| `geo:lat` / `geo:long` | `xsd:decimal` | Alternative WGS-84 predicates |

### On `qcy:FileContent`

| Predicate | Range | Description |
|---|---|---|
| `qcy:about` | `qcy:CanonicalEntity` | Primary entity this document is about |
| `qcy:hasContentCategory` | `qcy-e:*` | Document type (e.g. `qcy-e:ArchitecturalDesignDocument`) |
| `qcy:hasMimeCategory` | `qcy-e:*` | Mime category (e.g. `qcy-e:TextFileContent`) |
| `qcy:hasFileLocation` | `qcy:FileLocation` | Associated file location |
| `qcy:containsFragment` | `qcy:DocumentPageFragment` | Contained fragments |
| `qcy:mentions` | `qcy:Mention` | Mentions occurring in this content |
| `qcy:mime` | `xsd:string` | MIME type string |
| `qcy:sizeBytes` | `xsd:integer` | File size |
| `qcy:md5Hash` | `xsd:string` | MD5 checksum for deduplication |

### On `qcy:FileLocation`

| Predicate | Range | Description |
|---|---|---|
| `qcy:filePath` | `xsd:string` | Local/original file path |
| `qcy:remoteRelativePath` | `xsd:string` | Path on cloud storage (GCS) |
| `qcy:value` | `xsd:string` | Filename |
| `qcy:suffix` | `xsd:string` | File extension (e.g. `.pdf`) |

### On `qcy:DocumentPageFragment`

| Predicate | Range | Description |
|---|---|---|
| `qcy:about` | `qcy:CanonicalEntity` | Entity this fragment is about |
| `qcy:mentions` | `qcy:Mention` | Mentions in this fragment |
| `qcy:alternativeRepresentation` | `qcy:FileContent` | E.g. a rendered image of the page |
| `qcy:tag` | `xsd:string` | Free-text tags (multi-valued) |
| `qcy:textRaw` | `xsd:string` | Raw extracted text |
| `qcy:textSummary` | `xsd:string` | AI-generated summary |

### On `qcy:Mention`

| Predicate | Range | Description |
|---|---|---|
| `qcy:hasEntityCategory` | `qcy-e:*` | Category of the mentioned entity |
| `qcy:value` | `xsd:string` | Surface form as it appears in the document |
| `qcy:resolvesTo` | `qcy:CanonicalEntity` | The canonical entity this mention refers to |

### On `qcy:PageSelector`

| Predicate | Range | Description |
|---|---|---|
| `qcy:selectorSubject` | `qcy:FileContent` | The containing document |
| `qcy:selectorObject` | `qcy:DocumentPageFragment` | The fragment being selected |
| `qcy:value` | `xsd:string` | Page number or selector value |

---

## Common `qcy-e:` entity categories

These are the most frequently used values for `qcy:hasEntityCategory`:

| Enum IRI | Typical meaning |
|---|---|
| `qcy-e:Organization` | Company, contractor, public body |
| `qcy-e:Building` | A structure / building |
| `qcy-e:Person` | Individual |
| `qcy-e:Location` | Geographic place |
| `qcy-e:Product` | A manufactured item |
| `qcy-e:Document` | A referenced document |
| `qcy-e:Date` | A date or time period |
| `qcy-e:Project` | A project reference |

> Client datasets may define additional custom enum values under their own namespace.

---

## Common `qcy-e:` content categories (document types)

| Enum IRI | Description |
|---|---|
| `qcy-e:ArchitecturalDesignDocument` | Architectural drawings, plans |
| `qcy-e:StructuralDesignDocument` | Structural engineering documents |
| `qcy-e:TechnicalSpecification` | Technical specs and datasheets |
| `qcy-e:Contract` | Legal / contractual documents |
| `qcy-e:Report` | Reports and assessments |

---

## Traversal pattern (entity → evidence)

```
CanonicalEntity
  └── ← qcy:about ── FileContent
            ├── qcy:hasFileLocation ──→ FileLocation  (filename, path)
            ├── qcy:containsFragment ──→ DocumentPageFragment
            │         ├── qcy:textSummary             (AI summary)
            │         ├── qcy:alternativeRepresentation ──→ FileContent (page image)
            │         └── ← qcy:selectorObject ── PageSelector
            │                     └── qcy:value       (page number)
            └── qcy:mentions ──→ Mention
                      └── qcy:resolvesTo ──→ CanonicalEntity
```

Prefer `qcy:CanonicalEntity` as the entry point. Follow `qcy:about` backwards
(use `?content qcy:about ?entity`) to reach documents and fragments.

---

## Turtle modeling example

```turtle
@prefix qcy:   <https://dev.qaecy.com/ont#> .
@prefix qcy-e: <https://dev.qaecy.com/enum#> .

ex:canon1 a qcy:CanonicalEntity ;
  qcy:hasEntityCategory qcy-e:Building ;
  qcy:value "Building A" .

ex:doc1 a qcy:FileLocation ;
  qcy:filePath "original/path/xx.pdf" ;
  qcy:remoteRelativePath "Path/on/gcp.pdf" ;
  qcy:value "xx.pdf" ;
  qcy:suffix ".pdf" .

ex:content1 a qcy:FileContent ;
  qcy:about ex:canon1 ;
  qcy:hasContentCategory qcy-e:ArchitecturalDesignDocument ;
  qcy:hasMimeCategory qcy-e:TextFileContent ;
  qcy:hasFileLocation ex:doc1 ;
  qcy:md5Hash "e92ea99cc88c78084eb81499a79b115e" ;
  qcy:containsFragment ex:frag1 ;
  qcy:mime "application/pdf" ;
  qcy:sizeBytes 382872 ;
  qcy:mentions ex:mention1 .

ex:frag1 a qcy:DocumentPageFragment ;
  qcy:about ex:canon1 ;
  qcy:mentions ex:mention1 ;
  qcy:alternativeRepresentation ex:content2 ;
  qcy:tag "3D Rendering" , "ANLIKER AG" ;
  qcy:textRaw "All page text" ;
  qcy:textSummary "A summary of the page text" .

ex:mention1 a qcy:Mention ;
  qcy:hasEntityCategory qcy-e:Building ;
  qcy:value "Building A" ;
  qcy:resolvesTo ex:canon1 .

ex:sel a qcy:PageSelector ;
  qcy:selectorSubject ex:content1 ;
  qcy:selectorObject ex:frag1 ;
  qcy:value "1" .
```
