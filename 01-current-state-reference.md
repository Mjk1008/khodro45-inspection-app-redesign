# Inspection App: Current-State Reference

**What this is:** the map of today's internal inspector app ("Copilot Inspection", Flutter + GetX), built by reading the full source on 2026-10-31. Source of truth for *what exists*. The redesign spec ([02](02-redesign-spec.md)) says what changes.
**Evidence:** every claim comes from the source code (file:line citations are in the three appendix notes). `[Inference]` = my reading of the code, not stated in it. Nothing here comes from production data, analytics or interviews. **No screenshots of the running app were available**, so visual statements come from the code (colours, sizes, widgets), not from seeing it.
**Appendix (full detail with file:line):**
- [A: shell, lists, profile, login, design system, network](appendix/code-notes-A-shell-lists-profile.md)
- [B: inspection hub, car info/detail forms, segments, defects, end flow, cancel, AI QC panel](appendix/code-notes-B-inspection-hub-forms-defects.md)
- [C: photos, camera, car diagram, defect points, annotation, final approval, publish](appendix/code-notes-C-photos-diagram-final-approval.md)

---

## 1. What the app is

- **Users:** branch/onsite **inspectors** (کارشناس) and **supervisors** (سرپرست). Role decides which tabs and cards exist. Onsite inspectors (branch type 2) get a different list (visits at the owner's address).
- **Job:** run the ~145-point inspection of a used car, capture photos/video/defects/mileage/documents, hand it to a supervisor who approves and **publishes it, which starts the live auction**.
- **Platform:** Flutter (Android first; iOS/web folders exist), GetX, portrait-locked, Persian RTL, light theme only, no page transitions, no dark mode.
- **Environments:** `prod` and `staging` flavors; staging shows a red debug strip across the top of every screen.
- **Auth:** username/personnel code + password, no OTP. Token in local storage. Any 401 logs out. 403 = "account disabled", call 1485 ext. 280.
- **Offline:** requests that fail on network are queued in memory (3 retries). Photos have **no** queue; a failed upload shows a small retry icon and the retry copy is lost when the screen closes.

## 2. The real flow (it is not a linear wizard)

```
Splash (update gate, 2.5 s fixed wait)
 └─ Login ─► Home
     tabs: [بارگذاری مدارک: supervisor] [کارشناسی: ready + in-progress] [تایید نهایی: supervisor]
       card tap
        ├─ کارشناسی ─► (PATCH start_inspection if ready) ─► INSPECTION HUB (SegmentScreen)
        │       hub cards, any order:
        │         ├─ مدارک (documents)            supervisor only
        │         ├─ اطلاعات خودرو (brand/model/trim/year/option)
        │         ├─ جزئیات خودرو (server-driven form: plate, colour, fuel, dates…)   supervisor only
        │         ├─ تصاویر + ویدیوها (photo slots, grid or «عکاسی سریع» guided camera)
        │         ├─ کارکرد (odometer km + photo + note)
        │         ├─ grid of server segments (engine, body & tire, interior…) ─► components ─► defects (+photo, +grade)
        │         │       body & tire only: car diagram ─► tap part ─► defect ─► trace area ─► photo ─► grade
        │         └─ لغو کارشناسی (cancel, supervisor only)
        │       bottom bar: «پایان کارشناسی» (inspector) / «تایید» (supervisor)
        │         inspector end gates, in order:
        │           required photos → AI QC panel (~30 s) → confirm sheet + checkbox → mileage → body colour → PATCH end_inspection
        │           4xx → "defects missing location/photo" list → fix → auto re-submit
        └─ تایید نهایی ─► FINAL APPROVAL (supervisor)
                photo issues + document mismatches (AI) → acknowledge each → trim Q&A must match
                «انتشار» = starts the auction in 35 s, prices from the FMP model, no confirmation
                «عدم انتشار» (reason required) · «برگشت به کارشناسی» (no confirm)
Profile: stats (today / month / re-inspection % / average time), branches, my inspections, update, logout
```

**Key structural finding:** the hub has **no stepper and no overall progress**. The inspector learns what is missing only when the end flow rejects them. A segment card's green count means "components without a defect", so an **unopened segment looks complete** [B §1.4].

## 3. Screen inventory (today)

| Area | Screens |
|---|---|
| Shell | Splash · Update-required · Login (+ forgot-password sheet, disabled-account sheet, dev server switcher) |
| Home | Home with 3 tabs · list card · status filter · search · empty/error · onsite visit-details sheet · support sheet · QC report webview |
| Profile | Profile · Branches · My inspections · Help · Update sheet · Logout sheet |
| Inspection | Hub · Car info · Car details (plate widget, inline calendars, radios, dropdowns) · Mileage sheet · Photos grid · Quick-shoot camera · Preview/replace sheet · Annotate editor · Segment (categories → components → defects) · Defect media + grade rails · Car diagram · Pick component / pick defect sheets · Trace-damage screen · Spot photo sheet · AI QC panel · Confirm-end sheet · Missing-defects list · Cancel |
| Supervisor | Final approval · Photo-issue card · Document-status card (+ inline edit/re-upload) · Trim Q&A sheet · Not-published dialog · (unused: pricing/auction screen) |

## 4. Rules a redesign cannot break (summary; full list in appendix B §7 and C §8)

1. **Server-driven structure:** segments, components, defects, severity, required media, car-detail form schema, cancel reasons. Never hard-code.
2. **Role split:** `editable_by_inspector` / `editable_by_supervisor`; two different end flows. Supervisor-only access is enforced in the client today [Inference: server must also enforce].
3. **Inspector end gates in order** (above). AI outage must **not** block ("ادامه بدون بررسی"); skipping a flagged photo needs a confirm.
4. **Defect rules:** photo first; coordinate required when `highlight==1` (replaced / cut-and-replace exempt); severity and area are both 1–5 or nothing, only after a photo, only for catalogue defects (labels for the pricing model).
5. **Plate format** `NN-L-NNN-NNایران`; Jalali display, Gregorian storage; Persian digits in UI, English digits on the wire.
6. **Status mapping layer:** inspection statuses 1/3/5/9 vs appointment statuses 3/4/5/23/62. The number 5 means different things. Keep the mapper.
7. **Start-inspection PATCH** runs before the hub opens (status ready only).
8. **Final approval gates:** acknowledge all photo issues and mismatched documents, trim must match, editing clears the acknowledgement, not-published needs a reason.
9. **Appium test ids** (`Semantics` labels / `.id()`) are used by automated tests [Inference]. Keep or migrate on purpose.
10. Offline queue, white status-bar fix, update gate, 401 logout, double-tap guards.

## 5. Defects and risks found (what the redesign has to answer)

### 5.1 UX and visual
| # | Finding | Evidence |
|---|---|---|
| U1 | Two parallel colour systems; "primaryColor_50…500" are actually orange; 3 reds, 2 greens, 2 blues; dozens of hard-coded hex values | A §3, B §6, C §7 |
| U2 | Text styles built with `copyWith` on one style; theme `bodyLarge` is 40 px italic; many 10–13 px texts for important info | A §3 |
| U3 | Three app-bar builds, four button styles, four close-button styles, four icon families | A §3, B §6 |
| U4 | Hub has no progress/stepper; "done" is misleading (see §2) | B §1.4 |
| U5 | Error, empty and "no results" look the same; several screens go blank or spin forever on error | A §2.4, B §6 |
| U6 | **All API/media validation toasts are switched off** (`errorToastsEnabled=false`): wrong password, bad photo ratio, size limits, video length fail silently | A §1 |
| U7 | Tap targets 16–36 px (photo edit chip, retry, info icon, diagram markers, acknowledge/save buttons); diagram parts 32 px | B §6, C §7 |
| U8 | Tracing the damage outline with a finger on a small panel; no zoom, no undo of last region | C §7 |
| U9 | Snackbars at the top of the screen, far from the thumb; no haptics; no pressed states on many rows | B §6 |
| U10 | Final approval: three stacked buttons (publish / not publish / return), the two secondary look identical though one is terminal; no "3 remaining" summary explaining why publish is disabled | C §3.3 |
| U11 | Wrong icons (visit date uses a user icon; app version uses a location icon); stale help text | A §3, C §2.1 |
| U12 | Debug strip covers the top of every staging screen; offline banner rendered twice | A §1 |

### 5.2 Product and safety risks (decide, do not just restyle)
| # | Risk | Layer that would enforce a fix |
|---|---|---|
| R1 | **Publish = live auction in one tap**, prices from the model, nobody sees them in the app, start time uses the phone clock, no confirmation | Backend (`change_status` validation) + UI confirm. UI alone is not a control |
| R2 | «برگشت به کارشناسی» has no confirm and no reason | UI confirm + backend reason field |
| R3 | **Legal replacement (cabin/chassis/engine) radios default to «ندارد»** and are never counted empty: an untouched field is submitted as "no replacement" | Product decision + client validation; backend should reject unset |
| R4 | Retry after `end_inspection` rejection auto re-submits and **skips the confirm checkbox and the AI re-check** | Client flow |
| R5 | Mileage gate passes when car-detail is null; no range check; comments claim an AI mileage check that does not exist in code | Client + backend range rule |
| R6 | Body colour is required to end, but the inspector cannot see the card where it is set | Client flow + role config |
| R7 | Submit-time **silent purge** of incomplete defects | Client must tell what was removed |
| R8 | Trim Q&A one tap overwrites the inspector's trim, answer result ignored, fragile `question_id` | UI confirm + backend contract |
| R9 | Acknowledgements live in memory only; no audit trail for a decision that starts an auction | Backend (System of Record) |
| R10 | Every photo, including owner documents, is auto-saved to the phone gallery | Product/privacy decision |
| R11 | No persisted upload queue; photos lost on weak signal or when the screen closes | Client + idempotent upload endpoints |
| R12 | Cancel deletes the car's real green-paper photo (shared slug) and shows nothing on failure; reason IDs 40/116/119/121 hard-coded | Client + backend |
| R13 | Badge polling every 10 s with N+1 calls, also while on deep screens | Client (not UI work, flag to engineering) |
| R14 | Hard-coded service tokens and Sentry DSN in source; token in plain storage; QC webview spoofs a 2012 user agent | Engineering/security (out of scope for UI, flagged) |

### 5.3 Dead code (do not design for it)
`segments/widgets/car_svg.dart` (5,861 lines), `PricingAuctionScreen`, `BrandModelYearDocumentEditor`, `AuctionQcApi`, `MilageInputWidget`, `FeatureSegment`, `lib/data/**`, `/inspection/count/`. The pricing screen is still the best sketch of what a price/confidence display for supervisors could look like (range, seller price check) [C §3.8].

## 6. Design tokens in use today

- Brand colour `#24536C` (teal-navy). Background `#F6F6F6`. Cards white, radius 8, shadow black 4 %.
- Status chips: ready cream/orange, in-progress blue `#1098F7`, inspected green `#62B237`.
- Fonts: IranYekan FaNum (app), Vazir (login only). Sizes 10–18 mostly.
- Components: `uicat` package (internal git) provides buttons, tags, text fields, dialogs, snackbars. Keep it as the implementation layer; re-skin through tokens.

## 7. Open questions only the team can answer

1. Which segments does the API return today, in what order, and is `rate` server-computed?
2. Does the supervisor «تایید» in the hub call any finish endpoint? (The code never calls `end_inspection` there.)
3. Does `validate_defects` enforce severity/area for body & tire?
4. Should legal-replacement fields require an explicit choice? (R3)
5. Is supervisor-only access to final approval enforced by the server?
6. What do cancel reasons 40 / 116 / 119 / 121 mean in business terms?
7. How many inspectors use onsite mode vs branch mode, and what devices (screen size, Android version)?
8. Where is VIN captured, if at all?
