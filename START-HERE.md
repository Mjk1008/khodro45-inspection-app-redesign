# START HERE (instructions for Claude Design)

You are designing the redesign of Khodro45's internal inspector app. Everything you need is in this repository. There are **no screenshots of the current app**; use the text wireframes and the assets.

## Read in this order
1. `03-claude-design-prompt.md` → **Prompt 1 (master)**: role, mission, principles, design system, deliverables. Follow it exactly.
2. `02-redesign-spec.md` → source of truth (principles, tokens, flow, per-screen specs in §7, AI grammar §8, decisions §9, acceptance checklist §12).
3. `01-current-state-reference.md` → what exists today, rules that must not break, risks.
4. `04-current-screens-wireframes.md` → the current screens as text wireframes (what you are replacing).
5. `appendix/` → file:line detail when you need exact copy, states or rules.

## Assets you can reuse (real app files)
- `assets/car-parts/` 19 body-panel SVGs (+ `_red` variants) and `assets/data/car_svg_part_pixel.json` (panel hit-box geometry as fractions of the diagram).
- `assets/camera-guides/` 19 white outline guides used over the viewfinder (front, sides, corners, tyre, engine, boot, console, 360).
- `assets/brand/` Khodro45 logo, AI45 mark, plate graphic, plate-cover sticker.
- `reference/seller-wizard/` six screenshots of the seller-wizard redesign's source product, for the visual language (AI purple, orange actions, range card). Take the *grammar*, not the layouts.

## Rules
- Persian, RTL, mobile 375×812 (camera landscape). Orange = action, green = done, purple = AI, red = error/blocking, amber = attention.
- Redraw the app's UI; do not invent business rules. Server-driven content (segments, components, defects, form fields) must stay data-driven in the design (show realistic example data).
- Where the spec lists a **[Decision] D1–D9**, show the default and the main alternative.
- Do the work in the order of the prompts (1 → 6). Finish and show Prompt 1 deliverables first, then wait for feedback.

## Output
Put design output in `design/` in this repo (HTML/Figma export, component sheet, token table, copy deck). Name files by prompt number (e.g. `design/01-system-and-directions/`).
