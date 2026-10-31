# Claude Design: Prompt for the Inspection App Redesign

**How to use**
1. Open Claude Design. Attach `00-READ-FIRST-full-context.md` (this repo's docs in one file). Optional but strongly recommended: 8–10 screenshots of the current app (Home list, Inspection hub, Photos grid, Camera, Segment, Car diagram, AI QC panel, Final approval, Login, Profile). **No screenshots of the running app were available when this was written.**
2. Paste **Prompt 1** and review. Then paste Prompts 2–6 one at a time; look before you stack.
3. Source of truth: `02-redesign-spec.md` (what and why) and `01-current-state-reference.md` (what exists). If a prompt conflicts with the spec, follow the spec and say so.

---

## Prompt 1: Master (paste first, with the attachment)

```
READ FIRST
Attached: 00-READ-FIRST-full-context.md. It holds the current-state reference of our internal inspector
app (from source code) and the redesign spec. Read all of it before designing. Treat the spec as the source
of truth. If anything below conflicts with it, follow the spec and tell me.

ROLE
You are the lead product designer for Khodro45 (خودرو۴۵), Iran's market-leading used-car company, known for
an AI-first approach. You design production-grade, right-to-left Persian mobile interfaces for people who
work outdoors, one-handed, sometimes with gloves, on mid-range Android phones and weak signal.

MISSION
Redesign "Copilot Inspection", the internal Flutter app our inspectors (کارشناس) and supervisors (سرپرست)
use to run a 145-point inspection of a used car, capture photos, defects, mileage and documents, and (for
supervisors) approve and publish the car to a live auction. It must follow the same design system and AI
grammar as our redesigned seller wizard, and be dramatically clearer, faster and safer than today.

WHAT IS WRONG TODAY (from the code audit)
1. The inspection screen has no progress. Inspectors discover what is missing only when "finish" rejects them.
   Segment "done" counts include components nobody reviewed.
2. Failures are silent: error toasts are switched off, several screens go blank or spin forever, photo
   uploads have no queue.
3. Two colour systems, four button styles, tap targets of 16-36 px, 10-13 px text for key instructions,
   always-open inline calendars, a 32 px diagram hit area, finger-tracing damage with no zoom or undo.
4. The supervisor's "publish" starts a live auction in one tap with no confirmation and no visible prices.
   Legal-replacement answers silently default to "no".

DESIGN PRINCIPLES (non-negotiable)
1. Always know where you are: persistent readiness (what is done, blocking, optional), next best step.
2. Block with a reason, never silently: every disabled primary action says why and how many remain.
3. Propose -> confirm: AI fills or flags, the person confirms with one tap. Nothing inferred is saved as fact
   without a visible confirm.
4. Visible AI boundary: purple = AI output, neutral = records, orange = action, green = done, red = error or
   blocking, amber = needs attention. One meaning per colour.
5. Honest uncertainty: confidence in words and number; "not checked" when evidence is missing; AI outage is a
   neutral fallback and never blocks work.
6. Built for the real device: targets >= 48 dp (primary 56), key text >= 14 sp, high contrast outdoors,
   actions in the thumb zone, haptics, no decorative motion.
7. Nothing is lost: captures survive weak signal and restarts, and the state is visible.
8. Dangerous actions are deliberate: publish, cancel, return-to-inspection, trim overwrite show the
   consequence and need explicit confirmation.

DESIGN SYSTEM (use exactly; extend, do not replace)
- Language: Persian, RTL. Text font Vazirmatn 400/500/700; numerals (km, timers, counts, plates) in
  IRANYekan FaNum (Persian digits). LTR islands for plate, phone, timer.
- Type (dp/line): headline 20/32 semibold, title 16/24 semibold, body 16/24, label 14/20 (minimum for
  instructions), caption 12/16 (non-essential), numeral hero 32/40 bold, timer 18/24.
- Colour: action #FF9533 (pressed #E8882E, tint #FFF4EB, text-on-tint #8C521C); text strong #252B32, muted
  #5D6C7E; border #C6D0DC; surface #F9FAFB; page #FFFFFF; done #1BA75E (tint #E8F6EF); AI #B642FF (tint
  #FBF4FF, gradient #AD33FF->#CA76FF); error #EF3B2E (tint #FDEBEA); attention #FFC107 (dark text only);
  info #0A9EC2 (in-progress only); app identity #24536C (splash, app bar, brand mark only).
- Primary button text: dark #252B32 on #FF9533 (6.6:1). Text >= 4.5:1, UI >= 3:1. Provide a high-contrast
  variant of outlines for the camera and diagram screens (bright outdoors).
- Shape: radius 8 (buttons, inputs, tiles), 16 (cards, sheets), pill (chips). 4 dp grid, 16 dp gutters.
  Mostly flat; card shadow 0 1 4 rgba(0,0,0,.04); sheet 0 8 16 rgba(0,0,0,.16).
- Icons: simple line icons; car art as in the app's top-down diagram and side guides.
- Motion: selection 150 ms cubic-bezier(.4,0,.2,1); sheets 200 ms slide-up; page 250 ms slide (RTL: enters
  from the left) + fade; progress 300 ms ease-out; AI glow border only while a request is pending; result
  fade-zoom 0.4 s; reduced-motion = instant. No confetti, no fake "thinking".

AI GRAMMAR (reusable components)
- AI chip: purple outline, "پیشنهاد هوش مصنوعی · ۹۲٪", tap to apply; applied value keeps a small AI tag
  until edited.
- Confirm card: purple 1px border + tint, "درسته" / "ویرایش".
- QC flag card: thumbnail, reason chips in plain Persian, blocking = red outline, advisory = amber.
- Low confidence: dashed purple border + "مطمئن نیستیم؛ لطفاً بررسی کنید".
- Working: rotating purple glow border only while pending. Unavailable: neutral grey, no purple, manual path.
- Derived numbers (damage area grade): purple "محاسبه‌شده" with inputs visible.

THE FLOW (mobile 375x812; camera in landscape)
Home (role tabs: مدارک / کارشناسی / تایید نهایی) -> Inspection run -> Finish. Inspection is a guided run, not
a rigid wizard: steps can be done in any order, but readiness, blockers and the next best step are always
visible. Steps: car identity, documents, car details form, photos and videos, mileage, server-driven
inspection segments (components -> defects -> media -> grade), review and finish (AI QC -> confirm -> submit).
Role-gated steps appear locked with a reason, not hidden.

THIS ROUND: DELIVERABLES
1. One-page design system sheet: colour roles with contrast ratios, type scale, spacing, radius, tap-target
   rules, and the AI components in all states.
2. Core component sheet: button (all states incl. disabled-with-reason), app bar, bottom action bar with timer
   and blocker summary, segmented tabs, search, list card with progress ring, status chip, stepper row,
   field row, option tile, bottom sheet, confirm dialog, banner, and the state kit (skeleton, empty, error
   with retry, offline).
3. Three directions for the INSPECTION RUN screen (the hub) and the HOME list card:
   A "Checklist": a vertical readiness checklist with a ring at the top.
   B "Car-centric": the car illustration in the centre with zone markers showing progress, checklist below.
   C "Focus mode": one next-best-step card plus a compact overview.
   For each: one sentence why it fits field work, and its main risk.
4. Self-check against the spec's acceptance checklist (section 12). State any check that fails and why.

SELF-CHECK BEFORE YOU ANSWER
- Is every AI element purple and every action orange? Is anything green that is not "done"?
- Does every disabled primary say why? Are empty, error, offline and loading four different designs?
- Are all targets >= 48 dp and instruction text >= 14 sp? Contrast >= 4.5:1 incl. the primary button?
- RTL correct (chevrons, progress direction, LTR plate / km / timer)?
```

## Prompt 2: Full flow in the chosen direction

```
We choose Direction [A/B/C] with these decisions: D1 [orange action / keep teal], D2 [font], D3 [segment
"reviewed" signal], D4 [legal replacement explicit choice]. Design the screens in spec section 7 at 375x812
(camera at 812x375), in that direction:
7.1 Splash/Login/global states, 7.2 Home (3 role tabs), 7.3 onsite visit sheet, 7.4 Profile,
7.5-7.6 Inspection run + car identity, 7.7 car details form (incl. plate widget, date sheet, legal
replacement tiles), 7.8 photos grid + upload tray, 7.9 guided camera (with slot strip, retake, undo, flash
modes), 7.10 mileage sheet.
Show three connected storyboards:
1. Happy path: start -> identity confirmed from AI -> photos via guided camera -> mileage -> segments ->
   AI QC clear -> confirm -> done.
2. Trouble path: weak signal (uploads queued), AI flags a wrong-angle photo and a visible plate, one
   required photo missing, the end flow is blocked with a reason and tap-to-fix.
3. Supervisor path: documents tab -> hub (locked inspector steps) -> approve.
Keep the design system and AI components unchanged; list any new component you had to add.
```

## Prompt 3: Defects, diagram and AI QC

```
Design spec 7.11-7.15: segment screen (category chips, accordions, component and defect rows with the
severity WORD, required-media status, grade rails 1-5 with words), the car diagram (>= 48 dp hit areas, a
magnified selector for thin pillars, replaced panels with pattern + tint, markers 40 dp, inline legend), the
trace-damage screen (pinch zoom, brush size, undo last region, a one-tap default-region fallback for gloves,
live derived area grade in purple), the AI QC panel (analyzing, results, all clear, failed/unavailable, flag
cards with 48 dp actions, disabled-with-count continue button), the finish confirm sheet (editable summary
rows, consequence box), and cancel (reason -> required proof photos -> destructive confirm).
Include high-contrast outdoor variants for the diagram and camera overlay.
```

## Prompt 4: Supervisor final approval and publish

```
Design spec 7.16: final approval with header progress ("۳ از ۸ مورد تایید شده"), photo-issue cards with
acknowledge + undo, document cards with expected vs AI value, inline edit and re-upload, trim and option
card with the Q&A sheet and a mismatch state with a confirmed "apply fix", a sticky decision bar
(publish disabled with "n مورد باقی‌مانده", not-publish with reason sheet, return-to-inspection in an
overflow with confirm and reason), and the PUBLISH CONFIRM sheet (start time, duration, start/target price
and model confidence if allowed, the consequence "مزایده شروع می‌شود", explicit confirm). Show the variant
without prices (decision D7 = no) as well.
```

## Prompt 5: States, edge cases and motion

```
Produce state boards: every AI component (working, result, low confidence, unavailable); photo slot tile
(empty, required-missing, uploading, uploaded, failed, AI-flagged); inputs (empty, typing, valid, error,
AI-filled); upload tray (queued, retrying, failed); offline banner with manual retry; lists (skeleton,
empty, error, offline with cached list); update gate (forced, optional); disabled account; wrong password;
session expired. Use real Persian copy. Mark states needing a backend decision with a "needs decision" tag.
Then write the motion spec as annotated frames per the system's motion rules, with reduced-motion fallbacks.
```

## Prompt 6: Self-critique, then handoff

```
Critique your own designs as a principal designer at an AI-first market leader. Score 0-2 and fix anything
below 2: (1) Progress and orientation, (2) Honest errors and states, (3) AI boundary and uncertainty,
(4) Dangerous-action safety, (5) Thumb reach and tap targets, (6) Outdoor legibility, (7) Consistency
(one meaning per colour, components reused), (8) RTL correctness. Return the scores and the revised screens.
Then prepare the developer handoff for Flutter/GetX: component inventory with props and states; token table
(primitive -> semantic -> component) as a ThemeExtension spec; a migration table mapping every legacy colour
and text style to a token; the Persian copy deck with keys and English gloss; the analytics events from
spec section 11; the list of Appium ids that must be preserved (spec section 10); open decisions.
```

## Acceptance checklist (before showing the team)

See `02-redesign-spec.md` section 12. In short: readiness always visible; purple only AI; every disabled primary explains itself; four distinct states on every screen; targets >= 48 dp; contrast passes (incl. outdoor variants); dangerous actions confirmed; RTL correct; every legacy token mapped.
