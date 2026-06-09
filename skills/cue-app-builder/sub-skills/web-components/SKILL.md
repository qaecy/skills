---
name: cue-web-components
description: >
  Reference for the @qaecy/cue-ui web component library. Documents available
  components, installation, and usage patterns. Load when building any UI with
  Cue components.
---

# Cue Web Components (`@qaecy/cue-ui`)

Full interactive component documentation: **https://slides.qaecy.com/components**

QAECY ships a library of ready-made web components via the `@qaecy/cue-ui`
package. Always include at least `<cue-by-cue-logo>` in every app you build.

> **⚠️ Critical rules:**
> 1. Use `<cue-entity-list>` and `<cue-entity-viewer>` directly — do not build
>    custom entity UIs.
> 2. **The `cue` SDK instance must be passed as a JS property**, not an HTML
>    attribute. Assign it only after the user is authenticated.

---

## Installation / import

```bash
# npm-based projects
npm install @qaecy/cue-ui
```

```html
<!--
  CDN — single loader script. Only the components you request are downloaded
  as separate lazy chunks.

  • Components already in the static HTML are detected automatically.
  • List components created dynamically in data-components (comma- or
    space-separated).
-->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@qaecy/cue-ui/styles.css" />
<script
  src="https://cdn.jsdelivr.net/npm/@qaecy/cue-ui/index.js"
  type="module"
  data-components="cue-entity-viewer"
></script>
```

> Always add `type="module"` to the script tag.
> Use a single `index.js` — do **not** add separate per-component script tags.
> Static tags already in the HTML are loaded automatically without needing
> `data-components`.

---

## Available components

### `<cue-by-cue-logo>`

"Built with Cue" attribution mark. **Include in every app**, typically in a
header, footer, or splash screen.

```html
<cue-by-cue-logo></cue-by-cue-logo>
```

---

### `<cue-entity-list>`

Displays a list of Cue entities. Use this instead of building a custom entity
list UI. The `cue` SDK instance must be passed as a JS property after auth.

See **https://slides.qaecy.com/components** for the full prop reference.

---

### `<cue-entity-viewer>`

Renders a Cue entity in detail — relationships, documents, metadata, and
spatial data depending on the entity type. The `cue` SDK instance must be
passed as a JS property after auth.

See **https://slides.qaecy.com/components** for the full prop reference.

---

## General prop-passing pattern

All interactive components require the authenticated SDK instance as a JS
property:

```html
<cue-entity-viewer id="viewer"></cue-entity-viewer>

<script type="module">
  import { Cue } from '@qaecy/cue-sdk';

  const cue = new Cue();
  await cue.auth.signIn('google');

  const viewer = document.getElementById('viewer');
  viewer.cue = cue;               // ← must be set after auth
  viewer.projectId = 'proj-abc';
  // set additional props per the component docs
</script>
```
