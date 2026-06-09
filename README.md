# QAECY Agent Skills

A growing collection of reusable agent skills for the [QAECY](https://qaecy.com) ecosystem. Skills are instruction sets that extend your coding agent's capabilities — install them once and your agent gains domain-specific knowledge it can apply on demand.

This repo currently focuses on the **Cue platform**, but more skills covering other QAECY tools and workflows will be added over time.

## Available Skills

| Skill | Description |
|---|---|
| [cue-app-builder](skills/cue-app-builder/SKILL.md) | Expert guide for building apps on the QAECY Cue platform using the Cue SDK |

## Installation

Install all skills using [`npx skills`](https://www.npmjs.com/package/skills):

```bash
npx skills add qaecy/skills
```

Install a specific skill:

```bash
npx skills add qaecy/skills --skill cue-app-builder
```

Install globally (available across all projects):

```bash
npx skills add qaecy/skills -g
```

### Supported Agents

Skills are compatible with all major coding agents including GitHub Copilot, Claude Code, Cursor, Codex, OpenCode, and [70+ more](https://www.npmjs.com/package/skills#supported-agents).

## Usage

Once installed, mention the skill topic in your coding agent to activate it. For example:

- *"Build a Cue app that lists my projects"*
- *"Add Cue authentication to my React app"*
- *"Write a SPARQL query against the Cue knowledge graph"*

## Web Components

The `cue-app-builder` skill includes guidance for the `@qaecy/cue-ui` web component library (`<cue-by-cue-logo>`, `<cue-entity-list>`, `<cue-entity-viewer>`). Full interactive component documentation is available at:

**https://slides.qaecy.com/components**

## Creating Your Own Skills

See the [Agent Skills specification](https://agentskills.io/) and the [`npx skills` CLI](https://www.npmjs.com/package/skills) for guidance on creating and publishing skills.
