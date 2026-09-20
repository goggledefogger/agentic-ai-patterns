---
type: pattern
date: "2026-08-13"
source: The dashboard v7 shell — giving .btn a display:inline-flex for icon spacing un-hid every [hidden] button on the page: Stop rendered beside Send while idle, and Save-to-my-brain rendered under "Nothing waiting"
tags:
  - code
  - frontend
  - css
  - anti-pattern
---

# The Hidden Attribute Loses to Any Display

`hidden` is semantics enforced by the weakest layer in the cascade. The UA
stylesheet's `[hidden] { display: none }` is author-overridable by *any*
author rule that sets `display` — no `!important`, no specificity contest;
author origin simply beats user-agent origin. So the day a component gains
`display: inline-flex` for icon alignment, every element of that component
that JS hides with the `hidden` attribute quietly starts rendering, on every
screen, in every state.

Two properties make it nasty. It **fails open**: the things that appear are
exactly the ones designed to be absent — a Stop button next to Send while
nothing is streaming, action buttons under an empty-state message that says
"Nothing waiting." And it **arrives by unrelated diff**: the CSS change that
triggers it mentions neither `hidden` nor the elements it breaks, so review
reads it as cosmetic. The JS meanwhile keeps setting `el.hidden = true` and
observing (correctly) that the property is set.

## The Pattern

Re-assert the platform contract explicitly, once, in the base layer:

```css
[hidden] { display: none !important; }
```

- One global rule, in the reset/base stylesheet, with `!important` — this is
  the rare legitimate use: it restores a *semantic attribute's* meaning
  against accidental presentational override, and no author rule should ever
  win against `hidden` anyway. (An element that needs "hidden but styled
  for transition" should use a class, not the attribute.)
- The general shape: **a platform default is not a contract.** UA styles,
  library defaults, framework conventions — anything enforced only by "nobody
  has overridden it yet" dies silently the moment someone styles nearby.
  If your code *depends* on the default (JS toggling `hidden`, semantics
  read by tests), pin the default explicitly where you own it.
- Detection is visual, not logical: DOM inspection shows `hidden` present
  and asserts pass. Only rendering the page shows the ghost buttons — one
  more case for verifying on the surface the user actually uses.
