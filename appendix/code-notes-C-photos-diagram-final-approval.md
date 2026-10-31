# Notes C: photos, camera, car diagram, defect points, paint editor, final approval, CHANGELOG

Repo: `/Users/mj.khorshidsavar/Downloads/inspection-app-new-main` (read-only). Paths below are relative to `lib/screens/` unless they start with `lib/` or `assets/`.
Odd folder name: `specifying_pointsـonـsvg` (U+0640 Arabic tatweel instead of `_`; imports use `%D9%80`). Called `spec/` below.
`[Inference]` marks anything I inferred rather than read in the code.

---

## 0. TL;DR

- **Photos**: the list of photo slots comes from the server (`GET /inspection/{slug}/imagetemplate/`). There are two ways to fill it: a 2-column grid with one card per slot, or a «عکاسی سریع» (quick shoot) mode. Quick shoot is a full-screen custom camera (camerawesome) that walks through every empty slot in order. It forces landscape, lays a white SVG outline of the target angle over the viewfinder, and records video for the `360` and `engine-video` slots (30 s countdown). Photos are WebP at quality 95 with **no downscaling**, and the cap is 20 MB. Upload is a direct awaited PATCH per photo with a progress %. A failed upload gets a retry icon. There is **no offline queue**, no background upload and no local draft that survives leaving the screen.
- **Car diagram**: a hand-coded `CustomPainter` (6,952 lines) shows a top-down car. 19 body panels have tap boxes from a JSON file, plus 6 extra regions in code (2 mirrors, 4 wheels). Tapping a part opens a chain: component chooser → defect chooser → a draw screen where the inspector traces the damage area on a single-panel SVG (that area auto-computes the «مساحت» (area) grade 1–5) → the photo sheet (photo is required) → severity/area rails. Marker colour comes from the server (`severity_color`). Replaced panels are tinted pink.
- **Final approval** («تایید نهایی», supervisor only) shows photo issues and document mismatches from AI/QC. The supervisor can edit mileage, gearbox or colour inline, or re-upload a document. A trim Q&A (questions that pin down trim and option) opens automatically when FMP has one. «انتشار» (publish) stays disabled until every issue is acknowledged and the trim matches. Publish **directly schedules the auction**: it reads `/fmp` base/step/target/price and POSTs `change_status scheduled_auction`, starting 35 s later and lasting the FMP `auction_time` (default 20 min). The other actions are «عدم انتشار» (do not publish; reason required, optional photo) and «برگشت به کارشناسی» (return to inspection).
- **The pricing screen (`PricingAuctionScreen`) is dead code.** Nothing navigates to it. It holds the FMP estimated price range, a seller price input and the smart-margin «منطقی/غیرمنطقی» check (is the seller's price reasonable). `BrandModelYearDocumentEditor`, `FinalApprovalActionsBar`, `FinalApprovalViewOnlyBanner` and `AuctionQcApi` are also unused.

---

## 1. Photo capture

### 1.1 Slot model and the required-shot list
- The server returns categories, each with `images[]`. Fields per slot: `title`, `slug`, `can_skip`, `inspection_order`, `icon{url}`, `image{uuid,url,mime_type}` (`car_images/data/model/car_images_segment_model_new.dart:79-115`).
- **Required** means `can_skip == false && image == null` (`:118`). `hasMissingRequiredImages()` can skip the `inspection_vehicle_documents` category (`:64-77`).
- The list is sorted by the first image's `inspection_order`, then within each category (`car_images_controller.dart:149-163`).
- The same screen serves two modes, set by `showCarDocuments`. True shows only the `inspection_vehicle_documents` category («مدارک خودرو», car documents). False shows everything else («تصاویر + ویدیوها», photos and videos) (`controller:129-133`, `screens:92`).
- Video slots belong to the category with slug `videos` (`controller:236`). Camera video mode is chosen by the slot slug `360` or `engine-video` (`camera_controller.dart:71`).
- **Local guide SVGs** (`assets/icons/svg_on_camera/`, map in `lib/core/utils/icons.dart:80-107`):

| slot slug | guide SVG |
|---|---|
| front, rear, left-side, right-side | front / rear / left-side / right-side |
| front-left-corner, front-right-corner, rear-left-corner, rear-right-corner | corner SVGs |
| front-left-tire, rear-left-tire, front-right-tire, rear-right-tire | one shared `tire.svg` |
| spare-tire | spare-tire |
| boot-interior, boot-instruments | boot-* |
| engine-performance, connections-inside-the-engine, engine-video | `engine.svg` |
| console, center-console, left-console | `console.svg` |
| inside-of-driver-door | `front-left-door.svg` |
| 360 | `view-360.svg` |

  - Unused bundled assets: `front-right-door`, `rear-left-door` and `rear-right-door` exist but nothing maps to them.
  - The comment at `icons.dart:77-79` says the backend `icon.url` guides return 404, so local assets are the real source. Unmapped slugs fall back to the network icon, then to a generic camera icon.
- **AI-checked standard slots** (`segments/data/model/ai_check_result_model.dart:302+`): front, rear, left/right-side, 4 corners, 4 tires, … These are the photos the AI check panel scores.
- **Document slots** that skip compression: `car-card-front`, `car-card-rear`, `green-paper`, `national-card-front` (`controller:240-245`, `final_approval_controller.dart:79-84`). On the documents screen the size check is also turned off (`checkImageSize: widget.showCarDocuments ? false : true`, `controller:350,376`).
- **Other slugs referenced** in final approval re-upload hints: `dashboard` (mileage), `console` (gearbox), `front` (colour) (`final_approval_controller.dart:1046-1059`).
- The full required list is **server-driven**, so it is not knowable from the client. [Inference] The guide map above is roughly the photo set: about 20 slots plus documents.

### 1.2 Car images screen (`car_images/screens/car_images_screens.dart`)
- **Opened** from the segment screen (outside my slice). It is also opened from final approval for a re-upload (`final_approval_controller.dart:1063-1091`), and from the AI-check retry with a `targetImageUuid`. In that last case it auto-opens that one slot and pops back with the uploaded `ImageCategoryModel`, or null (`controller:101-120`).
- **Props**: `carInfoModel`, `carName`, `showCarDocuments`, `showIncompleteOnly` (shows only required-missing slots, `controller:166-174`), `targetImageUuid`.
- **Layout**:
  - AppBar: `CustomAppBar(title:'', carName, showHelpButton:true)` (`:59`).
  - Background is hardcoded `0xfff6f6f6` (`:71`).
  - `CategoryWidget`: a chip filter with «همه» (all) plus each category name (`:85-93`, `controller:186-193`).
  - Documents mode only: a source toggle «دوربین» / «گالری» (camera / gallery), 44 px tall (`:101-131`). The choice is remembered for the visit and skips the chooser sheet next time (`controller:278-310`).
  - Photos mode only: list item 0 is a big «عکاسی سریع» (quick shoot) CupertinoButton in primaryColor, then a divider labelled «یا» (or) (`:177-249`).
  - Each category has a title at 18 pt and a **GridView with 2 columns, mainAxisExtent 180, spacing 24** (`:261-283`).
- **Card states** (`:288-525`):
  - Empty: a guide SVG at 100 px in `secondary500`, or a camera icon. Tap → `captureImages`.
  - Uploading: a local preview, a 35% black scrim and `UploadProgressWidget(progress)` (`:379-417`).
  - Uploaded image: a network thumbnail `${url}/300/400/fit/`. An **edit pencil chip** sits top-left: 8 px padding + 16 px icon ≈ 32 px, which is too small a target. Tapping it re-captures; tapping the card opens the preview sheet (`:477-499`).
  - Uploaded video: a placeholder asset with a play icon at 55 px.
  - Required-missing or upload error: the dotted border turns red `0xffff4869` and the index circle turns red (`:296-312,438`).
  - Title gets a ` *` suffix when `canSkip == false` (`:335`).
  - Retry: a 20 px `rotateLeft` icon with 4 px padding (≈ 28 px target) that re-calls `uploadImages` with the stored bytes or file (`:341-359`).
- **Bottom bar**: `CustomBottomNavigatio(title: «ثبت اطلاعات» (submit), isPrimaryEnabled: !hasMissingRequired, inspectionStartAt)` (`:60-70`). On tap → `checkAllImagesIsValid`. If required slots are missing it shows a snackbar «لطفاً قبل از ثبت اطلاعات، عکس‌های اجباری را آپلود کنید.» (please upload the required photos before submitting); otherwise it pops (`controller:608-634`).
  - The comment at `controller:625-627` matters: there is **no AI check on submit**. Flagged photos are reviewed once at the end («پایان کارشناسی» (end of inspection) for the inspector, «تایید» (approve) for the supervisor).
- **Chooser sheet** (`chooseTheImage`, `lib/core/utils/utils.dart:278+`): title «انتخاب عکس» or «انتخاب ویدیو» (choose photo / video), rows «گرفتن عکس» (take photo) and gallery. The video-only category offers video camera and video gallery only.
- **Quick shoot** (`:183-231`):
  - Builds a flat list of every slot sorted by `inspection_order`.
  - If all slots are filled: snackbar «همه تصاویر گرفته شده‌اند» (all photos are taken).
  - Otherwise `Get.to(CameraPage(forceLandscapeCamera:true, initialIndex:0, onImageCaptured: uploadImages(file)))`. Upload starts per capture **in the background of the grid screen**, without blocking the camera.

### 1.3 Camera (`camera/camera_page.dart`, `camera/camera_controller.dart`)
- Plugin: `camerawesome`, `CameraAwesomeBuilder.custom`. Back sensor, `FlashMode.none` at start, **4:3** aspect (`camera_page.dart:351-356`). Video is FHD with audio (`:103-105`). The physical shutter button is enabled (`:351`).
- **Preview box**: `Get.width × Get.width*4/3`, centred on a `0xff252B32` background (`:39,99-101`). [Inference] In landscape the preview does not fill the screen and the layout depends on width only.
- **Overlay** (`:120-186`, repeated three times for video, recording and photo modes; copy-paste):
  - The guide SVG is tinted white (`ColorFilter srcIn`). It rotates with native orientation through `RotatedBox` quarterTurns (landscape_left=3, landscape_right=1, portrait_up=0, else 2).
  - Bottom pill: black at 75% alpha showing `"{title} | {i+1}/{total}"`.
  - While recording, the pill turns into a red countdown `mm:ss` (`:239-262`).
- **Force landscape**: if the phone is held portrait, a 50% black scrim shows a rotate Lottie and «موبایل خود را بچرخانید.» (rotate your phone) (`:65-92`). It is only shown in quick shoot.
- **Controls** (`TakePhotoUI`, `camera_controller.dart:259-466`), all 56×56 `UICatButton` tinted on a dark theme and wrapped in `AwesomeOrientedWidget` so they rotate:
  - Top-left **flash toggle**: always ↔ none. There is **no auto mode**. The icon refresh uses a hacky 100 ms delay and `setState` (`:318-331`). The top-right slot is an empty 56 px spacer (a help button was commented out, `:336-350`).
  - Bottom-left **back** (`arrowRight2`): stops any recording, then `Get.back()`.
  - Centre **shutter**, 88 px white ring. Photo mode is white; video mode is red; recording shows a red rounded square.
  - Bottom-right «رد کردن» (skip): only when `canSkip == true`. It jumps to the next **empty** slot (`:436-458`).
- **Sequence**: on start it jumps to the first empty slot (`:32-33`). After a successful capture it goes to the next empty slot, and pops after the last (`:142-160`). Already-filled slots are skipped automatically, so retaking a filled slot is not possible from here.
- **Photo pipeline** (`:162-209`):
  1. Capture.
  2. `FlutterImageCompress` to WebP at **quality 95, no resize** (`:100-117`), saved to `appDocs/captured_images/{slug}_{ts}.webp`.
  3. If larger than 20 MB, the shot is rejected with the toast «حجم تصویر بیش از ۲۰ مگابایت است.» (image is over 20 MB) and stays on the slide (`app_media_picker.dart:19-29`).
  4. `GalleryService().saveFileToGallery` (not awaited). **Every inspection photo goes to the phone gallery.** [Inference] This is a privacy concern: plate and owner documents end up on the device.
  5. `onImageCaptured(slug, file)`, then next slide.
  - Failure is only `debugPrint`; there is **no user-facing message** (`:210-211`).
- **Video pipeline** (`:212-247`):
  - A 30 s countdown auto-stops recording (`:43-54`).
  - `checkVideoDuration` rejects anything over 31 s («مدت زمان ویدیو نباید بیشتر از ۳۰ ثانیه باشد.», must not exceed 30 s) or under 10 s («… کمتر از ۱۰ ثانیه باشد.», must not be under 10 s) (`app_media_picker.dart:350-366`).
  - The file is copied to `captured_videos/{slug}_{ts}.mp4` and saved to the gallery. Video size cap is 100 MB («حجم ویدیو بیش از ۱۰۰ مگابایت است.», `app_media_picker.dart:344`).
  - The gallery picker path uses `pickVideo(maxDuration: 30s)` (`:220`).
- Grid-mode camera and gallery picks go through `AppMediaPicker.pickImageFromCamera/Gallery` (system picker, not camerawesome). They use the same compression (`app_media_picker.dart:43-58`, WebP q95) and an optional size or aspect check.

### 1.4 Upload, queue, retry, offline
- `PATCH /inspection/{slug}/add_image/` multipart with `image` (stream from file, or bytes named `image.jpg` even though they are WebP) and `template_slug`. Send and receive timeouts are 60 s. `onSendProgress` drives the %, and a Sentry tracker records progress (`car_images/data/datasource/car_images_date_source.dart:35-88`). The response is `uploaded_image{uuid,mime_type,url}` (`upload_image_model.dart`).
- On error: `uploadError=true` and the bytes are kept in memory (`lastImageBytes`/`lastImageFile`). Snackbar «بارگذاری تصویر ناموفق بود، دوباره تلاش کنید.» (photo upload failed, try again) or the video variant, shown at the top for 5 s (`controller:466-486`). The user taps the retry icon.
- **No persistent queue, no automatic retry, no connectivity awareness.** Retry bytes live in RAM on the model and are lost when the screen closes. Quick-shoot captures are also saved as files under appDocs, but nothing re-uploads them later. [Inference] Weak connectivity at a branch, or onsite, will cause lost or duplicated work.
- Delete: `DELETE /inspection/{slug}/remove_image/` with body `{file_uuid}` (`:91-98`). It is triggered from the preview sheet behind an `AcceptanceDialog` confirm (`lib/core/widget/bottom_sheet_view.dart:633-695`).
- On success the uploaded URL is seeded into `InspectionImageUrlCache` for the AI panel (`controller:460-465`).

### 1.5 Preview, gallery and edit (`PreviewBottomSheet`, `lib/core/widget/bottom_sheet_view.dart`, outside my slice, skimmed)
- `showImageSegmentBottomSheetView` is non-dismissible with drag disabled (`car_images/widget/bottom_sheet_image_view.dart:31-53`).
- It is a slider across the category's images with an index badge, an add-photo action on empty items, and video playback with a scrubber.
- Top row:
  - «شخصی سازی» (customize) opens the **PaintImageScreen** annotation editor. Only for images, not on the `doc` page type (`:565-620`).
  - `add_photo_alternate` replaces the photo.
  - Delete (confirmed) and close.
- [Inference] The word «شخصی سازی» does not tell the user this is "annotate photo".

### 1.6 Paint and annotation (`paint/`)
- `PaintImageScreen` uses `pro_image_editor` (`paint/screen/paint_screen.dart`). Blur, filter, tune, crop/rotate and emoji are all disabled (`:25-43`).
- Main editor bottom bar: paint, text, sticker. The only **sticker is the Khodro45 plate sticker** (`AppImages.plateSticker`, id `sticker-khodro45`, `:38-41`). [Inference] It is used to cover licence plates.
- Paint sub-editor: freehand, arrow, line, rectangle, circle. There is no eraser, opacity or fill (`:45-56,108-163`). Initial stroke is 4 px. Initial colour is `pageNameColor`, which is the defect's severity colour, or orange `0xFFff783f` (`:54-55`).
- AppBar: page name split on ` - ` into a component line (black) and a defect line in severity colour (`:271-289`), plus redo, undo and close.
- Buttons: «ذخیره» (save) in the main editor; «تایید» (confirm) in the paint editor. If nothing was edited and there were no bytes, save just pops (`:357-365`).
- Exit warning: «خروج» (exit) / «آیا از خروج از ویرایشگر مطمئن هستید؟ تغییرات شما ذخیره نخواهد شد.» (leave the editor? changes will not be saved) with «انصراف» (cancel) / «تایید» (confirm) (`:201-250`).
- Output max 2000×2000 (`:197`). Done saves to the gallery, except for an unedited defect photo (`:386-391`).
  - Defect page type → `uploadDefectImages` → `carComponentRepository.uploadDefectImage` (PATCH `/inspection/{slug}/add_defect/` with the file. The endpoint lives in car_segment, outside my slice).
  - Otherwise → `PATCH add_image` with `template_slug = defectslug` (`paint_controller.dart:20-101`).
  - Pops the `ImageModel` via `Navigator.of(rootNavigator:true)`. The code comments explain why `Get.back` loses the result above bottom sheets (`:46-49`).
  - Error: «بارگذاری تصویر ناموفق بود، لطفا مجددا تلاش کنید.» (upload failed, please try again).
- **Every defect photo is forced through the paint screen.** In `show_image_controller.dart:84-105`, pick → `paintImage` → upload. Cancelling the editor means nothing is uploaded.

### 1.7 AI recognition on images (what my slice touches)
- There is no on-device ML. AI happens server-side via the "ai-lens" gateway (`AiLensClient`).
- Final approval reads `POST /inspection-check {slug, role:'supervisor'}`. Each image comes back with per-check verdicts:
  - `slug_check` «عدم تطابق تصویر» (photo does not match the slot)
  - `people_validation` «وجود شخص در تصویر» (person in photo)
  - `plate_validation` «پلاک نمایان» (plate visible)
  - `analysis_failed` «بررسی خودکار انجام نشد» (automatic check did not run)

  Sources: `segments/.../ai_check_result_model.dart:254-297`, `final_approval/data/model/inspection_check_model.dart`.
- `POST /inspection-qc {inspection_id}` returns document checks: `green_paper_check`, `gearbox_check`, `mileage_check`, `color_check`, `brand_model_year_check`, each with `expected_value` vs `ai_value` (`inspection_qc_model.dart`).
- Defect severity and area ("ai_metadata") are **labels collected for AI training and pricing**, not predictions. See §2.4.
- CHANGELOG mentions an "AI Lens Integration" for mileage and gearbox, plus an "AI Check Result Screen". These are in other slices.

---

## 2. Car body diagram (`car_svg/`) and defect points (`spec/`)

### 2.1 CarSvgScreen («نواقص ماشین», car defects) (`car_svg/screen/car_svg_screen.dart`)
- Opened from the car segment flow (component list), with `CarSegmentPrerequisite` (car info + segment). Back, both system and appbar, does `Get.off(CarSegmentScreen)`: it **replaces** the route rather than popping (`:31-41`).
- AppBar: an info icon (24 px, no padding, so a small target) opens `WholeSvgInfoWidget` «راهنمای صفحه» (page guide) with 4 bullets (`lib/l10n/app_fa.arb:171-175`):
  1. Points come from components marked orange in بدنه و لاستیک (body and tyres).
  2. To edit an image, location or note, go back to that component.
  3. Tap a point to see its info and photos.
  4. Colours, from low to high importance: grey → orange → red.
  - [Inference] Bullet 2 is stale. The diagram now records defects directly, so the help text is out of date.
- **Diagram**: `CustomPaint(size: (Get.width-32) × ×1.416, painter: CarSvg(defects))` (`:77-83`, `controller:40-41`).
- Hit layers, bottom to top:
  - `PartHitBox` for each JSON part that has defect entries.
  - `PartHitBox` for each active extra region.
  - **Defect markers**: circles filled with `severity_color` holding the defect's network SVG icon at 20 px, ≈24 px target. Tap → photo sheet (`:114-135`).
- `PartHitBox` pads thin parts to a **32 px minimum** and flashes a 10% primary highlight on press (`car_svg/widget/part_hit_box.dart:20,41-64`). [Inference] 32 px is under the 44–48 px guideline, and the thin pillars sit next to doors, so mis-taps are likely, especially with gloves.
- Below the diagram: `CarComponentList` «لیست اجزا» (component list) with the hint «اجزایی که روی نقشه نیستند را از این لیست باز کنید و نواقص آن را ثبت کنید» (open parts that are not on the map from this list and record their defects). It groups unreachable components by `categotryName` (sic) into `CategoryAccordionCard`s, collapsed by default (`widget/car_component_list.dart`).
  - Tapping a row does `Get.off(CarSegmentScreen)`: it leaves the diagram for the segment list and **loses context**. The comment admits the inspector has to pick the row again (`controller:301-308`).
- FAB legend (`widget/float_button.dart`) with a white chip and 8 px dots:
  - «قابل انتخاب» (selectable): white with border
  - «غیرقابل انتخاب» (not selectable): `0xff7a93af`
  - «شیشه» (glass): `0xffd0ddec`
- Bottom bar: «ثبت اطلاعات» (submit) → `submitSegment` (`:51-57`).

### 2.2 Parts list and geometry
- `assets/jsons/car_svg_part_pixel.json` has 19 rects in fractions of the diagram: ceiling, engine-door, trunk-door, front-bumper, rear-bumper, front/rear × left/right door (4), front/rear × left/right fender (4), front/middle/rear × left/right column (6).
- Extra regions in code (`car_svg/data/car_svg_extra_parts.dart:63-116`):
  - «آینه راننده» (driver mirror) `left-mirror`; «آینه شاگرد» (passenger mirror) `rights-mirror`. The comment says the backend really spells it that way: "do not fix".
  - 4 wheels «چرخ جلو/عقب راننده/شاگرد» (front/rear, driver/passenger), slugs `driver's-front-tire-tread` etc. They also match by Persian component name, e.g. «آج لاستیک جلو راننده» (front driver tyre tread) or «لاستیک جلو راننده» (front driver tyre).
  - Sides: راننده (driver) = left, شاگرد (passenger) = right.
- Not on the diagram (reached through the list): spare tire, boot interior and instruments, roof items, glass, headlights, chassis and rocker («رکاب») (`car_svg_extra_parts.dart:10-23`, `car_component_list.dart:7-9`).
- Painter colours (`car_svg/widget/car_svg.dart`): stroke `0xff30445B`, panel fill `0xff7B93AF`, glass `0xffD0DDEC`, orange accents `0xffFF793F`/`0xffFFB321` gradient. Every body panel fill calls `checkIsReplaced(slug)` → white, or **pink `0xfffebec5`** (`AppColors.replacedDefectColor`, `lib/core/utils/color.dart:91`) when a `replaced` or `cut-and-replace` defect exists (`car_svg.dart:6937-6951`). `shouldRepaint` always returns true.
- Per-panel SVGs for the drawing screen live in `assets/icons/car_parts/*.svg` plus `*_red.svg` variants (19 parts × 2). Left parts rotate 90°, right parts -90° (`spec/controller/specifying_controller.dart:266-328`).

### 2.3 Tap → record flow (`car_svg/controller/car_svg_controller.dart`)
1. `onTapPart` or `onTapComponents` (`:311-422`). If the region maps to more than one component (a wheel is tyre + rim), `SelectComponentSheet` asks «جزء مورد نظر را انتخاب کنید» (choose the part). Rows without defects are disabled at 50% opacity.
2. If the component has more than one defect type, `SelectDefectSheet` asks «برای افزودن تصویر، نوع نقص را انتخاب کنید» (choose the defect type to add a photo). Each row has a dot in `severity_color` and a count of photos already attached. Defect types come from the server per component (`defect{name,slug,icon}`, `severity`, `severity_color`, `highlight`, `required_media`, `ai_metadata`). Shared rows are in `widget/select_defect/select_sheet_shared.dart` (rows 14 px vertical padding ≈ 48 px tall, which is fine).
3. **Needs a point** if `highlight == 1` (SVG) and no coordinate yet (`:426-428`). This mirrors backend `validate_non_empty_coordinate_inspection_defect`. → push `SpecifyingPointsScreen`. If it returns anything other than true, the draft is discarded and removed on the server if it had been recorded (`:358-377,462-488`).
4. **Photo first, deferred registration**: `requirePhoto = true` always on the SVG flow (`:350`). A new spot does not call `add_defect` until the photo uploads (`:353`). The coordinate is held in `pendingCoordinate` and saved after the media lands (`commitDeferredRegistration`, `:537-571`).
5. The photo sheet `ShowSpotImageWidget` opens in a bottom sheet capped at 90% height, dismissible (`:573-605`).
   - On close without the required media, the defect is wiped locally and on the server (`removeDefect`) with a warning: «نقص بدون عکس ثبت نشد.» (defect not saved without a photo) or «نقص بدون ویدیو ثبت نشد.» (… without a video) (`:607-644,455-460`).
6. **Replaced** (`replaced`, `cut-and-replace`) skips drawing: the draw screen shows the red panel and confirm pops true right away (`:430-434`, `spec controller:43-45,341-344`).
- **Submit** (`:45-108`):
  1. Purge SVG defects that have no coordinate.
  2. Purge defects missing required media. If any were removed, warn «نقص‌هایی که عکس/ویدیو اجباری نداشتند حذف شدند. لطفاً دوباره با عکس ثبت کنید.» (defects missing required photo/video were removed; please record them again with a photo) and **stop**.
  3. `validateDefects(slug, segmentSlug)` (car_segment repo). Valid → pop. Invalid → `pushReplacement(CarSegmentScreen(showIncompleteOnly, highlightComponents))`. Network error → «بررسی اطلاعات ناموفق بود، دوباره تلاش کنید.» (checking the data failed, try again).
  - [Inference] The silent server-side deletion of defects at submit is risky. The inspector may not notice which ones went.
- Media rules (`:438-453`): `required_media` '1' = photo, '2' = video. SVG spots always need at least one photo. Mode 2 needs at least one item whose type contains `video`. But the defect sheet's video picker is commented out (`show_image_controller.dart:136-155`). [Inference] A required-video defect probably cannot be satisfied from this sheet.

### 2.4 Spot photo sheet (`car_svg/widget/show_spot_image/`)
- Header: component name in primary colour, a 44 px «افزودن تصویر» (add photo) button with a camera icon, and a 44×44 close button (`show_image.dart:71-121`). These are good targets.
- Body:
  - Empty: a gallery icon at 80 px.
  - Carousel (viewport 1, no loop) with 40 px prev/next controls and a counter `i/n`.
  - A red dot plus the defect name.
- **Grade rails** `DefectGradeScales` («شدت آسیب» (damage severity) and «مساحت آسیب» (damage area), 1–5, labels «کمترین میزان» / «بیشترین میزان» (lowest / highest), button «تایید») are shown only when all of these hold (`ai_metadata_policy.dart:121-140`, `show_image.dart:247-265`):
  - segment is `kBodyAndTireSegmentSlug`, and
  - there is media or a stored grade, and
  - the FMP catalog flags the defect with `label_severity` or `label_area` (`fmp_defect_list_model.dart: needsGrading`), and
  - the defect is not a replacement (تعویض, برش و تعویض).
- Saving re-calls `addDefect(..., note, aiMetadata)`. Valid range is 1..5, and the backend returns 400 otherwise (`component_model.dart:247-277`). Success closes the sheet. Error: «ثبت شدت و مساحت آسیب ناموفق بود، دوباره تلاش کنید.» (saving severity and area failed, try again) (`show_image_controller.dart:263-303`).
- The area rail is **prefilled from the drawn region** (`aiMetadata.defectArea = grade`, `spec controller:330-338`, `car_svg_controller:386-390`).
- While capturing, a full overlay with a spinner blocks the sheet (`show_image.dart:271-277`).
- Photos: chooser → camera or gallery → **paint editor (mandatory)** → upload → append to `showImageList` → `_showLatestImage` → `onMediaCommitted` → refresh. The sheet stays open so the grading can happen (`show_image_controller.dart:73-164`).

### 2.5 SpecifyingPointsScreen: draw the defect region (`spec/screen/specifying_point_screen.dart`)
- Grey `0xff84888b` background. White header with back, title «محل قرار گیری نقص مورد نظر را مشخص کنید» (mark where the defect is), component name, a «پاک کردن» (clear) refresh icon, and a live line: «محدوده‌های نقص را روی قطعه بکشید» (draw the defect areas on the part) → «مساحت آسیب: {g} از ۵ · {n} محدوده» (damage area: g of 5 · n areas) (`:113-177`).
- First-run coach: a 32% black overlay with the same instruction text and a Lottie tap hand at random positions. Tap to dismiss (`:286-361`). [Inference] The hint is about *drawing* but the animation shows a *tap*, which contradicts it.
- **Drawing**: freehand pan strokes in percent-of-SVG coordinates (`spec/controller:143-202`). Samples are de-duplicated when less than 1.5% apart. A stroke closes into a polygon. Several regions are allowed.
  - Region is valid if its area is ≥ 0.2% of the part. Otherwise «محدوده را کامل بکشید» (draw the whole area).
  - Fill is severity colour at 35% plus a 2.5 px stroke. A centroid marker carries the defect icon (`spec/widget/defect_region_painter.dart`).
- **Area grade** (`spec/domain/defect_area_grade.dart`): shoelace area ÷ the panel's fill ratio (a table of 19 slugs, e.g. column 0.2039, door 0.87), summed and clamped. Grade bins: ≤20%→1, ≤40%→2, ≤60%→3, ≤80%→4, otherwise 5. **Only one coordinate is sent** to the API: the area-weighted centroid (`spec controller:72-77`). The polygons themselves are **not persisted**; only grade, % and the centroid are.
- Previously recorded spots for other defects on the same component are drawn from `coordinate` (`:257-285`).
- Confirm «تایید» (bottom bar 70 px, `PrimaryButton`; `spec/widget/select_kind_of_damage.dart`) runs `saveSpecificPoint` (`spec controller:340-422`):
  - Deferred: set `defect.coordinate`, pop true.
  - Otherwise: `PATCH /inspection/{slug}/add_defect/ {component_slug, defect_slug, coordinate:[x,y] as 2-dp strings}`, then success toast «تغییرات با موفقیت اعمال شد.» (changes applied).
  - On error it removes the overlays and pops false.
- Back or system back → confirmation sheet «خروج از مشخص کردن نقص» (leave defect marking) / «محدودهٔ نقص هنوز مشخص نشده است. آیا مطمئن هستید که می‌خواهید خارج شوید؟» (the area isn't marked yet, leave anyway?) with buttons «ادامه» (continue) and «خروج» (exit) (`:67-91`). It asks even when regions were already drawn.
- Leftovers: `DamageTypeModel` hard-codes one type «تعویض» (replace) and is unused in practice (`spec/data/model/damage_type_model.dart`). `overlayEntries` and `Overlay(key)` are legacy.

### 2.6 "Paint thickness"
- **There is no paint-thickness (paint gauge) screen in this slice.** `paint/` is the image annotation editor (painter), not a paint-depth meter. I found no paint-thickness values, panels or colour coding in these folders. [Inference] If paint thickness is recorded, it is probably a defect type or field in the car_segment components (another slice).

---

## 3. Final approval (supervisor) (`final_approval/`)

### 3.1 Entry and access
- Opened only from the cars tab «تایید نهایی» (final approval) and only if `InspectorSession.isSupervisor`. Otherwise the tap does nothing, silently (`cars/controller/tab_bar_controller.dart:267-281`).
- Args: `slug`, `carName`, `appointmentId = car.appointment ?? car.id`. On return it refreshes the list and tab counts.
- Copy strings are centralised in `final_approval/final_approval_copy.dart`. **Good**: a redesign can reuse this file.

### 3.2 Load sequence (`controller/final_approval_controller.dart:444-500`)
1. `carSegmentRepository.getCarInfo(slug)` gives `inspectionId` and `appointment`. On failure it falls back to the carName.
2. In parallel:
   - `POST ai-lens /inspection-check {slug, role:'supervisor'}`
   - `POST ai-lens /pricing/fmp/carwide-option {appointment_id}`: `option_title`, `distinguishing_feature`, `interphone`, `trimyear_id`
   - `POST ai-lens /inspection-qc {inspection_id}`
   - `carDetailSegmentRepository.getCarInfo(slug)` for trim ids (brand/model/year/trimyear)
3. `FinalApprovalReviewMapper.map` builds the review (§3.4). Acks are cleared.
4. `getCarDetailForm()` loads the `body_color` and `transmission` option lists for the inline editors (`:752-773`).
5. Probe `POST pricing45 /fmp/trim-question/`. If any answer is selectable → `trimQuestionAvailable` → **auto-open the trim sheet** after the first frame (`:480-487,504-520`).
- Error state: an empty review plus `state.error()`. [Inference] The screen then shows empty sections with no error message or retry.

### 3.3 Screen layout (`screens/final_approval_screen.dart`)
- AppBar «تایید نهایی». Scroll body with 24 px padding. Sections, each with a `headlineSmall` title:
  1. «مشکلات عکس‌های کارشناسی» (inspection photo issues): `PhotoIssueCard` per item, or the empty hint «مشکل عکسی برای این کارشناسی ثبت نشده» (no photo issue recorded) (`:134-148`).
  2. «مدارک» (documents): `DocumentStatusCard` per item, or «وضعیت مدارک در گزارش AI این کارشناسی نیست» (document status is not in this inspection's AI report) (`:150-184`).
  3. Either «تایید تریم و آپشن» (confirm trim and option) with `TrimVerificationCard` when the trim Q&A exists, or «تایید آپشن ماشین» (confirm car option) with `CarOptionApprovalCard` (`:186-204`).
- **Sticky bottom bar**, three stacked full-width buttons (`:41-119`):
  - «انتشار» (publish): orange `PrimaryButton`. Disabled unless `canPublishWithAcknowledgedIssues`, and while any action is running.
  - «عدم انتشار» (do not publish): `SecondryButton` → dialog.
  - «برگشت به کارشناسی» (return to inspection): `SecondryButton`.
  - Footer hint «برای مشاهده قیمت به بک آفیس مراجعه کنید» (to see the price, go to the back office).
  - **The supervisor never sees a price in the app** before publishing.
  - [Inference] The three stacked buttons take about 200 px of a phone screen. The two secondary actions look identical, but «عدم انتشار» is terminal.

### 3.4 Review mapper / QC (`data/mapper/final_approval_review_mapper.dart`)
- **Photo issues**: one per `inspection-check.items.inspection_image`. It skips gearbox and green-paper images that have no issue keys.
  - Title = ordered issue labels (people → slug → plate → others) + ` – ` + image title (`:42-57,135-148,217-227`).
  - [Inference] An inspection_image with **no** issue keys that is not gearbox or green paper is still added as an "issue" titled with its image name. Any image the endpoint returns becomes an item to acknowledge.
- **Documents**, two sources:
  - From QC, one per check where `status ∈ {2,3}` (needs review) and **not a live match** (`expected_value` normalised equals `ai_value`; Persian/Arabic digits, commas and دستی/اتوماتیک (manual/automatic) are normalised, `inspection_qc_model.dart:77-147`). Status is always `mismatch`.
    - Titles: «برگه سبز» (green paper), «گیربکس» (gearbox), «کیلومتر» (mileage), «رنگ» (colour), «کنترل برند، تیپ و مدل خودرو» (check brand, trim and model).
    - Shows `reason`, «مقدار وارد شده :» (entered value) `expected_value`, and «تشخیص هوش مصنوعی:» (AI detected) `ai_value`, or «پاسخی یافت نشد» (no answer found) / «وجود ندارد» (missing).
    - Edit kind: mileage → number field; color_check → body-colour dropdown; gearbox_check → transmission dropdown; brand_model_year → **none ("review-only")** (`:257-271`).
  - From inspection-check, the `doc_image` items plus gearbox/green-paper images not already covered by QC. Title gets the suffix «– عدم تطابق تصویر» (photo mismatch); reason = `message` or `meta.error` (`:88-113,171-183`).
- **Car option**: `brand – model – trim – year – option` (option from carwide, or «بدون آپشن» (no option)). An info tooltip shows `distinguishing_feature`, or «راهنمای آپشن برای این خودرو موجود نیست» (no option guide for this car). Plus «شماره داخلی» (internal extension) from interphone (`widgets/car_option_approval_card.dart`).

### 3.5 Cards
- **PhotoIssueCard** (`widgets/photo_issue_card.dart`): a 56 px thumbnail (tap → full-screen zoom ×5, `review_image_thumbnail.dart:5-35`), the title, and an `IssueAcknowledgeButton` «تایید» → «تایید شد» (approved, green check) at 36 px tall (`issue_acknowledge_button.dart`). [Inference] 36 px is a small target, and there is no undo for an acknowledgement.
- **DocumentStatusCard** (`widgets/document_status_card.dart`): green or red tinted card (status is always mismatch in practice), 56 px thumbnail, title, reason, AI value.
  - Manual editor «مقدار ثبت‌شده» (recorded value) with a TextField (digits, thousands separator, LTR) or a `DropdownButtonFormField`, and a «ثبت» (save) button (36 px).
  - Otherwise «آپلود مجدد» (re-upload) or «بارگذاری» (upload) when `canReupload` (mismatch && no manual edit && not brand_model_year).
  - Acknowledge button when `requiresPublishAck` (mismatch).
- Editing or re-uploading **clears that item's ack**, so it must be re-confirmed (`final_approval_screen.dart:165-180`, `controller:890-926`). The item stays in the list after the fix. The copy for «در حال بررسی مجدد…» (re-checking…), «مغایرت برطرف شد» (mismatch resolved) and «مقدار ثبت شد؛ مغایرت هنوز برقرار است» (value saved; mismatch remains) exists, but **no recheck call is made**. `_recheckDocumentAfterUpdate` just invalidates and toasts «مقدار با موفقیت ثبت شد» (value saved) (`:923-926`). `AuctionQcApi` (re-queue QC) exists but is unused.
- Validation: empty mileage → «انتخاب یا وارد کردن مقدار الزامی است» (selecting or entering a value is required). Failures → «ثبت مقدار انجام نشد» (value was not saved) or the API message.
- Edit endpoints (other slices' repos): `carSegmentRepository.saveMileage(slug, mileage)`, `carDetailSegmentRepository.saveCarInfo({transmission}|{body_color})`, `carInfoSegmentRepository.saveCarInfo(trimYearId)`.
- **Re-upload resolution** (`controller:938-1061`):
  1. Find the slot by image URL (normalised, `/W/H/fit` stripped), then by slug hints.
  2. If a template slug is found: the camera/gallery chooser → `PATCH add_image`. Toast «آپلود انجام شد» (upload done) or «آپلود ناموفق بود، دوباره تلاش کنید.» (upload failed, try again).
  3. Else open `CarImagesSegmentScreen(targetImageUuid)`.
  4. Else warn «اسلات این مدرک پیدا نشد؛ از بخش تصاویر خودرو بارگذاری کنید.» (this document's slot was not found; upload it from the car photos section).
  - Thumbnails are cache-busted with an `imageEpoch` query param.

### 3.6 Trim Q&A (`widgets/trim_question_sheet.dart`, `trim_verification_card.dart`, `data/model/trim_question_model.dart`)
- A modal sheet at 85% height titled «سوالات تریم و آپشن» (trim and option questions), hint «پاسخ به این سوالات الزامی است.» (answering is required). It is dismissible.
- Step 1: the question text and a **2-column Wrap of choice tiles** (12 pt title, 10 pt subtitle, which is small).
  - Disabled answers get the subtitle «برای این سال شناسه تریم ندارد» (no trim id for this year).
  - Answers with options get «در مرحله بعد باید آپشن را هم انتخاب کنید» (you must pick an option next).
  - If nothing is selectable: an orange banner «برای این سال هیچ تریم‌ایری پیدا نشد؛ گزینه‌ها قابل انتخاب نیستند.» (no trim found for this year; options cannot be chosen) and «تلاش مجدد» (retry).
- Step 2 (options): «کدام آپشن را می‌خواهید؟» (which option?), option tiles, «تایید آپشن» (confirm option), «بازگشت به سوال» (back to question). **Tapping an option already submits**, so the confirm button is redundant.
- On pick: `POST pricing45 /fmp/trim-answer/ {appointment_id, question_id, answer, option, description:null, trimyear_id}`. Success means `status == 2`. **The result is ignored**: the sheet closes either way (`:172-227`).
  - `question_id` is often missing from the trim-question response. Fallback chain: `lastQuestionId` → `TrimQuestionIdStore` (GetStorage plus a hard-coded seed `'175:1280': 74` «تارا / V4 اتوماتیک», `data/datasource/trim_question_id_store.dart:15-17`) → regex over the raw JSON (`trim_question_api.dart:76-88`). [Inference] This is fragile and needs a backend contract fix.
- Errors: «دریافت سوالات تریم ناموفق بود.» (loading trim questions failed), «سوال و جواب برای این برند و مدل ثبت نشده است.» (no Q&A for this brand and model), «برای این سال گزینه‌ی قابل انتخابی وجود ندارد.» (nothing selectable for this year).
- **Matching** (`controller:582-601`): the selected `trimyear_id` must equal the inspection's `trimYearId`. Fallback: normalised display-line text equality.
  - Match → toast «با کارشناسی قبلی تطبیق دارد.» (matches the earlier inspection).
  - Mismatch card shows «مقدار وارد شده» (entered value) vs «مقدار کارشناسی‌شده» (inspected value), an orange «مقدار وارد شده با مقادیر کارشناسی‌شده تطبیق ندارد.» (entered value does not match the inspection), «ثبت اصلاح روی کارشناسی» (apply fix to inspection: writes trimYearId via `saveCarInfo`, toasts «اصلاح تریم با موفقیت ثبت شد.» (trim fix saved) or «ثبت اصلاح تریم انجام نشد.» (trim fix failed)) and «پاسخ مجدد به سوالات» (answer again).
  - Contact hint «در صورت بروز مشکل با این شماره‌های داخلی تماس بگیرید.» (call these extensions if there's a problem) + interphone.
  - No selection yet → «ابتدا سوالات تریم و آپشن را تکمیل کنید.» (finish the trim and option questions first) + a button to open the sheet.
- **Risk**: the supervisor's answer can **overwrite** the inspector's trim with one tap, and there is no confirmation dialog.

### 3.7 Publish gate and actions (`controller:122-137,671-739`)
- `canPublish` requires all of:
  - review loaded;
  - if trim Q&A is available: a selection exists **and** it matches;
  - every photo issue acknowledged;
  - every mismatch document acknowledged.
- `onPublish` re-checks the gate and shows a message for each failure:
  - «ابتدا سوالات تریم و آپشن را تکمیل کنید.» (finish trim questions first) and reopens the sheet
  - «تا تطبیق تریم و آپشن برقرار نشود امکان انتشار نیست.» (cannot publish until trim and option match)
  - «برای انتشار، همه مشکلات عکس و مدارک را تایید کنید.» (approve all photo and document issues to publish)
- Then `POST ai-lens /fmp {inspection_id}`. It needs `status == 2` and `base, step, price, target` present, otherwise «شروع مزایده انجام نشد» (auction start failed).
- Then `POST {base}/api/back_office/appointments/{id}/change_status/` with `{transition_slug:'scheduled_auction', form_data:{start_price: base, step_price: step, fair_market_price: price, target_price: target, tags:[int], start_time: now+35s 'yyyy-MM-dd HH:mm:ss' (device clock), duration: 'H:MM' from auction_time (default 20)}}` (`data/datasource/start_auction_api.dart:26-79`, `data/repository/start_auction_repository.dart:28-46`).
- Success → `Get.offAll(HomeScreen)` + «با موفقیت شروع مزایده ثبت شد» (auction start recorded).
- **Business meaning**: publishing = **starting the online auction (مزایده) immediately**, with prices fully dictated by the FMP model. No human sees or edits start, step or target in the app.
  - [Inference] `start_time` uses the device clock and local timezone, which is a risk if the phone clock drifts.
- **عدم انتشار** (do not publish) (`controller:212-442`):
  - Dialog with a reason dropdown. Reasons come from `GET back_office/statuses/active-reasons/`, filtered to status group id **23** (`AppointmentStatus.notPublished`, `lib/core/utils/constant.dart:89`), and picked in a bottom-sheet list.
  - Optional image «تصویر (اختیاری)» (photo, optional) with «افزودن تصویر» (add photo) / «حذف تصویر» (remove photo). It is uploaded via `PATCH add_image` with `template_slug: non-publishable-reason-doc`.
  - Then `change_status {transition_slug:'not_auctioned', form_data:{'__reason__': reasonId}}`.
  - Required: «انتخاب علت عدم انتشار الزامی است.» (choosing a reason is required). Success → home + «وضعیت به «عدم انتشار» تغییر کرد» (status changed to not published).
  - If the reasons list is empty, the button only shows «ثبت عدم انتشار انجام نشد» (not-published could not be saved).
- **برگشت به کارشناسی** (return to inspection) (`:1188-1209`): `change_status {transition_slug:'re_inspection'}`. **No confirmation, no reason field.** On success `Get.back()` with no toast. On failure it does nothing visible except the global Dio toast.
- `AppointmentTransition` (`lib/core/utils/constant.dart:101-110`): `re_inspection` (پایان کارشناسی → در حال کارشناسی: end of inspection → in inspection), `not_auctioned`, `scheduled_auction` (→ waiting for auction).

### 3.8 Pricing / auction screen (`screens/pricing_auction_screen.dart`, `controller/pricing_auction_controller.dart`): **UNUSED**
- No `PricingAuctionScreen(` call exists anywhere outside its own file. [Inference] It is the earlier or alternate design: «تایید و مشاهده قیمت» (approve and view price) existed in copy, and it was replaced by "publish = auction" plus «برای مشاهده قیمت به بک آفیس مراجعه کنید». It is still the best spec for "what AI price suggestion could be shown":
  - Step 1 «مرحله ۱ - قیمت اعلامی» (step 1 – announced price): «قیمت تخمینی (بازه)» (estimated price range) = FMP `dealer_price_min – dealer_price_max` (toman), scaled by smart-margin `GET /smartmargin/pre-negotiation-price?appointment_id` → `override.fraction_of_fair_market_price` (`:379-419`). Formatted LTR "min – max" at 22 pt. If unavailable: «تخمینی وجود ندارد» (no estimate).
  - Step 2 «مرحله ۲ - اینپوت فروشنده» (step 2 – seller input): «قیمت پیشنهادی فروشنده» (seller's asking price), a numeric field with thousands separator. Helper «این فیلد از فروشنده گرفته می‌شود» (this comes from the seller).
  - Debounced 1 s or on blur → `GET /smartmargin/check-seller-suggested-price?seller_suggested_price&expected_price=dealer_price` → `{ok, ratio, bound_fraction, threshold, excess}` (`data/datasource/check_seller_suggested_price_api.dart`, `domain/price_comparison.dart`).
  - Comparison card «مقایسه» (comparison) with tag «منطقی» (reasonable) / «غیرمنطقی» (unreasonable) and text such as «اختلاف X٪ با میانگین بازه - کمتر از حد مجاز Y٪: قیمت فروشنده قابل قبول است.» (X% off the range average, under the Y% limit: seller price is acceptable) (`domain/estimated_price_formatter.dart:24-38`).
  - Banner «قیمت فروشنده در بازه تخمینی - مزایده مجاز» (seller price in range – auction allowed) / «… خارج از بازه مجاز - مزایده مجاز نیست» (out of range – auction not allowed).
  - **«شروع مزایده» (start auction) is NOT gated by the comparison**: it is enabled even if the price is «غیرمنطقی» (`pricing_auction_screen.dart:139-148`). The banner says "not allowed", but nothing enforces it. Same start-auction and not-published logic as above.
- FMP model fields available (`data/model/fmp_apiview_model.dart`): `confidence_level`, `is_valid`, `price_classified(_min/_max)`, `dealer_price(_min/_max)`, `k45_price(_min/_max)`, `price` (fair market), `backoffice_price`, `base`, `target`, `step`, `auction_time`, `tags`. [Inference] This is a rich AI-price payload the redesign could expose, depending on role: confidence plus a range.

### 3.9 Other unused or dead pieces
- `BrandModelYearDocumentEditor` (`widgets/brand_model_year_document_editor.dart`): a cascading dropdown editor (برند (brand) → مدل (model) → تیپ (trim) → سال (year) → آپشن (option)) via `carInfoSegmentRepository.getBrands/getModels/getTrims/getYears/getTrimYears`. It submits a `trimYearId`. It is not wired: the controller's `onSubmitDocumentBrandModelYear` (`:820-854`) has no caller, and the mapper sets brand_model_year to review-only.
- `FinalApprovalViewOnlyBanner` «فقط مشاهده – بدون امکان ادیت عکس» (view only – photos cannot be edited) and `FinalApprovalActionsBar` are unused.
- `AuctionQcApi` (`POST ai-lens /auction-qc {inspection_id, callback_url: …/api/back_office/newinspection/qc/{slug}/auction_qc_ai_report/}`) is unused, and so is `AuctionQcRepository`.
- `FmpDefectListRepository` **is used**: by the car_svg screen for body segments (`car_svg_controller.dart:173-175`) and by the AI-metadata policy. It is an in-memory cache with a 24 h TTL.

---

## 4. Screen inventory (my slice)

| Screen / sheet | File | Purpose | In / out |
|---|---|---|---|
| CarImagesSegmentScreen | car_images/screens/car_images_screens.dart | photo + video slots, or documents | in: segment list, final approval re-upload, AI retry; out: pop on «ثبت اطلاعات», or pop(uploaded) in target mode |
| Image chooser sheet | lib/core/utils/utils.dart:278 | camera / gallery / video | from grid cards, defect sheet, doc re-upload |
| PreviewBottomSheet | lib/core/widget/bottom_sheet_view.dart | slider, replace, delete, «شخصی سازی» | → PaintImageScreen |
| CameraPage (quick shoot) | camera/camera_page.dart | guided sequential capture | Get.to from grid; Get.back at end |
| PaintImageScreen | paint/screen/paint_screen.dart | annotate photo, plate sticker, upload | pops ImageModel |
| CarSvgScreen | car_svg/screen/car_svg_screen.dart | top-down diagram, defect markers | back/list row → Get.off(CarSegmentScreen); submit → pop or pushReplacement(incomplete list) |
| SelectComponentSheet / SelectDefectSheet | car_svg/widget/select_defect/ | narrow tap to component, then defect | returns via Get.back(result) |
| SpecifyingPointsScreen | spec/screen/specifying_point_screen.dart | draw damage region → area grade + centroid | pops bool |
| ShowSpotImageWidget sheet | car_svg/widget/show_spot_image/ | defect photos + شدت/مساحت (severity/area) rails | dismiss; may delete draft |
| WholeSvgInfoWidget | car_svg/widget/whole_svg_info.dart | help sheet | — |
| FinalApprovalScreen | final_approval/screens/final_approval_screen.dart | supervisor QC gate + publish | from cars tab (supervisor); out: offAll(Home) or back |
| TrimQuestionSheet | final_approval/widgets/trim_question_sheet.dart | mandatory trim Q&A | auto-opens |
| Not-published dialog + reason sheet | final_approval_controller.dart:147-307 | reason + optional photo | → Home |
| PricingAuctionScreen | final_approval/screens/pricing_auction_screen.dart | **unused** | — |

---

## 5. API endpoints touched

Main API (DioClient, `ApiEndpoints.baseUrl`, `/api/v2`):

| Method | Path | Purpose | Key fields |
|---|---|---|---|
| GET | `/inspection/{slug}/imagetemplate/` | image slots by category | `[{name,slug,segment_type,images:[{title,slug,can_skip,inspection_order,icon,image}]}]` |
| PATCH | `/inspection/{slug}/add_image/` | upload slot photo/video (also paint result, doc re-upload, not-published reason doc) | multipart `image`, `template_slug` → `uploaded_image{uuid,mime_type,url}`; 60 s timeouts |
| DELETE | `/inspection/{slug}/remove_image/` | delete slot media | body `{file_uuid}` |
| GET | `/inspection/{slug}/{segmentSlug}/components/` | components + defects for the diagram | ComponentModel list (defects: `defect{name,slug,icon}`, `severity`, `severity_color`, `highlight`, `required_media`, `coordinate`, `images`, `ai_metadata`, `has_defect`, note) |
| PATCH | `/inspection/{slug}/add_defect/` | record defect / save coordinate / upload defect file / save ai_metadata | `{component_slug, defect_slug, coordinate:["x","y"]}` (spec); file + ai_metadata variants in car_segment repo |
| (car_segment repo) | removeDefect, validateDefects | purge drafts; segment validation | other slice |
| (other repos) | getCarInfo, saveMileage, saveCarInfo(carDetail), saveCarInfo(trimYearId), getCarDetailForm | final approval reads and edits | other slices |

Back office (`/api/back_office`):

| Method | Path | Purpose |
|---|---|---|
| GET | `/statuses/active-reasons/` | reason groups; app takes group id 23 (not published) |
| POST | `/appointments/{id}/change_status/` | `transition_slug` ∈ `scheduled_auction` (with `form_data` prices, `start_time`, `duration`, `tags`), `not_auctioned` (`__reason__`), `re_inspection` |

AI-lens gateway (`AiLensClient`; staging and prod URLs in `lib/core/utils/ai_lens_config.dart`). All calls suppress the global error toast and swallow errors to null.

| Method | Path | Purpose |
|---|---|---|
| POST | `/inspection-check` | per-image AI verdicts `{slug, role}` → `items.inspection_image[]`, `items.doc_image[]` |
| POST | `/inspection-qc` | document QC checks `{inspection_id}` → `checks[{check,status,title,image_url,image_slug,reason,expected_value,ai_value}]`, `qc{id,modified_time}` |
| POST | `/pricing/fmp/carwide-option` | option title, guide, interphone |
| GET | `/pricing/fmp/defect-list` | defect catalog (`label_severity`, `label_area`, `price_drop_pct`, `priority_rank`, `family`, `detect_difficulty`) |
| POST | `/fmp` | price model output `{inspection_id}` (§3.8 fields); status 2 = ok |
| GET | `/smartmargin/pre-negotiation-price` | `fraction_of_fair_market_price` (unused screen) |
| GET | `/smartmargin/check-seller-suggested-price` | seller price sanity check (unused screen) |
| POST | `/auction-qc` | re-run QC with a callback (unused) |

pricing45 (`FmpClient`):

| Method | Path | Purpose |
|---|---|---|
| POST | `/fmp/trim-question/` | `{brand_id, model_id, year_id, appointment_id}` → question + answers (list or map) + options |
| POST | `/fmp/trim-answer/` | `{appointment_id, question_id, answer, option, description, trimyear_id}`; status 2 = ok |

Local: `assets/jsons/car_svg_part_pixel.json` (diagram geometry).

---

## 6. CHANGELOG highlights (`CHANGELOG.md`, last updated 2026-08-23)
- Scope: the `ai-team` branch compared with `main`. 94 files, +7,538/−1,082 lines, 41 commits ahead.
- **Major**:
  - AI Check System: an AI Check Result Screen (1,698 lines), AI Lens integration for mileage and gearbox, and an "AI Metadata Policy" (defect grading).
  - Mileage check and gearbox green-paper check cards.
  - A Defect Grade Scale widget.
  - Category accordion cards, a visit details bottom sheet, a status chip and a status filter bar.
- **Infra**: AppLogger, global error handlers, a redacting API error interceptor, Sentry upgrades, a Dio memory cache, an image-upload Sentry tracker (341 lines), an inspection image URL cache, the inspector session, and Map IR (map provider) config.
- **Flow**: a merged «کارشناسی» (inspection) tab (ready + in progress); onsite inspection support (address-based UI, a single onsite list API by profile branch, visit details from the location button); improved login with server selection.
- **Media**: car images controller and screen improvements; the paint controller uploads with Sentry tracking; `AppMediaPicker` refactored with **web support** (there is a PWA manifest).
- **Build**: Kotlin 2.2.0, Java 17, GetX 4.7.3; Docker/Nginx removed; `defect-list.json` (50 lines) added.
- **Not in the CHANGELOG but visible in code** (newer than 2026-08-23, [Inference]): the final approval / supervisor flow (trim Q&A, QC docs, publish = scheduled auction), photo-first deferred SVG defects, the drawn-area grading, and local camera guides (backend icons 404). The CHANGELOG is a branch diff summary, not a versioned release log.
- Direction of travel: from inspector data entry → **AI-checked** capture (AI lens, QC) → **AI-labelled** defects (severity/area for the FMP price model) → a **supervisor gate that auto-starts the auction** from model prices.

---

## 7. UI defects and redesign targets

### Hardcoded values and inconsistency
- Hex colours are scattered:
  - `0xfff6f6f6` (grid background)
  - `0xffff4869` (error red, three places)
  - `0xFFff783f` (orange default in paint, three places)
  - `0x8E1C2631` (edit chip)
  - `0xff252B32` (camera background)
  - `0xff84888b` (spec background)
  - `0xff7a93af` / `0xffd0ddec` (legend)
  - `0xFF1F2937` (tooltip)
  - `0xE5E53935` (region default)
  - …plus the painter's ~100 literal colours.
- They mix `AppColors.*`, `context.colors.*` (uicat) and literals. Error red appears as `0xffff4869`, `AppColors.redColor` and `context.colors.error`.
- Button families are mixed: `UICatButton`, `CupertinoButton`, `PrimaryButton`/`SecondryButton` (sic), `OutlinedButton`, `TextButton`, `GestureDetector`+Container, and `.onTapOpaque`. There are three different close-button styles (32 px greyColor_900 square, 44 px, `IconButton`).
- Fixed pixel sizes: card image 175×131 inside a 180 px grid cell; `(Get.width-100)*0.7` thumbnail width; `Get.width*4/3` camera; SVG width `Get.width-32`. The layout keys off `Get.width`, not constraints, so it breaks on tablets, split screen and landscape.
- Typography: 10–13 pt for important info (trim tile subtitles 10 pt, doc fields 13 pt, component slug 11 pt shown to inspectors as a secondary label, which leaks the internal slug as UI text).
- The camera overlay code is duplicated three times. The not-published dialog is duplicated across two controllers.
- `WillPopScope` is deprecated. `withOpacity` is deprecated (painter, `show_image.dart:274`).

### Touch targets, gloves, one hand, glare
- Small targets:
  - edit chip ≈32 px (`car_images_screens.dart:487-497`)
  - retry icon ≈28 px (`:342-358`)
  - AppBar info icon 24 px with no padding (`car_svg_screen.dart:44`)
  - defect markers ≈24 px (`:118-128`)
  - part hit boxes minimum 32 px, adjacent and invisible until pressed (`part_hit_box.dart:20`)
  - ack and save buttons 36 px (`issue_acknowledge_button.dart:19,34`; `document_status_card.dart:211,336`)
  - choice tiles about 40 px with 12/10 pt text
- **Drawing the damage outline with a finger** on a panel that is about 40% of the screen, and on the pillars about 20%, is hard with gloves or outdoors. There is no zoom or pan, and no undo of the last region (only «پاک کردن» clears all).
- The camera is landscape-only in quick shoot, yet the controls rotate. The «رد کردن» (skip) and back buttons swap corners with rotation. [Inference] One-handed use in landscape is awkward, and the 88 px shutter is fine.
- Outdoor glare:
  - The white guide outline over a bright car or sky has low contrast and no shadow or outline stroke.
  - The `0xff7a93af` / `0xffd0ddec` legend greys are low contrast.
  - Grey-on-grey text (`black_60` 12 pt hints).
  - The diagram's blue-grey palette makes the pink "replaced" tint and the white "selectable" state hard to tell apart.
- Info-dense final approval: three stacked bottom buttons and long per-card editors. There is no summary of how many items are still unacknowledged («۳ مورد باقی‌مانده», "3 remaining"), so the user cannot see why «انتشار» is disabled.

### Dated or broken UX
- The help sheet text is stale (§2.1). The tap-hand Lottie contradicts the "draw" instruction.
- The component list row leaves the diagram (`Get.off`) and drops context. System back on CarSvg replaces the route rather than popping.
- No offline or poor-network handling at all (§1.4). Failures inside quick shoot are silent.
- Auto-saving every photo to the gallery (privacy, storage).
- `image.jpg` filename for WebP bytes.
- The «شخصی سازی» label for annotate. The edit pencil re-captures rather than editing.
- Flash is a binary always/none toggle with no auto or torch-for-engine-bay mode.
- Final approval's error state is empty sections with no retry.
- The trim-answer failure is ignored. «تایید آپشن» (confirm option) is redundant. The trim fix has no confirmation.
- «برگشت به کارشناسی» (return to inspection) and «عدم انتشار» (do not publish) are visually equal to each other and close to «انتشار» (publish).

---

## 8. What a redesign must preserve, and what is risky

### Must preserve (business rules and safety checks)
1. **Required-slot gate**: `can_skip=false` slots block «ثبت اطلاعات» (submit). Document slots are separate (`hasMissingRequiredImages`). Enforced in the **client UI + controller**. [Inference] I cannot confirm the server re-validates images; for defects the server does validate (`validate_defects`).
2. **Capture constraints**: video 10–30 s, under 100 MB; photo under 20 MB; documents uncompressed (OCR/AI quality); 4:3 back camera; guided slot order by `inspection_order`.
3. **Photo-first defects on the diagram**: no `add_defect` for an SVG spot until the photo is uploaded. Drafts without media are deleted. `highlight==1` needs a coordinate (mirrors backend `validate_non_empty_coordinate_inspection_defect`). Replaced and cut-and-replace skip drawing and tint the panel. Enforced in the **client** with **server** validation at segment submit.
4. **ai_metadata** severity/area are integers 1–5 (server rejects otherwise). Shown only for FMP-catalog defects with `label_severity/label_area`, never for replacements, and only after a photo exists. The area is prefilled from the drawn polygon (20% bins over the part fill ratio). These are **training and pricing labels**, so the data contract matters more than the UI.
5. **Mandatory annotation pass** for defect photos (plate sticker, marks), plus the people/plate AI checks. [Inference] The plate sticker is a privacy control. Keep it, or replace it with auto-blur.
6. **Final approval gates**: supervisor-only (client check in tab_bar_controller; [Inference] the server must also enforce it); ack every photo issue and mismatched doc; any edit or re-upload clears the ack; trim must match when Q&A exists; a not-published reason is required (group 23); the three transitions `scheduled_auction` / `not_auctioned` / `re_inspection`.
7. **Auction parameters come from FMP** (`base`, `step`, `price`, `target`, `tags`, `auction_time`); publish fails closed if any are missing. Start is now+35 s. Duration is "H:MM".
8. Side naming راننده (driver) = left, شاگرد (passenger) = right, and the backend slug quirks (`rights-mirror`, `driver's-…`, `student's-…` for passenger).

### Risky areas (redesign or decide explicitly)
- **Publish is one tap from a live auction**, with prices nobody reviews in-app, a device-clock start time and no confirmation dialog. Strongly consider a confirm step that shows start, target and duration, and possibly the FMP confidence. Which layer enforces that? Today it is only the client UI gate. [Inference] The backend `validate_scheduled_auction` checks format, not business sense.
- The unused pricing screen shows a "not allowed" banner but does not block the auction. If it is revived, decide whether the smart-margin check is **advisory or blocking**, and enforce that in the backend `change_status` validation, not only in the UI.
- «برگشت به کارشناسی» (return to inspection) has no confirm and no reason. «عدم انتشار» (do not publish) is terminal.
- The trim fix overwrites inspection data silently. The trim-answer API result is ignored, and question_id relies on a seeded or local store.
- Submit-time **silent purge** of incomplete defects (server deletes).
- No upload queue: moving to offline-first needs idempotency on `add_image`/`add_defect` (the server keys by `template_slug` or component+defect, so a retry overwrites. [Inference] This looks safe for slots, but defect images append).
- Drawn polygons are not persisted, so an edit cannot redraw the previous area; only the centroid survives.
- The hand-coded 6,952-line painter plus the JSON hit boxes plus the in-code extra regions are three sources of geometry. Changing the diagram art means re-deriving all of them. Unreachable components fall back to the list (logged by `_logUnreachableComponents`).
- Acks live only in memory: leaving the screen resets every acknowledgement (`acknowledgedIssueKeys.clear()` on load). There is no audit of who acknowledged what. [Inference] For a supervisor QC decision that starts an auction, an ack trail probably belongs server-side, in the system of record.
- Gallery auto-save of documents (national card, car card) is a privacy risk.
