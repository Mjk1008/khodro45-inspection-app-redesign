# Inspection App Redesign: Spec (PRD v0.1)

**Status:** draft for review · **Date:** 2026-10-31 · **Author:** AI PM (Khodro45)
**Inputs:** [01-current-state-reference.md](01-current-state-reference.md) (what exists, from source), the seller-wizard redesign system (`khodro45-wizard-redesign`: tokens, AI colour grammar, propose→confirm, motion, accessibility), the AI-native framework (Assisted/Copilot/Native; Intent → Context → Reason → Decide → Execute → Verify).
**Labels:** **[Code]** read in the source · **[Inference]** my reading · **[Assumption]** not confirmed internally · **[Decision]** needs an owner.
**Scope:** UI/UX redesign of the Flutter inspection app. Business rules, endpoints and server-driven structure stay. Items that are not UI work (R13, R14 in doc 01) are flagged, not specified.

---

## 1. Problem

Inspectors do a physical 145-point job outdoors, one-handed, often with gloves, on mid-range Android and weak signal. The app they use:
1. **Does not show where they are.** The hub has no progress; "done" counts are misleading; the end flow is the first moment gaps appear [Code].
2. **Fails silently.** Error toasts are globally disabled; several screens go blank or spin forever; photo uploads have no queue [Code].
3. **Looks and behaves inconsistently.** Two colour systems, four button styles, tap targets 16–36 px, 10–13 px text for key instructions [Code].
4. **Hides risk at the supervisor gate.** Publish starts a live auction in one tap with prices nobody sees in the app; several legal/price-relevant defaults are silent [Code].

The seller wizard was redesigned with a clear system (honest AI, propose → confirm, one meaning per colour, real-device constraints). The inspection app, which produces the data that price and auction depend on, still uses none of it.

## 2. Goals, non-goals, metrics

**Goal:** an inspector can finish a correct, complete inspection faster and with fewer rework loops, and a supervisor can approve with the evidence in front of them, in an app that follows the same design system as the seller wizard.

**Non-goals:** new backend features; changing inspection content (segments/components/defects stay server-driven); replacing the diagram art; dark mode (check only, R4); tablet layouts; changing roles or permissions.

### Metrics (each with how to measure before/after) [Assumption: events do not exist yet; R0 adds them]

| Metric | Target direction | Baseline source | Notes |
|---|---|---|---|
| **Median inspection duration** (start_inspection → end_inspection) | ↓ ≥10 % with no rise in re-inspection | App already shows per-inspector average time (profile) [Code]; need server timestamps `inspection_start_at/end_at` [Code] | **North-star.** UI alone may move this little because the physical inspection dominates [Assumption]; treat as guardrail-first |
| **End-flow rejections per inspection** (`end_inspection` 4xx) | ↓ | New event | Direct effect of visible progress + blockers |
| **AI QC flags per inspection, and share re-shot** | ↓ flags; ↑ re-shoot before end | ai-lens responses | Guided capture should reduce wrong-angle/plate flags |
| **Photo upload failure → abandoned** | ↓ | New event | Needs the persisted queue (engineering) |
| **Re-inspection rate** (profile `re_inspection.percent`) | not ↑ | Existing [Code] | Guardrail |
| **Supervisor time-to-publish** and **publish reversals** (return to inspection after publish attempt) | ↓ / not ↑ | back_office status changes | Evidence-first approval |
| Tap-error proxy: undo / delete-photo rate, wrong-defect removals | ↓ | New event | Larger targets |

## 3. Users and context

| Persona | Context | Needs | Notes |
|---|---|---|---|
| **Branch inspector** | In a branch bay, one phone, light varies, 20–30 min per car (timer target: green <20 min, amber <30 [Code]) | Know what's left, capture fast, never lose a photo | Primary |
| **Onsite inspector** | At the owner's address, weaker signal, map/route needed | Same + visit details | Same flow, different list |
| **Supervisor** | At desk/phone, reviews AI flags, completes documents, publishes | See evidence, decide, undo mistakes | Final approval + documents tab |

**Device [Assumption]:** mid-range Android, 360–412 dp wide, portrait (camera quick-shoot is landscape). Design on 375×812.

## 4. Product principles (adapted from the wizard system)

1. **Always know where you are.** Persistent readiness view: what's done, what's blocking, what's optional.
2. **Block with a reason, never silently.** Every disabled primary action says why and how many remain.
3. **Propose → confirm.** AI fills or flags; the inspector confirms with one tap. Nothing inferred is saved as fact without a visible confirm.
4. **Visible AI boundary.** Purple = AI output (OCR fills, QC flags, suggestions). Neutral = records. Orange = action. Green = done. Red = error/blocking. Amber = needs attention.
5. **Honest uncertainty.** Show confidence as words + number; show "not checked" when evidence is missing; AI outage = neutral fallback, never blocks.
6. **Built for the real device.** Targets ≥48 dp (primary 56), key text ≥14 sp, high contrast outdoors, thumb-zone actions at the bottom, haptics for confirm/undo, no decorative motion.
7. **Nothing lost.** Every capture survives weak signal, screen close and app restart (queue + visible state).
8. **Dangerous actions are deliberate.** Publish, cancel, return-to-inspection, trim overwrite: summary, consequence, confirm. Which layer enforces is stated (§9).

## 5. Design system (extend the wizard tokens)

### 5.1 Colour roles (one meaning per colour)
| Role | Value | Use |
|---|---|---|
| Action | `#FF9533` (pressed `#E8882E`, tint `#FFF4EB`, text-on-tint `#8C521C`) | Primary buttons, selected state outline |
| Text strong / muted | `#252B32` / `#5D6C7E` | Never use `secondary-300` for text |
| Border / surface | `#C6D0DC` / `#F9FAFB`, page `#FFFFFF` | Replaces the legacy `#F6F6F6` |
| Done | `#1BA75E` (tint `#E8F6EF`) | Completed step, OK |
| **AI** | `#B642FF` (tint `#FBF4FF`, gradient `#AD33FF→#CA76FF`) | AI-filled value, QC flag, suggestion |
| Error / blocking | `#EF3B2E` (tint `#FDEBEA`) | Missing required, failed |
| Needs attention | `#FFC107` (dark text only) | Advisory, "review" |
| Info | `#0A9EC2` | In-progress state only |
| **App identity** | `#24536C` (teal-navy) | Splash, top app bar background, brand mark only (**[Decision] D1**) |

CTA text: dark `#252B32` on `#FF9533` (6.6:1). All text ≥4.5:1, UI ≥3:1. **Check outdoors:** provide a high-contrast variant of borders/outline on the camera and diagram screens.

> **D1 [Decision]:** the current app's brand colour is teal `#24536C`, the wizard's action colour is orange. Recommendation: orange for actions (consistency with the seller product and with the "orange = act" grammar), teal kept as identity. Risk: inspectors lose the familiar teal CTA. Mitigation: staged rollout, release note, teal stays on the app bar. Alternative: teal action + orange for attention (breaks the wizard grammar).

Retire: `AppColors.primaryColor_50…500` (orange named "primary"), `#1098F7`, `#71C544/#62B237`, `#F03A47/#FF4869`, `#847373`, all one-off hexes. Map every legacy token to a role in the migration table (handoff).

### 5.2 Type
- **Family [Decision] D2:** Vazirmatn 400/500/700 for text (already bundled) + the existing **IRANYekan FaNum** as the numeral face for km, timers, counts, plates, prices (replaces "Vazirmatn-FD", which the app does not bundle). Single family would also be acceptable; do not keep three.
- Scale (dp/line): headline 20/32 semibold · title 16/24 semibold · body 16/24 · label 14/20 (minimum for instructions) · caption 12/16 (non-essential only) · numeral hero 32/40 bold · timer 18/24.
- Remove: theme `bodyLarge` 40 italic; w900; ad-hoc `copyWith` sizes.

### 5.3 Shape, space, elevation
Radius 8 (buttons, inputs, tiles) · 16 (cards, sheets) · pill (chips). 4 dp grid; screen gutter 16 (not 11). Mostly flat; card shadow `0 1 4 rgba(0,0,0,.04)`; sheet `0 8 16 rgba(0,0,0,.16)`.

### 5.4 Touch and ergonomics
Min target 48 dp; primary and camera actions 56; inter-target gap ≥8; primary action in the bottom 40 % of the screen; snackbars/toasts at the **bottom above the action bar**, never under the status bar; haptic on select, confirm, undo, error.

### 5.5 Motion
Selection 150 ms `cubic-bezier(.4,0,.2,1)`; sheets slide-up 200 ms; step/page transition 250 ms (RTL: enters from the left) + fade (the app currently has none); progress 300 ms ease-out; AI glow border only while a request is pending; result fade-zoom 0.4 s (scale .9→1). Reduced motion → instant/opacity. No confetti, no fake "thinking".

### 5.6 Core component set (build once, reuse)
Button (primary/secondary/tertiary/destructive, loading, disabled-with-reason) · Top app bar (one build) · Bottom action bar (timer + primary + blocker summary) · Card · Status chip · Segmented tabs (full width, counts) · Search field (live) · List item with progress ring · Section header · Field row (label/value/source/confidence/edit) · Text/number/plate/dropdown/date inputs · Option tile · Chip · Switch (platform) · Progress stepper + readiness checklist · **AI components:** AI chip (suggestion), confirm card, QC flag card, low-confidence dashed state · Photo slot tile (states) · Upload progress/queue row · Bottom sheet (standard, 16 radius, 48 close target) · Confirm dialog (consequence + typed/explicit confirm for dangerous) · Banner (offline, pending uploads, role, staging env) · State kit: **loading skeleton / empty / error+retry / offline** · Toast (bottom).

## 6. Information architecture and flow

### 6.1 Navigation model
Keep three bottom/primary areas by role; do not add navigation depth.
- **Home** (tabs by role) → **Inspection** (readiness hub + steps) → **End flow**.
- Inspection becomes a **guided run with a persistent readiness bar**, not a rigid wizard: steps can be done in any order (matches how inspectors move around the car) but progress, blockers and the next best step are always visible.

### 6.2 The new Inspection run (the "wizard")
Top: car identity (logo, year·model·brand, plate, appointment id) + timer (18 sp, amber at 20 min, red at 30 [Code thresholds]) + **readiness ring "۶ از ۹ مرحله"**.
Steps (server-driven where noted), grouped in a vertical stepper/checklist, each row = icon, title, status (not started / in progress / done / needs attention / blocked), one-line detail (e.g. "۱۲ از ۱۸ عکس"), chevron:

| # | Step | Source | Gate to end? | Role |
|---|---|---|---|---|
| 1 | Car identity (info) | brand/model/trim/year/option; AI-filled values purple | Yes | inspector, supervisor |
| 2 | Documents | `inspection_vehicle_documents` slots | Yes (supervisor path) | supervisor |
| 3 | Car details form | server JSON | Yes (supervisor path) | supervisor |
| 4 | Photos & videos | image templates; guided capture | Yes (required slots) | inspector |
| 5 | Mileage | km + odometer photo + note | Yes | inspector |
| 6…n | Inspection segments | `/segment/`, ordered by server | Yes via `validate_defects` | inspector |
| last | Review & finish | AI QC → confirm → submit | — | inspector / «تایید» supervisor |

**Status of a segment (fixes U4):** *not started* (no component reviewed), *in progress*, *done (n defects)*, *needs attention (n missing photo/location)*. Requires a "reviewed" signal per component; today only defect counts exist [Code]. **[Decision] D3:** either the backend exposes `reviewed_count`, or the client tracks "opened/visited" per segment locally (weaker, labelled as such).
**Blocker summary** sits above the finish button: "۳ مورد باقی‌مانده: ۲ عکس اجباری، کارکرد" with tap-to-jump. Replaces the end flow being the first time gaps appear. Same list powers the existing remediation screen.

### 6.3 Roles
Show locked, not hidden: a role-gated step appears greyed with "توسط سرپرست تکمیل می‌شود" so inspectors understand why (today cards vanish; null flags hide almost everything [Code]). Keep `editable_by_*` as the source.

## 7. Screen specs

> Common to every screen: RTL; loading = skeleton after 300 ms; empty vs error vs offline are three distinct states; error shows plain Persian cause + **retry**; the primary action is disabled only with a stated reason; Appium ids preserved (§10).

### 7.1 Shell
- **Splash:** brand mark on `#24536C`, native splash and Flutter splash identical (no teal→white flash); remove the fixed 2.5 s wait (wait only for the update check, ≤ its timeout); show a slim "در حال بررسی نسخه" only if >1 s. Update gate: forced = blocking page with changelog + button; optional = dismissible sheet.
- **Login:** username/personnel code + password; show-password toggle; inline errors incl. **wrong password** ("نام کاربری یا رمز عبور درست نیست") and network error (today silent [Code]); disabled-account sheet (keeps 1485 ext. 280 call button); forgot-password sheet with call button. Staging: server switcher moves into a debug sheet, not the main form.
- **Global states:** offline banner (single, with pending count, **manual retry button** inside the offline sheet); staging badge becomes a small corner chip instead of a full-width strip.

### 7.2 Home
- Full-width **segmented tabs** with counts: «بارگذاری مدارک», «کارشناسی», «تایید نهایی» by role; selected = orange underline + strong text; count pill neutral until non-zero action needed.
- **Search:** one live field («شناسه قرار یا کد پذیرش / پلاک»), clear button, no separate button; same behaviour across tabs.
- **Filter** (کارشناسی): two segments «در انتظار» / «در حال کارشناسی» with counts (keep logic).
- **List card (redesign):** top row status chip + elapsed/created time (with year); car identity line **brand · model · year** in reading order (no LRM hack), logo 40; plate graphic (existing component) + appointment id; branch or "در محل"; owner name; **progress ring for in-progress** (uses `templates_data.percent` already returned [Code]); one primary button (شروع / ادامه / تایید نهایی / بارگذاری مدارک) at 48 dp; "گزارش کارشناسی" as secondary text button; onsite: round map button 48 dp.
- **States:** skeleton cards; empty ("موردی برای نمایش نیست" + why); error ("خطا در دریافت فهرست" + retry); offline (cached list + banner) [cache is engineering work; design the state].
- **Pull-to-refresh** retained with Persian copy.

### 7.3 Onsite visit sheet
Header «جزییات مراجعه»; customer + call button; date with **calendar icon** (fix) and remaining-time tag; address + static map + «مسیریابی»; primary «شروع کارشناسی» / «ادامه کارشناسی» by status. Map tile provider key issue is engineering (R14).

### 7.4 Profile
Header (name, personnel code, rating tag); **stats as one card** with 4 tiles (today, this month, re-inspection % in amber/red, average time with the 20/30-min colouring [Code]); menu rows with correct icons; app version row; logout as destructive text button at the end (confirm sheet). Branches and My-inspections: same list pattern, error state added, search normalises digits like other lists.

### 7.5 Inspection hub / run (see §6.2)
Header, readiness ring, stepper list, blocker summary, bottom bar: timer + «پایان کارشناسی» (inspector) or «تایید» (supervisor). Button disabled state shows "n مورد باقی‌مانده". Cancel for supervisor moves to an overflow menu with a destructive confirm (it is irreversible [Code copy]).

### 7.6 Car identity (info)
One screen, cascade Brand → Model → Trim → Year → Option as **searchable bottom sheets** (search added; no full-page spinner on each pick; inline skeleton on the dependent field). AI-suggested values purple chip "پیشنهاد هوش مصنوعی · ۹۲٪" with one-tap apply; applied = purple tag "پیشنهاد AI" until the inspector edits (then neutral). Required fields show inline error under the field, not a Stack badge. Save disabled until all five are real selections (fix `trimyear:0` [Code]); failure shows a message.

### 7.7 Car details form (server-driven)
- Sections as cards with a header showing completion ("۴ از ۷"). **Collapse sections**; only the active one is expanded.
- **Fields:** text/number with stable controllers (fix the typing bug [Code]); thousands separators; uppercase where required; radio as large segmented tiles; dropdown as sheet with colour swatches for body colour; **date: collapsed field that opens a bottom-sheet Jalali/Gregorian picker** (replaces the always-open inline calendar; no "current year − 3" jump).
- **Plate:** keep the 4-part plate widget (2 digits · letter sheet · 3 digits · 2 digits + «ایران»), larger boxes (min 48 dp), letter sheet with V/D/S; auto-advance focus.
- **Legal replacement (cabin/chassis/engine): no default.** Three explicit tiles (دارد / ندارد), required, red until chosen (**R3, [Decision] D4**; if product keeps the default, the screen must show it as "پیش‌فرض: ندارد — تایید کنید" with an explicit confirm).
- AI values (document OCR) purple with confidence + "اعمال" chip; "برگه سبز خوانده نشد" fallback state.
- Validation on save: scroll to and focus the first error; summary banner "۲ فیلد ناقص".
- Leaving with changes: «ذخیره / خروج بدون ذخیره» sheet (keep).

### 7.8 Photos and videos
- **Header:** progress "۱۴ از ۲۰ عکس اجباری" + a single prominent «عکاسی سریع» button (56 dp, orange).
- **Grid** of slot tiles (2 columns, tile ≥ 140 dp high). States: empty (guide icon), required-missing (red outline + "اجباری"), uploading (progress ring on thumbnail), uploaded (thumbnail + small check), failed (red + **48 dp retry**), AI-flagged (purple badge). Tap uploaded → preview; edit (re-shoot) is a labelled button inside preview, not a 32 px chip.
- **Preview sheet:** replace, annotate («علامت‌گذاری»; rename from «شخصی سازی»), delete (confirm), close at 48 dp.
- **Upload tray:** a persistent bottom banner "۳ عکس در صف ارسال" with retry-all; survives screen close (**requires persisted queue, engineering; R11**).
- **Privacy [Decision] D5:** stop auto-saving every photo to the phone gallery (R10), or make it a setting with default off.

### 7.9 Guided camera («عکاسی سریع»)
Landscape. Viewfinder full-bleed 4:3 as today. **Overlay:** white guide outline **with dark outline stroke** (readable on bright sky/paint), label pill "نمای جلو · ۳ از ۲۰", slot strip of thumbnails along the long edge to jump back and **retake** any slot (today filled slots cannot be retaken from here [Code]). Controls: shutter 88, back, skip (only if `can_skip`), flash **auto/on/off + torch for engine/boot**. After capture: 400 ms thumbnail confirm with **undo** before moving on. Failure (size, ratio, duration) shows a message (today silent [Code]). Rotate-phone prompt stays. Video slots (`360`, `engine-video`): same shutter turns red, countdown 30→0, min 10 s with live message "حداقل ۱۰ ثانیه".

### 7.10 Mileage
A bottom sheet (keep as sheet): the odometer **photo frame is fully tappable**, then a large numeric field with live thousands separator and unit, then optional note. Validation inline: required, plausible range (**[Decision] D6** use `car_detail_meta.mileage_range` already returned [Code]; out-of-range → amber "عدد غیرعادی است، تایید می‌کنید؟", not a hard block). AI chip if OCR proposes a value from the photo [Assumption: no such service in code today; design as optional state].

### 7.11 Segment, components, defects
- **Segment screen:** category chips as horizontal scroller with counts, search, collapsed category accordions with badge "n مورد". **Component row:** name, shield if guaranteed, defect count; expanded → description field (stays enabled; explain if locked), defect list.
- **Defect row:** name, severity indicator (icon + colour from server **plus the `severity_text` word**, which the API provides and the UI never shows [Code]), toggle (platform switch, 48 dp row).
- **Defect detail:** required-media status line ("عکس اجباری ✕/✓" using the server `required_media_text`), media grid 3 cols with 48 dp add tile, note field, location status ("محل مشخص نشده" + button).
- **Grade rails (body & tire):** two 1–5 rails with words ("کم … زیاد"), cells ≥48 dp, preview of the effect, «تایید» persistent; area prefilled from the drawn region shown as purple "پیشنهاد از ترسیم · ۳".
- **Submit «ثبت اطلاعات»:** validation result → jump list "نقص بدون محل (۲) / بدون عکس (۱)" with tap-to-fix (replaces the title that always says "without location").
- Info/legend is shown in context (inline legend row at top, dismissible), not only behind an app-bar icon.

### 7.12 Car diagram and defect location
- Diagram scales to full width; **part hit areas ≥48 dp** (thin pillars get a magnified selector sheet "ستون‌ها" instead of tiny boxes); markers 40 dp with severity colour and icon; replaced panels use a **pattern + tint** (not colour alone). Legend inline.
- Tapping a part → bottom sheet listing its components/defects (one step instead of two), then photo, then location.
- **Trace damage:** zoom/pan (pinch) on the panel, brush size selector, **undo last region**, "clear all", live "مساحت: ۳ از ۵" in purple (derived value), confirm. A tap fallback "نقطه‌ی مرکزی" for gloves: one tap places a default-size region, adjustable by handle. Hint animation shows a draw gesture (not a tap).
- Back from the component list keeps context (no `Get.off`); the system back pops one step.

### 7.13 AI QC panel (before finishing)
Purple AI header + honest copy. States: analyzing (real progress only; max ~30 s, "کمی بیشتر طول می‌کشد" after 10 s) · results · all clear ("موردی یافت نشد") · failed ("بررسی انجام نشد" + «تلاش مجدد» + «ادامه بدون بررسی», never blocks).
Flag card: 120×90 thumbnail (zoomable), reason chips with **plain Persian** (wrong angle / plate visible / person in photo), severity (blocking = red outline, advisory = amber), actions at 48 dp: «عکس مجدد», «تایید است» (confirm dialog «این تصویر بدون اصلاح ثبت می‌شود»), «افزودن لوگوی پلاک». Bottom button states: «ادامه» when all blocking resolved; otherwise **a visibly disabled button with "n تصویر باقی‌مانده"** (today it looks active and does nothing [Code]).

### 7.14 Finish: confirm
Summary card (editable rows: tap a row to jump and fix, then return) with identity, mileage, colour (+AI tag), note; checkbox «اطلاعات را تایید می‌کنم»; consequence line «پس از تایید تا زمان QC امکان ویرایش ندارید» in amber box; primary «تایید و پایان». After a remediation round the **same confirm is shown again** (R4 fixed in UI; client flow change flagged to engineering).

### 7.15 Cancel (supervisor)
Reason picker → shows which proof photos the reason needs (server-driven, not hard-coded IDs; flag R12) → destructive confirm that names the consequence → on failure show the error. Never delete shared document photos from this screen.

### 7.16 Final approval (supervisor)
- **Header:** car identity + progress "۳ از ۸ مورد تایید شده".
- **Sections** as collapsible cards: photo issues (thumbnail, issue chips in AI purple, acknowledge button 48 dp with **undo**), documents (expected vs AI value side-by-side, inline edit field or re-upload, status), trim & option (see below).
- **Trim Q&A:** non-auto-opening sheet launched from the trim card (opens automatically once, with a skip-for-now); result of the answer API shown; mismatch card with before/after and «ثبت اصلاح» behind a confirm (R8).
- **Decision bar (sticky):** one primary «انتشار» (disabled with "۵ مورد باقی‌مانده" and tap to scroll to the first), secondary «عدم انتشار», and «برگشت به کارشناسی» moved to an overflow menu with its own confirm + reason (R2).
- **Publish confirm sheet (new, R1):** shows exactly what will happen: start time, duration, **start/target price if the product allows showing them** (**[Decision] D7**: today the supervisor never sees prices in-app), model confidence if available, and the consequence "مزایده شروع می‌شود"; confirm button requires an explicit press. Start time should come from the server clock (backend). **Layer:** the UI confirm informs; the **backend must validate** price sanity and caller role (UI is not a control).
- **Not published:** reason sheet (required) + optional photo.
- **Audit:** acknowledgements are saved server-side with user and time (**backend, R9**). Until then, label them as "local".

## 8. AI grammar for this app

| Element | Treatment |
|---|---|
| Value proposed by AI (car identity, OCR fields, plate logo, area suggestion) | Purple outline/tint chip with confidence band (**بالا / متوسط / پایین** + %). Tap = apply. Applied value keeps a small AI tag until edited |
| Flag from AI (QC) | Purple icon + red/amber outline for blocking/advisory |
| Low confidence (≤80 %) | Dashed purple border + "مطمئن نیستیم؛ لطفاً بررسی کنید" |
| AI working | Rotating purple glow border **only while the request is pending** |
| AI unavailable | Neutral grey card, no purple, manual path visible, never blocks |
| Not checked | Say so: "بررسی نشده". Absence of evidence is never shown as a pass |
| Derived numbers (area grade, percent) | Purple "محاسبه‌شده" with the inputs visible |

**Authority table (who may assert what):**
| UI may say… | Only if… | Enforced by |
|---|---|---|
| "تصاویر اجباری کامل است" | server template list shows no required-missing | Client reads server data; server re-validates at `end_inspection` [Inference: confirm] |
| "بررسی هوشمند انجام شد" | ai-lens returned a result for this inspection | AI service |
| "آماده انتشار" | all acknowledgements saved + trim matches | Client gate **and** backend |
| "مزایده شروع شد" | `change_status` succeeded and was re-read | Backend (System of Record) |

Mapped to the framework: today the app is **Assisted** (AI flags, human decides, old workflow). The redesign moves it to **Copilot** (visible propose → confirm, one readiness view), not Native; we do not claim more. Rubric snapshot (0–2): Intent 1 (goal is fixed by the appointment) · Context 1 · Capabilities 1 · Execution 1 · Authority 1 · Evaluation 0 → **5/12 target after R2**, evaluation only moves with R0 instrumentation.

## 9. Decisions needed (owners to assign)

| ID | Decision | Default in this spec | Risk if wrong |
|---|---|---|---|
| D1 | Action colour: orange vs keep teal | Orange action, teal identity | Retraining, brand pushback |
| D2 | Font: Vazirmatn + FaNum numerals vs single family | As §5.2 | Bundle size, digit rendering |
| D3 | Segment "reviewed" signal: backend vs local tracking | Ask backend; fall back to local | Misleading progress again |
| D4 | Legal replacement default | No default, explicit choice | Blocks flows backend expects |
| D5 | Gallery auto-save | Off | Inspectors lose local copies |
| D6 | Mileage plausibility behaviour | Amber confirm, not block | Typos pass or friction |
| D7 | Show prices/confidence to supervisor before publish | Show if product agrees | Legal/commercial |
| D8 | Persisted upload queue | Required for R2 | Photo loss continues |
| D9 | Re-enable error toasts (replace kill switch with the new feedback system) | Yes, R0 | Silent failures remain |

## 10. Constraints for implementation (Flutter / GetX)

- Implement tokens as a `ThemeExtension` (colour roles, text styles, spacing, radius) and **remove ad-hoc `copyWith` styling**; keep `uicat` as the component layer and re-skin through tokens.
- **Preserve Appium ids** (`Semantics` labels / `.id()`): `profileButton, backButton, callButton, completedDocumentTab, inspectionTab, finalApprovalTab, searchTextField, startInspectionN, continueInspectionN, QC Report {id}, car_info, car_details, car_documents, car_images_segment, car_milage, {slug}_segment, car_image_N, edit_image, delete_image, close_image, add_image_2, camera, gallery, video, videoGallery` [Code]. Keep or migrate with the QA owner.
- Keep: offline queue + pending banner (make it single and add retry), white status-bar fix, update gate, 401 logout, double-tap guards, start-inspection call order, Jalali display/Gregorian storage, digit normalisation.
- Navigation: stop using replace-routes (`Get.off`) where back should return to the previous context.
- Do not touch: status mapping layer, branch scoping (flag the hard-coded "شیخ بهایی" rule and the fuzzy matching to engineering).
- Accessibility: Semantics labels in Persian for all icon-only buttons; text scales to 130 % without clipping; colour never the only signal.

## 11. Releases (each shippable and measurable)

| Release | Ships | Exit check |
|---|---|---|
| **R0 Foundations + instrumentation** | Tokens/theme extension, core component set, state kit (skeleton/empty/error/offline), toast + feedback system (replaces kill switch), single app bar, events listed below | Events flowing; two screens (Login, Home list) on the new kit |
| **R1 Entry and lists** | Splash, Login, Home tabs/search/cards with progress ring, Profile, onsite sheet | Support tickets "can't log in" ↓ [Assumption]; list load time not worse |
| **R2 Inspection run** | Hub with stepper + readiness + blocker summary, Photos grid + upload tray + guided camera upgrades, Mileage, Car identity, Car details form | End-flow rejections ↓; photo upload abandonment ↓ (needs persisted queue D8) |
| **R3 Defects and AI QC** | Segment/defect screens, grade rails, diagram + trace upgrades, AI QC panel, finish confirm | Median duration not ↑; re-inspection not ↑; defect-photo errors ↓ |
| **R4 Supervisor** | Final approval, publish confirm, trim flow, cancel, dark-mode check | Publish reversals ↓; time-to-publish ↓ |

**Events to add in R0 (names are proposals):** `inspection_started/ended`, `step_viewed{step}`, `blocker_shown{type,count}`, `end_rejected{reason}`, `photo_captured/uploaded/failed/retried{slot}`, `ai_suggestion_shown/applied/edited{field,confidence}`, `qc_flag_shown/resolved{type,action}`, `publish_attempted/confirmed/reversed`, `error_shown{screen,type}`. Each tagged `ui_version`.

## 12. Acceptance checklist (design review)

- [ ] Readiness visible on the inspection screen at all times; every disabled primary says why.
- [ ] Purple only for AI output; orange only for actions; green only for done; red only for error/blocking.
- [ ] Empty, error, offline and loading are four different designs on every list/detail screen.
- [ ] Every tap target ≥48 dp (camera shutter 88); no text under 12 sp; instruction text ≥14 sp.
- [ ] Contrast ≥4.5:1 text, ≥3:1 UI, checked on a bright-outdoor variant for camera and diagram.
- [ ] Dangerous actions (publish, cancel, return, trim overwrite) have consequence + explicit confirm.
- [ ] AI components have working / result / low-confidence / unavailable states.
- [ ] RTL: chevrons, progress direction, LTR islands for plate, km, phone, timer.
- [ ] Every legacy colour/text style maps to a token.
- [ ] Appium ids list reviewed.

## 13. Weakest assumptions (challenge me)

1. **That a UI redesign moves inspection time.** The job is physical; screens are a fraction of the 20–30 minutes. Mitigation: make duration a guardrail and judge success on rework (end rejections, re-inspections) first.
2. **That orange can replace teal as the action colour** without friction for the team. No inspector research was done.
3. **That the backend can give a per-component "reviewed" signal and server-side acknowledgement audit.** If not, progress is local and weaker, and R9 stays open.

Also unverified: no screenshots or analytics were available; device mix and network conditions are assumed; the hub's supervisor «تایید» path appears not to call a finish endpoint (needs confirming).
