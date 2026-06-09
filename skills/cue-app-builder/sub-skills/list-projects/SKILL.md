---
name: cue-list-projects
description: >
  List all Cue projects the current API key has access to. Use at the start of
  any app-building session to discover available project IDs and names.
  Triggers: list projects, which projects, project ID, what projects do I have.
argument-hint: '(no arguments — uses CUE_API_KEY)'
---

# Cue List Projects

Authenticates with `CUE_API_KEY` and prints all projects the user is a member
of, including their IDs. Run this first to find the `projectId` needed by all
other sub-skills and app code.

---

## Prerequisites

- `CUE_API_KEY` environment variable set
- Node.js 18+ (npx handles the CLI automatically)

## Usage

```bash
CUE_API_KEY=<key> npx @qaecy/cue-cli app-builder-tools list-projects
```

## Output

A JSON array of project objects, e.g.:

```json
[
  {
    "id": "proj-osl-3f2a1b2c-9d8e-4f7a-b6c5-1a2b3c4d5e6f",
    "name": "Oslo Airport Extension",
    "organizationID": "org-7d09-f32a-43cb-9879-777e7a4d652b",
    "isPublic": false,
    "graphType": "qlever",
    "lastSync": "2025-10-01T17:50:14.387Z"
  }
]
```

Copy the `id` value and use it as `projectId` in app code or when invoking
the `project-summary` or `query-tester` sub-skills.
