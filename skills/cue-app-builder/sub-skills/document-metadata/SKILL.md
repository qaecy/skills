---
name: cue-document-metadata
description: >
  Explains the Cue document model: the FileLocation / FileContent split,
  what metadata fields exist, and how Cue generates alternative
  representations for formats that cannot be viewed natively in a browser
  (e.g. DWG → DXF, IFC → ThatOpen fragments). Use when building document
  browsers, file viewers, or any feature that surfaces document metadata.
  Triggers: document metadata, FileContent, FileLocation, alternative
  representation, DWG, IFC, ThatOpen, document schema, file fields.
argument-hint: '(no arguments — conceptual reference + SPARQL patterns)'
---

# Cue Document Metadata

---

## FileLocation vs FileContent

These two classes model a document's **identity** (where it lives) separately
from its **content** (what it contains). The distinction matters for
deduplication and for supporting multiple renderings of the same source file.

| Class | Purpose | Key fields |
|---|---|---|
| `qcy:FileLocation` | Physical file reference — original path on disk or cloud storage | `qcy:value` (filename), `qcy:suffix` (**extension including leading dot**, e.g. `".pdf"`), `qcy:filePath`, `qcy:remoteRelativePath` |
| `qcy:FileContent` | A specific version/encoding of the file content | `qcy:mime`, `qcy:sizeBytes`, `qcy:md5Hash`, `qcy:hasContentCategory`, `qcy:hasMimeCategory`, `qcy:about` (primary entity), `qcy:textRaw` (full extracted text), `qcy:textSummary` (AI-generated summary) |

One `FileLocation` always has exactly one `FileContent` for the **original**
file. When Cue generates alternative representations (see below), those are
additional `FileContent` nodes — they do **not** get their own `FileLocation`.

The relationship from a document's perspective:

```
FileContent  ──qcy:hasFileLocation──→  FileLocation
     │
     └── qcy:containsFragment ──→  DocumentPageFragment
                                        └── qcy:alternativeRepresentation ──→  FileContent (alt)
```

---

## Alternative representations

Cue automatically converts documents that cannot be viewed natively in a
browser into a web-friendly format. The converted file is stored as a new
`FileContent` node hanging off the original document's fragment via
`qcy:alternativeRepresentation`.

| Source format | Alternative representation | Purpose |
|---|---|---|
| `.dwg` (AutoCAD) | `.dxf` | Web-renderable CAD exchange format for browser viewers |
| `.ifc` (BIM) | ThatOpen fragments (`.frag` / `.json`) | Binary format consumed by the ThatOpen 3D viewer |
| `.dwg` (AutoCAD) | `.md` | Text description of model space and paper spaces |
| `.ifc` (BIM) | `.md` | Text description of BIM elements and spatial decomposition |

The alternative `FileContent` has its own `qcy:mime`, `qcy:sizeBytes`, and
`qcy:remoteRelativePath` (via its linked `FileLocation`), but the same
`qcy:about` entity as the source document.

The `.md` alternative representations are particularly useful for AI/LLM
pipelines — they expose the structural content of otherwise opaque binary
formats as plain text that can be embedded or searched.

---

## SPARQL patterns

Use `npx @qaecy/cue-cli app-builder-tools sparql` to test these queries
(see the `query-tester` sub-skill).

### All metadata for every document in a project

```sparql
SELECT ?content ?name ?suffix ?mime ?size ?hash ?contentCategory WHERE {
  ?content a qcy:FileContent ;
           qcy:hasFileLocation ?loc .
  ?loc qcy:value ?name ;
       qcy:suffix ?suffix .
  OPTIONAL { ?content qcy:mime ?mime }
  OPTIONAL { ?content qcy:sizeBytes ?size }
  OPTIONAL { ?content qcy:md5Hash ?hash }
  OPTIONAL { ?content qcy:hasContentCategory ?contentCategory }
}
ORDER BY ?name
```

### Find all documents that have an alternative representation

```sparql
SELECT ?srcName ?srcSuffix ?altMime ?altSize WHERE {
  ?content a qcy:FileContent ;
           qcy:hasFileLocation ?loc ;
           qcy:containsFragment ?frag .
  ?loc qcy:value ?srcName ;
       qcy:suffix ?srcSuffix .
  ?frag qcy:alternativeRepresentation ?alt .
  OPTIONAL { ?alt qcy:mime ?altMime }
  OPTIONAL { ?alt qcy:sizeBytes ?altSize }
}
ORDER BY ?srcName
```

### Fetch the alternative representation for a specific document

```sparql
SELECT ?altContent ?altMime ?altPath WHERE {
  <CONTENT_IRI> qcy:containsFragment ?frag .
  ?frag qcy:alternativeRepresentation ?altContent .
  OPTIONAL { ?altContent qcy:mime ?altMime }
  OPTIONAL {
    ?altContent qcy:hasFileLocation ?altLoc .
    ?altLoc qcy:remoteRelativePath ?altPath .
  }
}
```

### Text content and summaries for documents

```sparql
SELECT ?name ?textRaw ?textSummary WHERE {
  ?content a qcy:FileContent ;
           qcy:hasFileLocation ?loc .
  ?loc qcy:value ?name .
  OPTIONAL { ?content qcy:textRaw ?textRaw }
  OPTIONAL { ?content qcy:textSummary ?textSummary }
  FILTER(BOUND(?textRaw) || BOUND(?textSummary))
}
ORDER BY ?name
```

### Storage paths for all original files (e.g. to build download URLs)

```sparql
SELECT ?name ?suffix ?remotePath WHERE {
  ?content a qcy:FileContent ;
           qcy:hasFileLocation ?loc .
  ?loc qcy:value ?name ;
       qcy:suffix ?suffix .
  OPTIONAL { ?loc qcy:remoteRelativePath ?remotePath }
}
ORDER BY ?name
```

---

## Notes

- `qcy:suffix` **always includes the leading dot** (e.g. `".pdf"`, `".ifc"`). Never filter with a bare extension like `"pdf"` — use `FILTER(LCASE(?suffix) = ".pdf")` to handle mixed case.
- `qcy:hasContentCategory` is a semantic label assigned by Cue's classifier
  (e.g. `qcy-e:ArchitecturalDesignDocument`). It is separate from the MIME
  type and may not always be present.
- `qcy:md5Hash` is used by the sync pipeline for deduplication — two uploads
  of the same file will share a `FileContent` node.
- `qcy:remoteRelativePath` is a GCS-relative path. Combine it with the
  project's storage bucket base URL to build a download link.
