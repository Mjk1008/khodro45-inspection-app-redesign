# Notes B: inspection wizard, car detail/info, segments, defect grading, segment_defect

Source: `/Users/mj.khorshidsavar/Downloads/inspection-app-new-main/lib/screens/` (read-only). All paths below are relative to `lib/screens/`. `file:line` cites. **[Inference]** = my reading, not stated in code. Persian UI strings are quoted exactly.

Files read fully: everything in `car_detail/**`, `car_info/**`, `segments/**`, `car_segment/**`, `segment_defect/**`. `segments/widgets/car_svg.dart` (5,861 lines) skimmed: one `RPSCustomPainter` with 128 hard-coded `Path`s and hex colors; **never imported anywhere** (dead code; the live SVG screen is `car_svg/`, outside this slice).

Other dead code found: `segments/widgets/milage_input.dart` (`MilageInputWidget`, has a mileage-range + guarantee icon, `hintText: 'hintText'` literal), `segments/widgets/feature_segment.dart` (`FeatureSegment`), `car_segment/data/model/highlight_type_enum.dart`, `InfoTextRulesWidget` (`car_segment/widget/info_bottom_sheet.dart:77`), `CarDetail` class in `car_detail/data/model/inspection_model.dart:89+`. `AiPanelMode.standardImages/defect/documents` exist but are never instantiated; only `mileage` and `supervisorDocuments` are live (`segments/controller/segment_controller.dart:416,443`).

---

## 1. The wizard skeleton

### 1.1 It is a hub-and-spoke, not a linear wizard
- Entry: from the car list, if `car.status == InspectionStatus.ready` the app calls `PATCH /inspection/{slug}/start_inspection/` then pushes `SegmentScreen` (`cars/controller/tab_bar_controller.dart:283-289`, outside slice). If start fails it silently returns (no message).
- `SegmentScreen` (`segments/screens/segment_screen.dart`) is the hub: a vertical scroll of cards. Each card opens a separate full-screen page or bottom sheet; returning re-runs `getData()` (`segment_controller.dart:277,856,871`, `car_segment_list.dart:135`, `camera_segment.dart:86`).
- No step order is enforced. No "next" button between steps. The only forward action is the bottom bar button: `'پایان کارشناسی'` (inspector) or `'تایید'` (supervisor) (`segment_screen.dart:50-61`).
- There is no stepper, no overall progress bar, no "x of y done". [Inference] An inspector learns what is missing only when the end flow rejects them.

### 1.2 Cards on the hub, in render order (`segment_screen.dart:72-153`)
| # | Card | Widget | Opens | Captures | Visible to |
|---|---|---|---|---|---|
| 0 | Header row: `'شماره پذیرش : '` + appointment id, plus status chip | `CarIdAndStatus` (`segments/widgets/car_id_and_status.dart:13-21`) | nothing | none | all |
| 1 | Documents (server name, slug `inspection_vehicle_documents`) | `DocumentSegment` (`document_segment.dart`) | `CarImagesSegmentScreen(showCarDocuments: true)` (`segment_controller.dart:838-842`) | document photos (green paper, card, ID…) | hidden when `editableByInspector==true` |
| 2 | `'اطلاعات خودرو'` | `CarInfoSegment` (`car_info_segment.dart:31`) | `CarInfoPage` | brand/model/trim/year/option | always (isDisabled never passed) |
| 3 | `'جزئیات خودرو'` | `CarDetailsSegment` (`details_segment.dart:45`) | `CarDetailPage` | server-driven form: plate, color, fuel, insurance… | hidden when `editableByInspector==true` |
| 4 | `'تصاویر + ویدیوها'` + percent | `CarImagesSegmentWidget` (`camera_segment.dart:30-90`) | `CarImagesSegmentScreen(showCarDocuments: false)` | standard car photos/videos | hidden when `editableBySupervisor==true` |
| 5 | `'کارکرد'` (+ `' و {additional_information title}'`) | `MileageAndInfoWidget` (`mileage_widget.dart:41-68`) | bottom sheet | odometer km + odometer photo + free note | hidden when `editableBySupervisor==true` |
| 6 | 2-column grid of inspection segments | `CarSegmentList` (`car_segment_list.dart:37-139`) | `CarSegmentScreen` per segment | component defects, photos, grades | hidden when `editableBySupervisor==true` |
| 7 | `'لغو کارشناسی'` (red) | inline (`segment_screen.dart:115-152`) | `CancelInspection` page | reason + proof photos | only `editableBySupervisor==true` |

**Role logic is inverted by naming.** `isDisabled: carInfoModel.value.editableByInspector ?? true` hides the card (not disables) when the inspector *can* edit (`segment_screen.dart:82,88,95,98,112`). Net effect [Inference]:
- **Inspector** (`editable_by_inspector=true`): sees car info, images, mileage, segment grid. Ends with `'پایان کارشناسی'`.
- **Supervisor** (`editable_by_supervisor=true`): sees documents, car info, car details, cancel. Ends with `'تایید'` ("تکمیل مدارک" flow per comments, `segment_controller.dart:405-411`).
- If both flags are null, `?? true` hides almost everything.
- All cards use `Visibility(visible: !isDisabled)`, so a role never sees a greyed/disabled state; items just vanish.

### 1.3 Which segments exist
- Server-driven: `GET /inspection/{slug}/segment/` → `CarSegmentsModel` list (name, slug, order, icon, `segment_type`, `component_count`, `component_with_defect_count`, `rate`) (`segments/data/model/car_segments_model.dart:24-54`).
- Client filter: keep slugs starting with `inspection`, drop `inspection_documents`, `inspection_videos`, `inspection_positive-points` (`segment_controller.dart:94-103`). Then the grid drops `inspection_vehicle_documents` (`car_segment_list.dart:35-36`). The order is the server order (no client sort on `order`).
- Slugs referenced in code: `inspection_vehicle_documents`, `inspection_body_and_tire` (the only segment with severity/area grading and the SVG "quick defects" button, `car_segment_controller.dart:27`, `car_segment_screen.dart:191`), `inspection_electrical_system` (image category holding the odometer photo `instrument-cluster-km`, `segment_controller.dart:90-91`), `inspection_features`, `inspection_positive-points` vs `inspection_positive_points` (hyphen vs underscore mismatch: controller filters the hyphen form at `:101`, grid hides defect count for the underscore form at `car_segment_list.dart:111`). [Inference] one of the two checks is dead.
- Actual segment names (e.g. engine, body, interior) are not in code; they come from the API. Category names that the body segment uses are hard-mapped to icons: `'شاسی و پالونی','لاستیک','رکاب','محفظه صندوق','ناودانی','سقف','درب','گلگیر','سپر','شیشه و آینه','جلو پنجره و چراغ','رینگ'` (`car_segment/widget/category_accordion_card.dart:133-146`); comment says ~12 categories, "hundreds" of component rows (`:9,50`).

### 1.4 Progress: how it is computed and shown
- Per segment card (`car_segment_list.dart:65-118`):
  - Star tag with `rate` (server value; shown only if non-null). Colour: `rate <= 2` → `dismiss` (red), `< 4` → `warning`, else `success`.
  - Green check count = `componentCount - componentWithDefectCount` (`:106`, colour `0xff8DD169`). Note this is "components without defect", not "components reviewed". [Inference] a segment the inspector never opened shows all components as green, i.e. it looks complete.
  - Red X count = `componentWithDefectCount ?? 1` (`:113`, colour `0xffF3616C`); the `?? 1` fallback shows a fake defect when the field is missing.
  - `componentCount!` / `componentWithDefectCount!` force-unwraps (`:106`) crash if null.
- Images card: `templates_data.percent` printed as `'NN٪'` in primary colour, top-left; a transparent duplicate on the right for symmetry (`camera_segment.dart:49-76`). `total`/`completed` are parsed but unused (`car_info_new_model.dart:200-226`).
- Inside a segment: each category card shows an orange badge with recorded defects (`category_accordion_card.dart:81,111-122`); each component header appends `(n)` defects (`car_segment_screen.dart:381-383`).
- Timer: bottom bar shows `InspectionTimerWidget` counting from `inspection_start_at` in red 18px (`core/widget/custom_bottom_navigation.dart:34,72-77`, outside slice). Same bar on hub, car info, car detail, segment pages.
- No progress for mileage, car info, car details (no "done" tick on their cards).

### 1.5 Moving inside a segment (`car_segment/screens/car_segment_screen.dart`)
- Category filter chips + search box at top (`CategoryWidget`, `:171-184`; options `'همه'` + server `category_name`s, `car_segment_controller.dart:885-909`).
- Categories render as accordions, all collapsed by default (`car_segment_controller.dart:54-60`); forced open when searching or in incomplete-only mode (`car_segment_screen.dart:260-261`).
- Components inside: header row toggles expansion (`:366-403`), or for `is_option` components an `'آپشن'` pill + switch (`:405-451`).
- Bottom button: `'ثبت اطلاعات'` (l10n `end_of_image`) → `POST /inspection/{slug}/{segmentSlug}/validate_defects/` (`:87-134`).
  - 200/201 → pop back to hub.
  - 4xx → **pushReplacement** the same screen in incomplete-only mode with `highlightComponents` from the response and a title from `incompleteTitleFor()`: `'نقص‌های بدون محل'`, `'نقص‌های بدون تصویر'`, or `'نقص‌های بدون محل و بدون تصویر'` (`section_validation_model.dart:140-153`).
  - Network failure → snackbar `'بررسی اطلاعات ناموفق بود، دوباره تلاش کنید.'` (`:100`).
  - Double-tap guard `_submitInFlight` (`:52,73-79`).
- Back arrow leaves without validation (no warning). Defects were already saved server-side per tap, so nothing is lost; an invalid segment is only caught at end.

### 1.6 What blocks moving forward (all gates, in order)
**Segment level**: `validate_defects` (server) requires coordinates for `highlight==1` defects and media per `required_media` (`section_validation_model.dart:1-37` sample payload).

**Inspector end flow** (`segment_controller.dart:367-455`, then `:457-753`, then `:799-836`):
1. Re-fetch image templates; if any required non-document photo is missing → warning snackbar `'لطفاً قبل از پایان کارشناسی، عکس‌های اجباری بخش تصاویر را آپلود کنید.'` and push `CarImagesSegmentScreen(showIncompleteOnly: true)`; after return, re-check; stop if still missing (`:324-365,430`).
2. Push `AiCheckResultScreen(panelMode: mileage)`; it polls ai-lens and blocks ادامه until every *blocking* flagged photo is re-uploaded or explicitly skipped (see 1.9).
3. On continue → `getData()` then confirm sheet `'آیا از اتمام کارشناسی اطمینان دارید؟'` with checkbox `'اطلاعات خودرو را تایید میکنم.'`.
4. Tap `'تایید'`:
   - checkbox unchecked → `'تایید اطلاعات خودرو الزامی است.'` (`:667-676`)
   - inspector and `car_detail` lacks `mileage` → closes sheet, `'لطفا کارکرد کیلومتر را وارد کنید'` (`:677-689`). Bug: if `carDetail` is null, `null == false` is false, so the gate passes.
   - `body_color` required-for-inspection and empty → `'لطفاً رنگ بدنه را انتخاب کنید.'` (`:690-699`). Note the inspector cannot see the car-detail card where body color is set (hidden for inspector, 1.2), so [Inference] body color must come from AI/supervisor or this blocks them with no path to fix.
5. `PATCH /inspection/{slug}/end_inspection/` (4xx accepted, `segments/data/datasource/car_segments_datasource.dart:39-60`):
   - 200/201 → clear image URL cache, `Get.offAll(HomeScreen)`.
   - 4xx with `segments` or `images` → `SegmentDefectPage`.
   - 4xx otherwise → `'اتمام کارشناسی انجام نشد. لطفاً اطلاعات ناقص را تکمیل کنید.'`; Dio error → `ApiError.userMessage`; other → `'اتمام کارشناسی انجام نشد، دوباره تلاش کنید.'` (`:826-835`).
   - Double-tap guard `_endInspectionInFlight` (`:703-709,777`).

**Supervisor end flow** (`segment_controller.dart:368-426`): every car-detail field whose `required` includes `inspection` must be non-empty, else `'فرم جزئیات خودرو ناقص است'`; then `AiCheckResultScreen(panelMode: supervisorDocuments)`; its continue pops the panel and the segment screen. **No `end_inspection` call and no confirm sheet in the supervisor path.** [Inference] the backend completes the supervisor step elsewhere or the AI screen is the only "approval".

### 1.7 Draft / save / resume
- **No client draft.** Every action writes through immediately:
  - defect toggle on → `PATCH add_defect`; off → `DELETE remove_defect` (`car_segment_controller.dart:319-414`)
  - note: saved on focus loss (`defect_image_list.dart:90-96`) via `add_defect` with `note`
  - component description: saved on keyboard "done" only (`animated_item.dart:197-200`) → `PATCH component_data`. [Inference] typing then tapping elsewhere loses it, unlike notes.
  - option availability switch → `PATCH component_data {available}` (`car_segment_controller.dart:488-510`)
  - photos: uploaded on pick (through `PaintImageScreen` for defects)
  - grade: on `'تایید'` under the rails (`defect_grade_scale.dart:92-107`)
- Form pages (`CarInfoPage`, `CarDetailPage`) save only on `'ثبت اطلاعات'`. Leaving with changes opens a sheet `'ذخیره سازی اطلاعات'` / `'اطلاعات وارد شده ذخیره نخواهند شد،‌ آیا مطمئن هستید؟'` with `'ذخیره اطلاعات'` / `'خروج از فرم'` (`car_info_controller.dart:398-428`, `car_detail_controller.dart:202-228`).
- Mileage sheet: saved on `'تایید'`; `'انصراف'` or X discards (`mileage_widget.dart:325-379`).
- **Resume** = reopen the car from the list; all state reloads from `GET /inspection/{slug}/` + `/segment/` + components. The timer continues from server `inspection_start_at`.
- **Offline:** none in this slice. No queue, no "saved locally" indicator. ai-lens calls explicitly suppress the offline-retry interceptor (`inspection_check_datasource.dart:23-26`, `car_segments_datasource.dart:117`). Failed defect photo upload stores bytes and shows a red retry icon (`car_segment_controller.dart:795-800`, `defect_image_list.dart:115-127`). [Inference] in a garage with weak signal, toggles that fail return `false` silently (no snackbar in `addDefect`), so the switch can show "on" while the server has nothing.

### 1.8 Cancel inspection (supervisor only)
File: `segments/widgets/cancel_inspection.dart`.
- Opened by the red `'لغو کارشناسی'` card (`segment_screen.dart:115-152`).
- Copy: title `'لغو کارشناسی'`, body `'آیا از لغو این کارشناسی اطمینان دارید؟ این اقدام غیرقابل بازگشت است.'`, label `'علت لغو'`, dropdown placeholder `'انتخاب کنید'` (`:157-207`).
- Reasons: `GET /inspection/cancel_inspection_reasons/` → `[{id, localized_title}]` (`cancel_inspection_reasons_model.dart:23-25`). Shown in a bottom sheet list titled `'علت لغو'` (`:586-665`). Reason texts are not in code.
- **Hard-coded reason → required proof photos** (`:52-62`):
  - 119 → `green-paper`, `sales-invoice`
  - 116 → `green-paper`, `center-console` (4:3 size check)
  - 121 → `green-paper`, `boot-interior` (4:3 size check)
  - 40 → `green-paper`, `power-of-attorney`
  - other reasons → no photos.
  Titles/icons come from the image template list passed in. [Inference] these mean e.g. missing docs/console/boot issues; confirm with ops.
- Photo tiles: 2-column grid, dashed border (red `0xffff4869` when missing), numbered circle, edit pencil, retry icon; preview sheet allows delete (`:209-465`).
- Buttons: red `'لغو کارشناسی'` (disabled until a reason is chosen, disabled colour `0xffFDEBEA`) and `'انصراف'` (`:98-151`).
- Validation: no reason → `'انتخاب علت لغو اجباری است.'`; missing photos → `'بارگذاری تصاویر الزامی است.'` with red borders (`:667-693`). Upload failure → `'بارگذاری تصویر ناموفق بود، دوباره تلاش کنید.'` (`:573`).
- Submit: `PATCH /inspection/{slug}/cancel_inspection/ {reason_id}`; success → pop twice + `'کارشناسی با موفقیت لغو شد.'` (`:695-708`). **Failure shows nothing.** No second confirmation despite "irreversible" copy.
- Screen has no AppBar and no SafeArea (`:96-153`); the top row can sit under the status bar.
- Removing a photo here calls the generic image delete, which also deletes the car's real green-paper photo (same template slug). [Inference] risky side effect.

### 1.9 What is AI-assisted
| Where | What AI provides | Confidence shown | How the inspector confirms/overrides |
|---|---|---|---|
| Car info (brand/model/trim/year/option) | `ai_car_detail.{brand_id,model_id,trim_id,year_id,option_id}.{value,confidence}` from `GET /inspection/{slug}/` | `UICatTag` `'NN٪'` with AI45 icon when current value == AI value (`car_info_page.dart:157-170`) | Empty fields auto-filled (`car_info_controller.dart:90-120`); an AI chip under the field shows the AI title; tapping it applies (`car_info_page.dart:267-288`); the user can pick a different value from the list |
| Car details (all form fields incl. plate, body color, fuel, body type) | ai-lens `POST /doc-auto-fill` OCR of `car-card-rear`/`green-paper`/`car-card-front` (`car_detail/data/model/doc_auto_fill_model.dart:1-11`). Backend `ai_car_detail` deliberately ignored (`car_detail_controller.dart:59-63`). Polls 6×5 s if `pending_count>0` (`doc_auto_fill_repository.dart:28-65`) | Same tag next to section title (`car_detail_page.dart:211-237`) | Auto-fill empties (not the 3 legal-replacement radios, `:147-150`); AI chip below field to re-apply (`:244-277`); edit freely |
| End confirm sheet | `ai_car_detail.body_color` | tag next to color row (`segment_controller.dart:585-597`) | display only |
| AI tag colour | `confidence <= 0.8` red (dismiss), `<= 0.9` warning, else green (`core/utils/extension.dart:193-198`) | | |
| AI QC panel (`AiCheckResultScreen`) | ai-lens `POST /inspection-check {slug, role}` (inspector or supervisor); returns only *failed* images | per-check `confidence` parsed (`inspection_check_result_model.dart:155-168`) but **not shown** | Per flagged photo: `'آپلود مجدد'` (re-shoot in place), `'تایید است'` (keep as-is, with a confirm dialog `'این تصویر بدون اصلاح ثبت می‌شود.'`), or `'افزودن لوگوی پلاک'` (manual plate logo in paint editor) |
| Plate logo | `<url>_plate_allure` variant drawn by AI; fallback to original | n/a | manual placement button (`ai_check_result_screen.dart:1042-1074,1563-1599`) |
| Defect grading | `ai_metadata {defect_severity, defect_area}` is *collected for* AI (name), entered by inspector | n/a | see section 3 |

**AI QC panel detail** (`segments/screens/ai_check_result_screen.dart`):
- Check keys and labels (`ai_check_result_model.dart:254-259`): `slug_check` `'عدم تطابق تصویر'` (wrong angle/slot, blocking, red tag), `people_validation` `'وجود شخص در تصویر'` (advisory only, never blocks), `plate_validation` `'پلاک نمایان'` (blocking, warning tag), `analysis_failed` `'بررسی خودکار انجام نشد'` (advisory, except job `status: 3` which blocks, `:276`, screen `:197,503,599-604`).
- Vehicle QC checks (backend report flows only, unused in live modes): `'کنترل برگ سبز'`, `'کنترل گیربکس'`, `'کنترل کیلومتر'` (`ai_check_result_model.dart:50-62`), status labels `'نیاز به بررسی'` / `'نیاز به بررسی دستی'` / `'تایید شده'` (`ai_check_result_screen.dart:1420-1425`).
- Phases: analyzing → results | failed (`:36`).
  - Analyzing: AI45 logo, orange spinner `0xffF48220`, `'در حال بررسی تصاویر آپلود شده'`, a 10 s countdown then up to four 5 s extensions labelled `'نیاز به بررسی بیشتر'`, else `'کمی بیشتر طول می‌کشد...'` (`:267-318,1231-1277`). Polling: wait 5 s, up to 6 calls (`inspection_check_repository.dart:31-32,45-70`) → ~30 s worst case. Back is blocked: `'بررسی هوشمند در حال انجام است؛ لطفاً کمی صبر کنید.'` (`:1118-1126`).
  - Failed: `'بررسی تصاویر ناموفق بود'` (mileage) / `'بررسی مدارک خودرو ناموفق بود'` (supervisor) + `'ارتباط با سرویس بررسی هوشمند برقرار نشد.'`; buttons `'تلاش مجدد'`, `'ادامه بدون بررسی'` (`:1281-1325`). **AI outage never blocks submission.**
  - Results: section title `'تصاویر نیازمند بررسی'`; cards with 120×90 thumbnail (tap → zoomable full-screen), issue chips, person advisory box `'ممکن است شخصی در تصویر دیده شده باشد؛ در صورت نیاز تصویر را بررسی کنید. برای ادامه اقدامی لازم نیست.'`, plate hints `'لوگوی پلاک به صورت خودکار قرار می‌گیرد؛ اگر روی تصویر دیده نشد، دستی اضافه کنید.'` / `'هوش مصنوعی نتوانست لوگوی پلاک را قرار دهد؛ لطفاً به‌صورت دستی اضافه کنید.'` (`:1455-1707`). Pending blocking card has red border `0xffE57373`. Resolved tags: `'آپلود شد'` / `'تایید شد'` (`:1747-1753`).
  - Bottom bar: `'ادامه'` when all resolved, else `'بررسی همه تصاویر الزامی است'` with a no-op tap (looks enabled, does nothing) (`:1210-1215`).
  - After re-uploads, ادامه re-runs the whole analysis once, keeping skips (`:724-732`).
  - Exit with unresolved: sheet `'خروج از بررسی'` / `'همه تصاویر بررسی نشده‌اند. آیا بدون تکمیل بررسی خارج می‌شوید؟'` buttons `'ادامه بررسی'` / `'خروج'` (`:1128-1146`).
  - Clean report: mileage/supervisor modes never auto-continue; they show `'بررسی هوشمند انجام شد و موردی برای اصلاح یافت نشد.'` (`:676-677,1393-1417`).
- **Doc/code drift:** comments say the mileage panel "leads with mileage and note fields so a typo can be corrected here" (`segment_controller.dart:432-438`) and that mileage save "kicks off the ai-lens check" (`:113-116`, `mileage_widget.dart:21-22`). Neither exists in code: no mileage field in the panel; `saveMileageAndInfo` only PATCHes. In mileage mode `_visibleChecks` is always empty because the ai-lens report has no `auction_qc_result`. [Inference] the mileage-vs-photo OCR check was removed or moved server-side.

---

## 2. Screen by screen

### 2.1 SegmentScreen (hub) — `segments/screens/segment_screen.dart`, logic `segments/controller/segment_controller.dart`
- Purpose: home of one inspection; route to each section; end/approve/cancel.
- Layout: `CustomAppBar` with car name (brand/model/year) + help button + back (`:35-49`); body bg `0xfff6f6f6` (`:62`); `SingleChildScrollView` padding 24; cards are white, radius 12, shadow alpha 0.04, row of 56 px icon + 16 px title + chevron (`car_info_segment.dart:16-36`); segment grid 2 columns, spacing 16, aspect 0.8 (`car_segment_list.dart:39-46`); bottom `CustomBottomNavigatio` (timer + button).
- Load: 4 sequential awaits: `GET /inspection/{slug}/`, car-detail form JSON, `GET /segment/`, image templates (`segment_controller.dart:84-88`). [Inference] slow on bad networks; parallelisable.
- States: loading `Loading.large()`; success content; **empty and error both render a blank white container** (`segment_screen.dart:156`), no message, no retry.
- Testing ids (`.id(...)`): `car_info`, `car_details`, `car_documents`, `car_images_segment`, `car_milage`, `{slug}_segment` (useful for UI tests; preserve).

### 2.2 Mileage + note sheet — `segments/widgets/mileage_widget.dart`
- Card title `'کارکرد'` (+ `' و {title}'` when the car-detail form has an `additional_information` step) (`:59-62`, title source `segment_controller.dart:320-322`).
- Sheet: title `'ثبت کارکرد'` (+ `' و …'`), close X (blocked while submitting) (`:107-136`).
- Odometer photo box: 4:3 dashed frame, camera placeholder, small 16 px "add photo" button top-left on a dark chip, caption `'کیلومتر (کارکرد)'`; upload progress overlay (`:138-245`). Photo goes to template `instrument-cluster-km` (`segment_controller.dart:198-251`). Photo is **optional** here and not validated.
- Field `'کارکرد خودرو (کیلومتر)'`: number keyboard, LTR, digits only, thousands separators (`:254-286`). Required only by an early `return` when empty (no message) (`:336`). No min/max, no comparison with previous mileage or with `car_detail_meta.mileage_range` (parsed at `car_info_new_model.dart:91-107`, unused here).
- Field `'{title ?? 'توضیحات تکمیلی'} (اختیاری)'`: 3-line text, optional; empty clears saved note (`:288-321`, `segment_controller.dart:120-125`).
- Buttons `'تایید'` (loading) / `'انصراف'` (`:325-379`). On save: snackbar `'کارکرد … با موفقیت ثبت شد.'` or `'خطا در ثبت کارکرد …'` (`segment_controller.dart:128-147`); sheet stays open on failure.
- Two PATCHes to the same endpoint (`mileage`, then `additional_information`) (`car_segments_datasource.dart:75-95`).

### 2.3 End-inspection confirm sheet — `segment_controller.dart:457-753`
- Close icon + title `'آیا از اتمام کارشناسی اطمینان دارید؟'`; subtitle `'درصورت تایید، مجاز به ویرایش اطلاعات تازمان QC نخواهید بود.'`.
- Summary rows with dashed dividers: year | model | brand (LTR rich text, `:280-318`), trim, option (`'بدون آپشن'` if none), mileage `'{n} کیلومتر'`, body color dot + title or `'رنگ بدنه انتخاب نشده'` + AI tag, additional-info title + text or `'ندارد'`.
- Checkbox `'اطلاعات خودرو را تایید میکنم.'`; buttons `'تایید'` (fixed 174×40) / `'بازگشت'` (border `0xff847373`).
- Uses `Spacer()` inside a `mainAxisSize.min` Column in a scroll-controlled sheet (`:636`) [Inference] layout risk on small screens; width hack `Get.width - 68` (`:621`).
- Read-only: to fix a value the inspector must close, go to the right card, edit, and restart the end flow (AI check runs again).

### 2.4 AiCheckResultScreen — see 1.9. Layout: white AppBar with AI45 logo 120 px, back arrow 18 px (`:1181-1192`), list padding 16, cards radius 12. Buttons are `OutlinedButton` min height 36 (`:1616,1638,1649`), below the 44–48 px touch target guideline. Issue text 11 px (`:1575,1605,1739`).

### 2.5 CancelInspection — see 1.8.

### 2.6 CarInfoPage — `car_info/screen/car_info_page.dart`, `car_info/controller/car_info_controller.dart`
- Purpose: confirm/correct the car's catalogue identity, which sets `trimyear`.
- Header: `'اطلاعات خودرو'` + `'شماره پذیرش : '` + appointment (`:313-323`).
- Fields (all dropdown-as-bottom-sheet, shown only once their list is non-empty) (`:75-109`):
  | Field | Label | Source | Depends on |
  |---|---|---|---|
  | brand | `'برند'` | `GET /api/back_office/brands/available/` | – |
  | model | `'مدل'` | `…/models/available/?brand_id` | brand |
  | trim | `'تریم'` | `…/trims/available/?model_id` | model |
  | year | `'سال'` (sheet title `'سال را انتخاب کنید'`) | `…/years/available/?trim_id` | trim |
  | option | `'آپشن'` | `…/trimyears/available/?trim_id&year_id` (id = trimyear id, title = option title) | trim+year |
  Order is Brand → Model → Trim → Year → Option (trim before year). Choosing a parent clears children and refetches (`car_info_controller.dart:249-298`). Placeholder `'انتخاب...'`.
- All five required: card turns `redColor3`, badge `'این فیلد اجباریست!'` (`:131,292-308`).
- Save: `PATCH /inspection/{slug}/appointment/ {trimyear}` with `trimYearId ?? 0` (`car_info_controller.dart:379`); success → pop + `'تغییرات با موفقیت ثبت شد'`. **Failure: silent** (error toast suppressed, `car_info_segment_datasource.dart:70`, and no else branch).
- States: full-page spinner on initial load **and on every dependent fetch** (`:318-356` set `isLoading(true)`), so the whole form flashes away on each pick. No error state: an exception in `getCarInfo` leaves an empty page (try/finally only).
- No search in the brand/model sheets; long lists scroll inside a 50 % height sheet with nested `ListView` in `SingleChildScrollView` (`:203-226`).
- VIN, plate, chassis number, engine number are **not** here; plate is in car details; VIN not found in this slice.
- `isEmptyField` checks `selectedX.value == null`, but initial values are `DropdownItemModel(id: null)` objects, never null (`:61-80`), so [Inference] a missing server value can pass validation and save `trimyear: 0`.

### 2.7 CarDetailPage — `car_detail/screens/car_detail_page.dart`, `car_detail/controller/car_detail_controller.dart`
- Purpose: supervisor/document form (`جزئیات خودرو`): plate, color, fuel, body type, insurance and guarantee dates, legal replacements, etc.
- **Form is server-driven** from a static JSON per environment: `https://inspection.{env}.technical45.com/static/car_detail_k45.json` (`core/utils/constant.dart:4-12`), cached 1 h, no auth (`car_details_segment_data_source.dart:28-34`). Sections are kept only if `application_visible` contains `inspection` (`car_detail_model.dart:46-51`), sorted by `order` (`car_detail_controller.dart:230-232`).
- Section model: `step, order, is_guaranteed (shield icon), title, value_meta_data (meta text right side from car_detail_meta), description, hint, form[]` (`car_detail_model.dart:3-44`). `hint` and `description` are parsed but **never rendered**.
- Field model: `type (number|text|tel|radio|dropdown|date|plate), name, title, label, placeholder, initial_value, options[{title, slug(hex colour or id), value, hint, is_render_form}], render_form{title_form, form[]}, required[] (apps), separated (thousands), date_constraints[{when_value, min, max}]` (`car_detail_model.dart:109-180`). `grid_column`, `type_new`, option `hint`, `description_form` unused.
- Field types (`car_detail_page.dart:317-338`):
  - **text/number/tel** (`:743-859`): white filled input; number keyboard for number/tel; `separated` adds thousands separators; text is force-UPPERCASE; `engine_size` gets a decimal keyboard, pattern `d.d`, and auto-appends `.0` on blur (`:802-809,830-841,862-887`). **A new `TextEditingController` is created on every build** (`:744`) and `onChanged` writes `controller.text = value` (`:826`) [Inference] cursor jumps and lost keystrokes, worst with Persian IME.
  - **radio** (`:889-1044`): equal-width outlined buttons (primary colour, selected filled), 16 px vertical padding; selecting an `is_render_form` option reveals a sub-form titled in green `AppColors.green2`; switching away deletes sub-form values.
  - **dropdown** (`:1267-1466`): 46 px box opening a bottom sheet list; options whose `slug` contains `#` show a colour swatch (body color); `'سایر'` shows a special icon; same render-form mechanism.
  - **date** (`:1046-1147`): inline `CalendarWidget` (see 4). Stored Gregorian `YYYY-MM-DD`, shown Jalali.
  - **plate** (`:341-651`): Iranian plate image with 4 boxes LTR: 2 digits, letter (opens grid sheet `'حرف پلاک'` with 32 Persian letters + `V` (wheelchair icon), `D`, `S`), 3 digits, 2 digits + `'ایران'`. Auto-advance focus (`:476-486,561-566,612-618`). Stored as `"22-ب-333-44ایران"`, English letters lowercased. Validation in controller: required, ≥4 parts, letter in list (`car_detail_controller.dart:547-606`). Sizes from `Get.width` fractions (`0.86, 0.018, 0.199, 0.016`).
- **Legal-replacement radios injected client-side** if the JSON lacks them: `'تعویض اتاق قانونی'` (`cabin_replaced`), `'تعویض شاسی قانونی'` (`chassis_replaced`), `'تعویض موتور قانونی'` (`engine_replaced`), options `'دارد'`/`'ندارد'`, orders 2/3/4, required (`car_detail_controller.dart:236-312`). **Defaulted to `false` (ندارد) and never counted as empty** (`:318-331,349-360,617-620`). A comment says "Stay empty until the inspector picks" (`:147-148`), contradicted by hydrate. Debug prints log these three values on GET/PATCH (`car_details_segment_data_source.dart:39-50,59-68`).
- Validation on `'ثبت اطلاعات'` (`car_detail_controller.dart:368-423`): required-for-inspection fields (null, `''`, or `0` count as empty, incl. visible render-form children); plate format. Errors: section card red + `'این فیلد اجباریست!'` badge (`car_detail_page.dart:169-299`). No auto-scroll to first error. Server 400 → field names highlighted + snackbar listing `'{title}: {message}'` (`:400-410,456-475`). Success → pop + `'تغییرات با موفقیت ثبت شد'`.
- Save: `PATCH /inspection/{slug}/car_detail/` with the whole `carDetail` map.
- States: spinner while loading; **no error state** (`getCarDetailForm` has no try/catch, `:49-83`, so a failure leaves the spinner on forever); no empty state.
- Note `isEmptyField` treats `0` as empty (`:625`), so a legit `0` value (e.g. guarantee km) cannot be saved.

### 2.8 CarSegmentScreen — `car_segment/screens/car_segment_screen.dart`, logic `car_segment/controller/car_segment_controller.dart`
- Purpose: record defects per component in one segment.
- AppBar: car name + info icon → `'راهنمای کارشناسی'` sheet (legend: dot = point on schematic `'این ایتم ها در نمای شماتیک خودرو نقطه مشخصی میتوانند داشته باشند'`, shield = guaranteed `'این ایتم ها توسط خودرو۴۵ به لحاظ صحت کارشناسی توسط بازرس گارانتی خسارات خواهند شد'`; `'توضیحات'` shown on sales site when no defect; `'نکات'` go to the report) (`info_bottom_sheet.dart:7-75`).
- Top: category chips + search (94 px band) or, in incomplete mode, a 64 px white band with marker icon + incomplete title (`:136-169`).
- Body-and-tire only: full-width primary button `'عکاسی سریع نواقص ماشین'` → `CarSvgScreen` via `Get.off` (replaces this page) (`:198-228`, `car_segment_controller.dart:911-913`).
- Category accordion card: 40 px grey icon tile, 14 px title, orange count badge, chevron (`category_accordion_card.dart:62-122`).
- Component row (`:287-403`): shield icon (guaranteed vs not), name `(n)`, chevron; or option pill `'آپشن'` + `CustomSwitch` (56×32) (`custom_switch.dart:27-28`). Turning an option off while it has defects → dialog `'عیب‌ها را حذف کرده، سپس آیتم را غیرفعال کنید'` (`option_dialog.dart`).
- Expanded component (`animated_item.dart:172-208`): `'توضیحات'` text field with server `default_description` hint, **disabled once any defect is on** (`:191`); then defect list.
- Defect row: multi-defect = name + dot (red if `highlight==1` SVG coordinate required, navy `0xff24536C` if `highlight==3`) + hairline + edit-spot pencil (red when coordinate missing in incomplete mode) + `CustomSwitch` coloured by `severity_color` with icon by severity (15 → X, 10 → warning, 5 → info, else check) (`animated_item.dart:286-361`, `car_segment_controller.dart:849-860`). Single-defect components use `Radio` per defect (`:271-284`).
- Turning a defect on (`car_segment_controller.dart:319-380`):
  - `highlight==1` → opens `SpecifyingPointsScreen` (`'محل قرار گیری نقص'`) first; cancel reverts the switch; `replaced`/`cut-and-replace` skip drawing.
  - `highlight==3` → `add_defect` then opens the same screen.
  - otherwise → `add_defect`.
  - If media required → warning `'برای این نقص باید حداقل یک عکس یا ویدیو آپلود کنید'` / `'برای این نقص باید حداقل یک ویدیو آپلود کنید'` and the picker opens automatically (`animated_item.dart:113-133,347-349`).
- Defect body when on (`defect_image_list.dart`): `'نکات'` note field; 40 px dashed add-media bar (red when required and missing) or 3-column 4:3 grid with delete (confirm sheet `AcceptanceDialog`), video play icon, and trailing add tile; retry icon on failure; required text `'افزودن حداقل یک عکس یا ویدیو الزامی است.'` / `'افزودن ویدیو الزامی است.'`; grade rails (section 3).
- Media pick: camera / gallery / video camera / video gallery sheet (`chooseTheImage`), photos go through `PaintImageScreen` (annotate) titled `'{component} - {defect}'` coloured by severity before upload (`car_segment_controller.dart:612-735,862-883`).
- States: loading spinner; error **falls through to `SearchEmptyState`** (same as "no search results") (`car_segment_screen.dart:141`); no retry.

### 2.9 SegmentDefectPage — `segment_defect/segment_defect_page.dart`, `segment_defect_controller.dart`
- Shown when `end_inspection` returns 4xx with missing pieces.
- Title `'نقص‌های بدون محل'` (always, even for missing photos), close icon; text `'محل و تصویر نقص های مربوط به بخش‌های زیر را باید مشخص کنید.'` (`:59-84`).
- List: optional first row `'تصاویر + ویدیوها'` with red count of required missing images; then one row per segment with total missing defects count, segment icon, chevron (`:93-193`). Bottom `'بازگشت'`.
- Tap → `CarSegmentScreen(showIncompleteOnly)` or `CarImagesSegmentScreen(showIncompleteOnly)`; **on return it immediately re-calls `end_inspection`** (`segment_defect_controller.dart:21-98`). That bypasses the confirm sheet, the checkbox, and the AI QC re-check. Success → Home. Still failing → replaces itself with a fresh copy (no transition). Failing with no lists → nothing happens (silent).
- No AppBar/SafeArea; inline spinner replaces list while loading.

---

## 3. Defect grading, mileage, car info

### 3.1 Defect data model (`car_segment/data/model/component_model.dart`)
- Component: `name, slug, is_single_defect, is_important (unused in UI), order, is_option, is_guaranteed, default_description, description, available, category, category_name, defects[]` (`:41-88`).
- Defect (`ComponenetDefects`, `:117-223`): `id, severity (int: 5/10/15 seen), severity_text, severity_color (hex string), defect{name, slug, icon}, images[], note, has_defect, coordinate[double], highlight (1 = SVG coordinate required, 3 = optional spot), highlight_text, required_media ('1' photo or video, '2' video), required_media_text, ai_metadata`.
- Severity is server-assigned per defect type, shown only as switch colour and icon. `severity_text`, `highlight_text`, `required_media_text` are parsed but not displayed. [Inference] the inspector never sees the word for the severity.

### 3.2 Inspector grading (body & tire only)
- Applies when segment is `inspection_body_and_tire` (`car_segment_controller.dart:27,559`) and the defect "needs grading": FMP catalog `GET /pricing/fmp/defect-list` flags `label_severity`/`label_area` (`ai_metadata_policy.dart:4-11,85-91`); fallback blocklist excludes `'مات بودن'` (matte lamps/glass) and replacements `replaced`, `cut-and-replace`, `'تعویض'`, `'برش و تعویض'` (`:23-31,79-83`).
- Rails appear only after at least one photo exists (or a grade is already stored) (`ai_metadata_policy.dart:121-139`), on purpose: grade from the picture.
- Widget `DefectGradeScales` (`car_segment/widget/defect_grade_scale.dart:110-142`):
  - `'شدت آسیب'` (severity) 1–5, "heat" variant: each filled cell coloured on a green → orange → red ramp (`:171-176`).
  - `'مساحت آسیب'` (area) 1–5, "mass" variant: rising primary-colour band.
  - End labels `'کمترین میزان'` / `'بیشترین میزان'`, 11 px.
  - Cells 44 px high, five per row, haptic tick on tap.
  - `'تایید'` appears only when both are set; disabled until changed; saves `PATCH add_defect {component_slug, defect_slug, note, ai_metadata:{defect_severity, defect_area}}` (`car_segment_controller.dart:566-610`, `car_segment_datasource.dart:87-108`). Failure → `'ثبت شدت و مساحت آسیب ناموفق بود، دوباره تلاش کنید.'`.
  - Backend rejects partial or out-of-range values (`component_model.dart:247-278`).
- Area can be prefilled from the drawn region (`drawnAreaPercent`, local only) (`car_segment_controller.dart:456-481`, `component_model.dart:135-137`).
- **Not enforced by the client**: the inspector can submit a segment without grading. [Inference] the backend's `validate_defects` may or may not require it; check.
- There are **no per-component questions** in this slice beyond: description, per-defect note, option availability switch.

### 3.3 Photos per defect
- Required media per defect from `required_media` (`'1'` ≥1 photo or video, `'2'` ≥1 video) (`animated_item.dart:83-110`). No max in client. Grid of 3.
- Each photo is annotated in `PaintImageScreen` before upload (draw on image). Videos upload with progress (`car_segment_datasource.dart:39-71`, 60 s timeouts).
- Coordinates: drawn on the car SVG in `SpecifyingPointsScreen` (outside slice).

### 3.4 Severity colours
- Defect switch active colour = `severity_color` from server; icon by severity (`car_segment_controller.dart:849-860`).
- Segment rate tag: red ≤2, amber <4, green ≥4 (`car_segment_list.dart:75-82`).
- Grade heat ramp: `AppColors.greenColor → primaryColor_500 → redColor`.
- Dot before defect name: `Colors.red` (coordinate required) or `0xff24536C` (optional spot).

### 3.5 Mileage capture — see 2.2. Stored as `car_detail.mileage` (string digits). Odometer photo = image template `instrument-cluster-km`. Gate only at end confirm (inspector).

### 3.6 Car identity sources
- Brand/model/year/trim/option: `model_full_name` from `GET /inspection/{slug}/` (`car_info_new_model.dart:228-264`; IDs in `inspection_model.dart` `ModelFullName` incl. `trim_year_id`, `option_id`) + AI suggestions; editable in `CarInfoPage`.
- Plate: `car_detail.plate_number` field (type `plate`) in `CarDetailPage`, AI-filled from document OCR.
- Appointment id: `appointment` int, shown as `'شماره پذیرش : '`.
- VIN / chassis no. / engine no.: not referenced in this slice. [Inference] either a text field in the server JSON form or absent.

---

## 4. Calendar / appointment / scheduling in car_detail
- **There is no appointment scheduling in this slice.** `car_detail/widget/calendar/` is a date-of-record picker used for form fields of type `date` (e.g. insurance or guarantee expiry, inferred from `CarDetail` keys `insurance_expire_date`, `guarantee_expire_date`, `inspection_model.dart:105-106`).
- `CalendarWidget` (`calendar_widget.dart`) is always inline and expanded inside the form card:
  - Header row `'تقویم شمسی'` / `'تقویم میلادی'` with a toggle switch (`:401-419`).
  - Grey title strip: `'لطفا سال را انتخاب کنید'` or `'{day} {month} {year}'` (`:345-383`).
  - Tabs `'روز'` / `'ماه'` / `'سال'` (text links, green when active) (`:280-343`).
  - Year tab: wrap of 70 px chips within limits; month tab: 3×4 grid of Persian month names `'فروردین' … 'اسفند'`; day tab: 7-column grid with header rotated so column 1 = weekday of day 1 (no leading blanks) (`:105-278`).
  - Picking a day saves immediately via `onSelectedDate('Y-M-D')`; the page converts Jalali → Gregorian (`car_detail_page.dart:1233-1259`).
- Range: `date_constraints` from JSON: `min`/`max` as `today`, `today±N{d|m|y}`, or `YYYY-MM-DD`; constraint chosen by parent radio bool `when_value` (`car_detail_page.dart:1149-1221`). Default window: today − 11 years … today (`calendar_controller.dart:75-78`).
- Quirk: tapping the `'روز'` or `'ماه'` tab with no year picked jumps to **current year − 3** (`calendar_widget.dart:421-443`, comments "3 years ago"). Selected state uses green `AppColors.green2` borders only (no fill); disabled days `greyColor_15`.
- [Inference] each date field costs ~350–450 px of vertical space; several date fields make the form very long.

---

## 5. API endpoints touched by this slice
Base = `ApiEndpoints.baseUrl` (`…/api/v2`); "BO" = same host with `/api/back_office`; "AI" = ai-lens `AiLensClient` (`AiLensConfig.baseUrl`).

| Method | Path | Purpose | Key request → response | Cite |
|---|---|---|---|---|
| PATCH | `/inspection/{slug}/start_inspection/` | start (status ready) | – → 200/201 | `segments/data/datasource/car_segments_datasource.dart:29-36` |
| GET | `/inspection/{slug}/` | car info + state | → `appointment, slug, status, status_text, inspection_start_at, inspection_end_at, car_detail{}, ai_car_detail{key:{value,confidence,source}}, model_full_name{brand,model,trim,year,option,+ids}, templates_data{total,completed,percent}, car_detail_meta{mileage_range}, editable_by_inspector, editable_by_supervisor, logo, owner, car_number` | `:63-66`, `car_info_new_model.dart:42-88`, `car_detail/.../car_details_segment_data_source.dart:37-52` |
| GET | `/inspection/{slug}/segment/` | segment list | → `[{name, slug, order, icon{uuid,mime_type,url}, segment_type, segment_type_text, component_count, component_with_defect_count, rate}]` | `:69-72` |
| PATCH | `/inspection/{slug}/car_detail/` | mileage / note / whole car-detail form | `{mileage}` / `{additional_information}` / full map; 400 → field errors | `:75-95`, `car_details_segment_data_source.dart:55-89` |
| PATCH | `/inspection/{slug}/end_inspection/` | finish | → 200 or 4xx `{segments:[{slug,name,icon,components:[{slug,defects:[{slug,required_media,coordinate_required}]}]}], images:[{slug,segment_slug,image_required}]}` | `:39-60`, `section_validation_model.dart` |
| GET | `/inspection/cancel_inspection_reasons/` | reasons | → `[{id, localized_title}]` | `:98-101` |
| PATCH | `/inspection/{slug}/cancel_inspection/` | cancel | `{reason_id}` | `:104-111` |
| GET | BO `/newinspection/qc/{slug}/auction_qc_ai_report/` | backend AI QC report (only non-live panel modes) | → `{auction_qc_result{green_paper_check,gearbox_type,mileage_check:{status,confidence,description,human_review}}, image_error{inspection_image, image_defect}}` | `:114-133`, `ai_check_result_model.dart` |
| POST | AI `/inspection-check` | live AI QC (mileage + supervisor modes) | `{slug, role: inspector|supervisor}` → `{slug, inspection_id, pending_count, items{inspection_image[], doc_image[]:{slug,image_title,image_url,status(1..4),meta,checks:{slug_check|people_validation|plate_validation:{status,confidence,status_label}}}}}` | `inspection_check_datasource.dart:16-37`, `inspection_check_result_model.dart` |
| POST | AI `/doc-auto-fill` | OCR → form suggestions | `{slug}` → `{ai_car_detail, documents{slug:{status: ready|pending|no_ocr|failed|wrong_document|missing}}, pending_count}`; 8 s timeout | `doc_auto_fill_datasource.dart:13-38` |
| GET | `https://inspection.{env}.technical45.com/static/car_detail_k45.json` | car-detail form schema | → `[{step, order, is_guaranteed, title, value_meta_data, form[], application_visible}]` | `core/utils/constant.dart:4-12` |
| GET | BO `/brands/available/`, `/models/available/?brand_id`, `/trims/available/?model_id`, `/years/available/?trim_id`, `/trimyears/available/?trim_id&year_id` | catalogue cascade | → `[{id,title,…}]`; trimyears `{id, trim_id, year_id, option{id,title}}` | `car_info/data/datasource/car_info_segment_datasource.dart:22-63` |
| PATCH | `/inspection/{slug}/appointment/` | save trim-year | `{trimyear}` | `:66-73` |
| GET | `/inspection/{slug}/{segmentSlug}/components/` | components + defects | → see 3.1 | `car_segment/data/datasource/car_segment_datasource.dart:33-36` |
| PATCH | `/inspection/{slug}/add_defect/` | add defect / note / grade (JSON) and upload defect media (multipart `component_slug, defect_slug, file`) | → `{uploaded_file{uuid,url,mime_type}}` for media | `:39-71,87-108` |
| DELETE | `/inspection/{slug}/remove_defect/` | remove defect | `{component_slug, defect_slug}` | `:111-121` |
| PATCH | `/inspection/{slug}/component_data/` | option availability / description | `{available, component_slug, description?}` | `:74-84,124-136` |
| POST | `/inspection/{slug}/{segmentSlug}/validate_defects/` | segment submit | → 200 or 4xx `SectionValidationModel` | `:139-154` |
| GET | `/pricing/fmp/defect-list` (via `FmpDefectListRepository`) | which defects get graded | `label_severity`, `label_area` | `ai_metadata_policy.dart:4-6` |
| (car_images repo) | image templates list, image upload (`templateSlug`), image remove | odometer, cancel proofs, AI re-upload | | `segment_controller.dart:87,206`, `cancel_inspection.dart:430,542` |

---

## 6. UI defects (dated look, density, one-handed garage use)

**Hard-coded colours/sizes (no tokens)**
- Background `Color(0xfff6f6f6)` repeated (`segment_screen.dart:62`, `car_info_page.dart:37`, `car_detail_page.dart:118`, `car_segment_screen.dart:82`, `ai_check_result_screen.dart:1178`); `Color(0xffF6F6F6)` also in `mileage_widget.dart:198`.
- One-off hexes: `0xff24536C` (`car_id_and_status.dart:18`, `animated_item.dart:297`), `0xff8DD169`/`0xffF3616C` (`car_segment_list.dart:108,115`), `0xff847373` borders (`segment_controller.dart:741`, `segment_defect_page.dart:38`, AI screen `:1313,1626,1660`), `0xff8E9098` chevrons, `0xffff4869` errors, `0xffFDEBEA`, `0xff737373`, `0x8E1C2631` overlay chips, and a whole AI palette `0xffF48220, 0xffE57373, 0xffE59A00, 0xff8A6D00, 0xffFFF6E5, 0xffFFE0A3, 0xff00B86B, 0xffFDECEC, 0xffE0E0E0`. Switch colours as strings `'#00B86B'`, `'#ff06C78D'` (`car_segment_screen.dart:435`, `calendar_widget.dart:414`). Raw `Colors.red` (`animated_item.dart:297`, `defect_image_list.dart:147,404,437`).
- Font family string `'IranYekan'` inline (`cancel_inspection.dart:118`); font sizes 10–28 inline everywhere.
- Layout from `Get.width` math (`car_detail_page.dart:447-647`, `segment_controller.dart:621`, `cancel_inspection.dart:351`); fixed `Size(174, 40)` buttons (`segment_controller.dart:711,740`); fixed 160 px images card (`camera_segment.dart:33`).

**Inconsistent components**
- Four button styles for the same job: `PrimaryButton`, `ElevatedButton`, `CupertinoButton` (cancel, SVG shortcut), `OutlinedButton` with ad-hoc borders. Close buttons: `Icomoon.uniE916` image, `CupertinoIcons.clear` in `IconButton`, `Icons.close_rounded` 16 px, `IcomoonInspection.close` 12 px.
- Chevrons: `Icons.keyboard_arrow_left`, `Icomoon.arrowLeft2`, `Icons.arrow_forward_ios_rounded`, `keyboard_arrow_up/down_sharp` vs `_rounded`.
- Dropdown pattern duplicated three times (car info, car detail, cancel) with different titles/colours (`greyColor_950` vs `primaryColor`).
- Error highlight via a `Stack` + `Positioned.fill` background with a red badge pinned at `bottom: -9, right: 40` (`car_info_page.dart:124-310`, `car_detail_page.dart:169-299`): fragile, overlaps the next card.
- Required-field badge text `'این فیلد اجباریست!'` 10 px white on red.
- Switch is a custom 56×32 box with 7 px radius, not a platform switch (`custom_switch.dart`); on/off direction depends on RTL tricks.
- Title of `SegmentDefectPage` always says "without location" even when only photos are missing.

**Density and readability**
- Many 11–13 px texts for important instructions (AI advisories 11 px, grade end labels 11 px, required media 13 px).
- Calendar always open per date field; long scroll.
- Hub cards are 56 px icon + plain text; segment grid cards aspect 0.8 with small counts; no status per card (done / not started / has issues).
- Component description is disabled once a defect is on, without explanation.

**Discoverability**
- Odometer photo button is a 16 px icon in a corner (`mileage_widget.dart:211-243`); the big photo frame itself is not tappable.
- Mileage empty submit does nothing (no message) (`mileage_widget.dart:336`).
- `'بررسی همه تصاویر الزامی است'` button looks active but does nothing (`ai_check_result_screen.dart:1214`).
- Info about icons is hidden behind an AppBar info icon; legend not visible in context.
- "quick defects on SVG" button only in body segment and it *replaces* the current page (`Get.off`), so back goes to the hub, not the list.
- Roles hide cards instead of explaining why.
- Severity text, highlight text, required-media text from the API are never shown.

**One-handed / garage**
- Primary actions at bottom are good, but many targets are 36 px (AI card buttons), 24–28 px close icons, 16 px edit/delete chips on photos, calendar day cells in a 7-col grid with 8 px gaps, letter grid 50 px.
- Top-anchored snackbars (all `UICatSnackBarPosition.top`) appear far from the thumb and under the status bar; many last 5–6 s.
- Search box at top of segment page; category chips at top.
- Typing bugs in car detail text fields (controller recreated per build) hurt gloved/one-thumb typing.
- Whole-page spinner on every cascade pick in car info.
- No offline indication; failures of toggles/notes are silent.

**State handling gaps**
- Blank page on hub error/empty (`segment_screen.dart:156`); infinite spinner on car-detail load error (`car_detail_controller.dart:49-83`); empty page on car-info error; "no search results" on segment error; silent failures on car-info save, cancel, start inspection, add/remove defect, description save.

---

## 7. What a redesign must preserve, and what is risky

### Must preserve (business rules / safety checks)
1. **Server-driven structure**: segments, components, defects, severity, highlight, required media, car-detail form JSON, date constraints, cancel reasons. Do not hard-code lists in the new UI.
2. **Role split** via `editable_by_inspector` / `editable_by_supervisor` and the two different end flows (inspector `end_inspection`; supervisor AI documents check). Decide explicitly whether to keep hiding or show-as-locked.
3. **End gates, in order**: required non-document photos → AI QC (blocking: `slug_check`, `plate_validation`, failed job; advisory: person, empty verdict) → confirm sheet with explicit checkbox → mileage present → body color present → server `end_inspection` with `segments/images` remediation list. Copy `'درصورت تایید، مجاز به ویرایش اطلاعات تازمان QC نخواهید بود.'` carries a real consequence.
4. **AI outage must not block**: `'ادامه بدون بررسی'` path; bounded polling (≈30 s); fail-closed whitelist logic for slot ownership (`ai_check_result_screen.dart:421-436`).
5. **Skip needs a confirm** (`'این تصویر بدون اصلاح ثبت می‌شود.'`); re-uploads trigger one re-check; skips preserved across re-check.
6. **Coordinate-before-save** for `highlight==1` defects (avoids orphan defects that make `validate_defects` 400) (`car_segment_controller.dart:320-339`), with `replaced`/`cut-and-replace` exempt.
7. **Grading rules**: both severity and area 1–5 or nothing; grade only after a photo; body & tire only; catalog-driven with blocklist fallback.
8. **Plate format** `NN-L-NNN-NNایران`, allowed letters incl. `V/D/S`, lowercase storage.
9. **Legal-replacement fields** always sent as booleans (backend expects them).
10. **Option component rule**: cannot disable an option that still has defects.
11. **Cancel proof photos per reason** and required reason.
12. **Double-submit guards** on end inspection, segment submit, AI continue.
13. **Testing ids** (`.id('...')`, `Semantics(label: ...)`) used by automated tests: keep or migrate deliberately.
14. **Jalali display / Gregorian storage** for dates.
15. Thumbnails via media server resize suffix `/W/H/fit/` and cache-busting after re-upload.

### Risky areas
- **Legal replacements default to `'ندارد'`** (`car_detail_controller.dart:318-331`): an untouched field is submitted as "no cabin/chassis/engine replacement". This is a pricing/legal fact for the auction. A redesign that keeps the default hides a safety gap; one that removes it may block flows the backend expects. Needs a product decision. [Inference] high price-error risk.
- **Retry path skips confirmation**: `SegmentDefectPage` re-calls `end_inspection` automatically after each fix (`segment_defect_controller.dart:21-98`), bypassing the checkbox and AI re-check.
- **Mileage gate null bug** (`segment_controller.dart:677-678`) and no range or OCR cross-check, despite comments claiming an ai-lens mileage check. Mileage is a top price driver.
- **Body color gate for inspectors** who cannot open the car-detail card.
- **Hidden-vs-disabled role logic** with `?? true` defaults; null flags hide most of the hub.
- **"Green = OK" counts** on segment cards include never-reviewed components.
- **Cancel** removes the car's shared document photos and has no failure feedback; hard-coded reason IDs 40/116/119/121 will silently break if backend IDs change.
- **Car info** can save `trimyear: 0`; failures silent.
- **Text field controller recreation** in car detail; changing the widget tree may "fix" or newly break saved values because values are also written during `build` (`car_detail_page.dart:744-792`, `1271-1313`). Build-time side effects mean re-ordering or lazy-rendering sections changes what gets saved.
- **Widget cache** in segment screen (`_componentWidgetCache`, `car_segment_screen.dart:48,290-363`) keeps stale widgets for collapsed components.
- **Positive-points slug mismatch** (hyphen vs underscore).
- **Grading not client-enforced**; unclear if backend enforces.
- **Doc/code drift** in AI comments; do not design from comments.

### Open questions to confirm with the team
- Which segments the API returns today, and in what order; is `rate` computed server-side from defects?
- Does the supervisor `'تایید'` flow call any finish endpoint?
- Is `validate_defects` enforcing `ai_metadata` for body & tire?
- Should legal replacements require an explicit choice?
- Where is VIN captured, if at all?
- Real meaning of cancel reasons 40/116/119/121.
