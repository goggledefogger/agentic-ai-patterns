---
type: pattern
date: "2026-05-13"
source: A hobby inventory vault
tags:
  - ui
  - dashboard
  - plugins
---

# DataviewJS for Custom UI Dashboards

Use the Dataview plugin's JavaScript API (`dataviewjs`) to query vault metadata and render highly customized, CSS-styled HTML interfaces directly inside Obsidian notes, bypassing the limitations of standard markdown or `.base` file views.

## The Problem

The official Obsidian "Bases over Dataview" rule generally holds true for simple tabular data: `.base` files are first-party and portable. However, `.base` files are structurally rigid. If you want to build a visual, card-based gallery—complete with conditional status badges (e.g., green for active, red for empty), calculated fields, custom SVG icons, and complex flexbox layouts—you hit a wall. Standard markdown and Bases simply cannot render dynamic, rich UI components.

## The Pattern

Treat a specific Obsidian note as an application frontend. Install the community **Dataview** plugin and enable "Enable JavaScript Queries." Then, use a ```dataviewjs``` code block to query your vault's frontmatter and programmatically construct custom HTML and CSS.

### 1. Querying the Data
Instead of standard Dataview DQL, use the JavaScript API to fetch and manipulate pages. This allows you to apply real programming logic—grouping arrays, sorting, and conditionally filtering data before rendering.
```javascript
// Query all filament notes, sort by color, group by material
const pages = dv.pages('#filament').where(p => p["color-name"]).sort(p => p["color-name"]);
```

### 2. Constructing the UI
Map over the queried pages and construct raw HTML strings. You can inject specific inline CSS, apply transition effects (`onmouseover`), and map frontmatter statuses to specific color hex codes or icons.
```javascript
const st = STATUS_META[p.status] || { color: "#888", label: p.status || "?" };
const imgTag = `<img src="${app.vault.getResourcePath(imgFile)}" style="width:100%;aspect-ratio:1;object-fit:cover;">`;

const cardHtml = `
  <div style="width:170px; background:var(--background-secondary); border-radius:10px; overflow:hidden;">
    <a href="${p.file.path}" class="internal-link" style="text-decoration:none;">
      ${imgTag}
      <div style="padding:10px;">
        <div style="font-weight:600;">${p["color-name"]}</div>
        <span style="background:${st.color}22; color:${st.color};">${st.label}</span>
      </div>
    </a>
  </div>
`;
```

### 3. Rendering
Append the final HTML payload directly into the note using `dv.paragraph()`. The result is a native-feeling, highly visual dashboard that instantly reflects any changes made to the underlying markdown files.

## When to use this

This pattern is the *justified exception* to the "stay lean and avoid Dataview" rule. 

**Use this when:**
- You need complex visual layouts (e.g., Pinterest-style masonry grids, product galleries).
- You require conditional formatting (e.g., turning a badge red if inventory drops below 50g).
- You want to render data that requires complex math or string manipulation prior to display.

**Avoid this when:**
- A simple table or list will suffice (use `.base` instead).
- You are trying to manage tasks or simple logs.

## Watch-outs

- **Plugin Dependency:** If Dataview ever breaks, the UI breaks. The underlying data in your markdown files is completely safe and portable, but the visualization layer is tightly coupled to the plugin.
- **Internal Link Routing:** To make the cards clickable in a way that Obsidian understands natively, wrap the cards in standard `<a href="${p.file.path}" class="internal-link">` tags. This ensures Obsidian opens the clicked card as an internal note rather than treating it like an external web URL.
