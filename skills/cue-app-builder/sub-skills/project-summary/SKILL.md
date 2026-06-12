---
name: cue-project-summary
description: >
  Fetch the entity summary graph for a Cue project in markdown format. Shows
  all entity categories and their relationships with occurrence weights — the
  essential first step when building an app on an unfamiliar project, since
  schemas vary between clients and industries. Triggers: project schema,
  entity categories, summary graph, what entities exist, project structure.
argument-hint: '<projectId>'
---

# Cue Project Summary

Fetches the entity category relationship graph for a project and prints it as
a markdown table. Use this before writing SPARQL queries or designing a UI —
it tells you which entity types exist in the project and how they relate to
each other, so you know what's actually queryable.

---

## Why this matters

Every Cue project has a custom schema shaped by the documents it contains and
any tuner configuration applied by the project owner. A construction project
might have `Building`, `Organization`, `Contract`, and `FloorPlan` entities.
An infrastructure project might have `Asset`, `Workorder`, and `Supplier`.

Running this sub-skill first avoids writing queries against categories that
don't exist in the target project.

---

## Prerequisites

- `CUE_API_KEY` environment variable set
- Node.js 18+ (npx handles the CLI automatically)

## Usage

```bash
CUE_API_KEY=<key> npx @qaecy/cue-cli app-builder-tools entity-summary-graph -s <projectId>
```

### Focusing on a specific entity category

Pass the `--entity` flag to restrict the graph to relationships involving a
single entity category. The value can be either a full IRI or a prefixed one:

```bash
# Prefixed form
CUE_API_KEY=<key> npx @qaecy/cue-cli app-builder-tools entity-summary-graph -s <projectId> --entity qcy-e:Contract

# Full IRI form
CUE_API_KEY=<key> npx @qaecy/cue-cli app-builder-tools entity-summary-graph -s <projectId> --entity https://dev.qaecy.com/enum#Contract
```

Use this to drill into one category once the full summary has shown you which
categories exist.

## Output

A markdown-formatted table of entity category relationships and their
occurrence weights, e.g.:

```
qcy-e:DrawingSheet  -> qcy:hasAssociatedDiscipline  -> qcy-e:Discipline        (574)
qcy-e:DrawingSheet  -> qcy:hasDesignMetadata         -> qcy-e:DesignRevisionId  (359)
qcy-e:DrawingSheet  -> qcy:hasAssociatedDiscipline   -> qcy-e:ElectricalSystem  (286)
```

The two `Executing SPARQL query…` lines printed before the results are
diagnostic output from the CLI and can be ignored.

Use the category IRIs from this output (e.g. `qcy-e:DrawingSheet`) directly
in `qcy:hasEntityCategory` filters in your SPARQL queries.

---

## Interpreting the output

- The **left column** is the source entity category.
- The **middle column** is the predicate (relationship type).
- The **right column** is the target entity category.
- The **number in parentheses** is the occurrence count — higher means more
  prominent in this specific project's data.

Focus your app on the relationships with the highest counts; they represent the
richest, most reliable data in the graph.
