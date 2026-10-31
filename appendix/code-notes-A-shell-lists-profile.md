# Notes A: Khodro45 inspection app (Flutter/GetX), shell + lists + profile slice

Source: `/Users/mj.khorshidsavar/Downloads/inspection-app-new-main` (read-only). Package `inspection`, version `1.0.0+100` (pubspec.yaml:4).
Paths below are relative to `lib/` unless stated. `[Inference]` = my reading, not stated in code.
Slice covered: main.dart, core/** (utils, api, service, widget, config, handler), data/** (all dead/commented), l10n/app_fa.arb, screens/splash, login, home, cars, profile (+my_inspection), qc_report.

---

## 1. App purpose, bootstrap, env, auth, update, offline, logging, routes

### Purpose / users
- Internal field app for Khodro45 inspectors (کارشناس) and branch supervisors (سرپرست). Lists appointments/inspections assigned to the user's branches, starts or continues an inspection (hands off to `SegmentScreen`, outside slice), lets supervisors do document upload (بارگذاری مدارک) and final approval (تایید نهایی), shows the inspector's monthly stats and the QC report.
- Internal product name "Copilot": Android app id `com.khodro45.copilot.inspection`; label "Copilot Inspection" / "Copilot Inspection Staging" (android/app/build.gradle.kts:34,47-59). Comment says it coexists with the older "K45 Inspection" app `com.khodro45.inspection`. Release-app API key `app = 'copilot'` (core/service/release_app/release_app_api.dart:11). Logcat tag `CopilotAPI` (core/utils/app_logger_config.dart:55).
- ARB `appName` / `inspection` = "اپلیکیشن کارشناسی"; `login_title` = "سامانه جامع کارشناسی خودرو" (unused in login UI).
- Platforms: Android, iOS, web (kIsWeb branches exist), desktop folders present. Portrait locked (main.dart:107).

### Bootstrap (main.dart)
1. `_baseUrlForFlavor()` (main.dart:21-32): `appFlavor` or `--dart-define=APP_ENV`. `prod` → production, `staging` → staging, anything else/null → **staging**.
2. `debugPrint` silenced in non-prod unless logging enabled (main.dart:42-44). In prod debugPrint stays on (comment: so Sentry `debug:true` logs show).
3. Sentry only in production (main.dart:57-89): DSN hardcoded (sentry.hamravesh.com), `debug = true` in prod, sampleRate 0.5, tracesSampleRate 0.5, auto performance tracing on, frames/user-interaction tracing off. If init fails, app still starts.
4. `_bootstrap()` (main.dart:92-140): white status/nav bar; portrait; `GetStorage.init()`; non-prod writes `baseUrl` title to storage; `LoginRepository.loadLoginToken()`; starts `AppInternetChecker`; wraps `FlutterError.onError` and `PlatformDispatcher.onError` to Talker.
5. `GetMaterialApp` (main.dart:191-290): `defaultTransition: Transition.noTransition` (no page animations anywhere), RTL, locale `fa` only, `ThemeMode.light` only (no dark mode), `home: SplashScreen`, observers: `SentryNavigatorObserver` + optional `TalkerRouteObserver`.
6. Global `builder` layers (outer → inner): white `ColoredBox` + manual status-bar spacer (fix for black bar after lock/unlock, main.dart:235-243) → `CustomDebugBanner` → offline-pending banner (ValueListenable) → optional draggable Talker FAB → `_OfflinePendingBanner` → tap-anywhere-to-unfocus GestureDetector.
   - **Bug**: the offline pending banner is rendered twice (main.dart:249-279 and `_OfflinePendingBanner` 312-353). Both show `offline_retry_pending_count` "{count} مورد در انتظار ارسال" on `colorScheme.secondaryContainer`, elevation 2. [Inference] user sees two stacked identical strips when requests are queued.
   - App lifecycle `resumed` re-applies overlay style + `setState` (main.dart:173-187); `didChangeMetrics` also rebuilds whole app.

### Environments / base URLs (core/utils/base_url.dart)
| enum | URL |
|---|---|
| bmw | https://bmw.technical45.com/api/v2 |
| dev | https://dev.technical45.com/api/v2 |
| hamilton | https://hamilton.technical45.com/api/v2 |
| nissan | https://nissan.technical45.com/api/v2 |
| staging | https://staging.technical45.com/api/v2 |
| production | https://khodro45.com/api/v2 |
- Default static value `ApiEndpoints.baseUrl = BaseUrl.hamilton` (core/utils/api/api_endpoints.dart:6) but always overwritten in main.
- Derived hosts: `/v1` (login), `/back_office` (appointments, roles) by string-replacing `/v2`; QC report `https://backoffice.{env}.technical45.com/inspection-qc-report/{slug}` or `https://backoffice.khodro45.com/...` (screens/qc_report/qc_report_page.dart:18-24); car-detail JSON per env (core/utils/constant.dart:4-15).
- Side services (separate Dio clients, no login token, own retry interceptor): AI Lens (`ai-lens.technical45.com` / `ai-lens.c54.darkube.ir`, Bearer), FMP pricing (`pricing45.technical45.com` / `pricing45.khodro45.com`, Token), Smart Margin (`smartmarginprod.c54.darkube.ir`, Bearer from `--dart-define=SMART_MARGIN_TOKEN`, default 0.96 × fair market price). Files: core/utils/ai_lens_config.dart, fmp_config.dart, smart_margin_config.dart, api/*_client.dart.
- **Risk**: static service tokens for AI Lens (staging+prod), FMP pricing, and a Map.ir JWT are hardcoded in source (ai_lens_config.dart:8,11; fmp_config.dart:17; core/config/map_ir_config.dart:2). Values not copied here. [Inference] Map.ir JWT payload has `exp` in 2020, so map tiles in the visit sheet may fail.
- Dev server switcher on login: button "تغییر سرور" (dark #222222), only when `appFlavor == null` and not prod (screens/login/login_page.dart:103-153). Sheet "انتخاب سرور" / "سرور اپلیکیشن را برای تست انتخاب کنید." lists all non-prod envs; selecting writes storage and calls `SystemNavigator.pop()` (kills app; user must relaunch). [Inference] stored `baseUrl` is never read back at startup (main overwrites from flavor), so the switch may not persist. Not verified.
- Flavors: `prod`, `staging` (build.gradle.kts:47-59). `run_android.sh` defaults `FLAVOR=staging`, passes `APP_ENV`, `LOG_API`, `SMART_MARGIN_TOKEN`.

### Auth flow
- **Username + password, no OTP.** `POST {base with /v1}/auth/login/` body `{username, password}` (screens/login/data/datasource/login_datasource.dart:18-21). Response `{token}` (login_model.dart).
- Digits converted Persian→English (`toEnNumber`) for both fields (login_controller.dart:51).
- Token stored in **GetStorage key `token`, plain** (login_repository.dart:28-36), mirrored in `ValueNotifier<String?> LoginRepository.loginChangeNotifire`. Sent as `Authorization: Token <token>` on every DioClient request unless `extra['requiresAuth'] == false` (core/utils/api/dio_client.dart:130-135).
- 403 → bottom sheet "حساب شما غیرفعال شده است" + text "کارشناس گرامی، حساب شما غیرفعال شده است. جهت پیگیری و رفع این مشکل، لطفاً با شماره ۱۴۸۵ تماس حاصل فرمایید و داخلی ۲۸۰ را شماره‌گیری نمایید." + Lottie `Blocked.json` + button "تماس با ۱۴۸۵" (tel:1485) (login_controller.dart:52-67). Note: on 403 the repo still writes `token=''`.
- Success → `InspectorSession.clear()`, `VersionControl.reportInstall()`, `Get.offAll(HomeScreen)`.
- Other DioException (e.g. 400 wrong password) → `return` silently (login_controller.dart:78-80). Since API error toasts are globally disabled (see below), **wrong password shows no feedback**. [Inference, high confidence]
- Sentry user set to username *before* the login call (login_controller.dart:47).
- Forgot password: no flow; sheet "فراموشی رمز عبور" + "کارشناس گرامی، جهت دریافت اطلاعات ورود خود، لطفاً با شماره ۱۴۸۵ تماس بگیرید و داخلی ۲۸۰ را شماره‌گیری نمایید." + Lottie `call.json` + "تماس با ۱۴۸۵" (login_controller.dart:88-103).
- Session expiry: any 401 → `ApiErrorType.sessionExpired` → `logOut()` + `Get.offAll(LoginPage)` (dio_client.dart:66-73, 188-192). No refresh token.
- Logout (`LoginRepository.logOut`, login_repository.dart:38-50): clears `InspectorSession`, `GetStorage.erase()` except `release_app_device_id`, nulls notifier. Profile logout also clears Sentry user.

### Version control / in-app update (core/service/version_control.dart)
- On splash: `GET /release-app/check/copilot/{android|ios}/{x.y.z}/` (no auth, toast suppressed). Response `{should_update, force, download_url}`. If update: also `GET /release-app/retrieve/copilot/{device}/` → `{latest, supported, latest_version, supported_version, update_url, description}`; then `Get.offAll(UpdateRequiredPage(force, downloadUrl, description, onContinue))` (screen in `screens/update`, outside slice). Optional update's "continue" goes to Home or Login.
- After login: `POST /release-app/report/` `{app, device, device_id, device_model, version}` (release_app_api.dart:52-70). Device id = persisted random `{platform}_{ms}_{rand}` in GetStorage `release_app_device_id` (iOS uses identifierForVendor). Web reports as `android`.
- Profile "به روزرسانی نرم‌افزار" card re-checks and opens update sheet or success snackbar "شما درحال استفاده از آخرین نسخه برنامه می‌باشید." (profile_controller.dart:69-199). Update opens external URL (APK download) via `launchURL` (no store).
- Version semver strips `-staging` suffix (device_identity.dart:12-15).

### Network stack (core/utils/api/dio_client.dart) – interceptor order
1. `DioMemoryCacheInterceptor`: GET-only, opt-in via `extra['cacheDuration']` (in-memory, per URI).
2. VPN check: reads response header `k45-have-vpn: true` once per run → top warning snackbar "برای تجربه بهتر فیلتر شکن خود را خاموش کنید." 5 s (dio_client.dart:158-181).
3. `ApiErrorLoggerInterceptor` (only with `--dart-define=LOG_API=true`).
4. Main wrapper: onRequest: sets `validateStatus` 2xx only, `followRedirects=false`, rejects FormData files > 100 MB ("حجم فایل بسیار بزرگ است. حداکثر حجم مجاز 100 مگابایت است."), sets FormData sendTimeout 120 s if null, attaches token. onError: Sentry report only for 5xx; image uploads (`PATCH .../add_image/`, `PATCH .../add_defect/` with file) tracked separately by `ImageUploadSentryTracker` (breadcrumbs, slow-upload warning ≥30 s, fingerprint).
5. `OfflineRetryInterceptor` (offline queue, below).
6. `RequestRetryInterceptor`: up to 3 retries, backoff 1/2/4 s, for timeouts, connection errors, 502/503/504. Opt-out `extra['suppressRequestRetry']`.
7. `dio.addSentry(captureFailedRequests:false)`.
- Timeouts: 20 s connect/send/receive (api_request_policy.dart:5); `/inspection-check` 75 s.
- **Error toasts disabled globally**: `const bool errorToastsEnabled = false;` "Temporary kill switch" (core/utils/api/api_error_toast.dart:7). `showApiErrorToast` is a no-op, which also silences client-side validation toasts in `AppMediaPicker` (image >20 MB "حجم تصویر بیش از ۲۰ مگابایت است.", wrong aspect "تلفن را بچرخانید سپس تصویر بگیرید\n اندازه تصویر اشتباه است", video >100 MB, video duration <10 s / >30 s). [Inference] inspectors get silent failures on bad photos/videos.
- Error typing (api_error.dart): offline (timeouts/connection/SocketException), sessionExpired (401), validation (4xx), server (5xx), unknown. Server message extraction: `detail`, `result.message`, first value string/list (max 3 joined).
- Legacy helpers `internetConnectionDialog` / `serverError` in core/utils/utils.dart:65-176 (500 dialog with huge "۵۰۰" at fontSize 110) appear unused [Inference: not referenced in slice].

### Offline handling and retry (core/utils/api/offline_retry_interceptor.dart)
- Queues any request failing with an "offline" type when a navigator context exists (not web). Up to 3 attempts, backoff 1/2/4 s. `pendingCount` ValueNotifier drives the banner.
- Retries when: already online (connectivity_plus check, 800 ms double-check), on `onConnectivityChanged` to online, or after a 3 s "quiet window" following any lifecycle change (app switch reports brief "none").
- If still offline: non-dismissible UICat bottom sheet title "اتصال نداشتن به اینترنت", description "اتصال به اینترنت را بررسی و دوباره تلاش کنید.", Lottie `Wifi.json` 150×150 (fallback `no-wifi.png`). Auto-closes when queue drains. **No manual retry button** in this sheet.
- Queue is in memory only (lost on app kill). Replays FormData via `clone()`.
- Opt-out `extra['suppressOfflineRetry']` used by badge/count calls.
- `AppInternetChecker` (core/service/app_internet_checker.dart) only updates `isDeviceConnected` (InternetConnectionChecker); its old sheet is commented out. Older widget `InternetConnectionBottomSheet` ("خطای اتصال به اینترنت" / "اتصال شما به اینترنت قطع شده است. لطفا اینترنت خود را بررسی و مجدد اقدام نمایید." / "تلاش مجدد") still exists.

### Logging / observability
- Talker logger (core/utils/app_logger.dart). Everything gated by `--dart-define=LOG_API=true` (app_logger_config.dart:14-45); when on, enabled even in production (`forceEnableOnProduction = logApiFromEnv`). Talker FAB servers list includes production.
- Draggable debug FAB (bug icon, `secondary800` pill r40) opens Talker console (main.dart:214-228, 355-379). Only when logging enabled.
- `CustomDebugBanner` (core/widget/custom_debug_banner.dart): red (alpha 150) strip at top showing env title (left) and `v{version}` (right), 10 px bold white, LTR. Shown in **any debug build or any non-prod env**; hidden only in release + prod. It overlays the top of every screen (covers app bar area).
- Sentry: `ErrorReporter` (core/service/error_reporter.dart) wraps captureException/message with endpoint tag, contexts, fingerprint. `sampleByLevel` defined but not wired in slice [Inference].

### Navigation map (GetX imperative, no named routes)
```
SplashScreen (logo 248w, white)  -- splash_controller.dart
 ├─ update needed → Get.offAll(UpdateRequiredPage)   [screens/update]
 ├─ token present → (2.5 s delay) Get.offAll(HomeScreen)
 └─ else          → (2.5 s delay) Get.offAll(LoginPage)
LoginPage → success → Get.offAll(HomeScreen)
HomeScreen (CustomAppBar: logo title, leading profile icon, action headphone)
 ├─ profile icon → Get.to(ProfileScreen)
 ├─ headphone → support sheet "پشتیبانی و راهنمایی": tel:1485 "تماس با پشتیبانی"/"داخلی ۲۸۰"; "راهنما" → TermAndConditionScreen
 ├─ tab 0 بارگذاری مدارک → card "بارگذاری مدارک" → SegmentScreen (start_inspection first if status==1); "گزارش کارشناسی" → QcReportPage
 ├─ tab 1 کارشناسی → card "شروع/ادامه کارشناسی" → [PATCH start_inspection if ready] → SegmentScreen; location btn → VisitDetailsBottomSheet → same start
 └─ tab 2 تایید نهایی → card → FinalApprovalScreen(carName, slug, appointmentId)   [screens/final_approval]
   (on return from any: list reload + badge refresh)
ProfileScreen → BranchScreen | MyInspectionScreen (→ QcReportPage) | update sheet | logout sheet → Get.offAll(LoginPage)
Any 401 → Get.offAll(LoginPage)
```
- Splash always waits 2.5 s after the update check (splash_controller.dart:26-36), on top of the network round-trip (check timeout up to 20 s + 3 retries). [Inference] cold start can be slow on bad networks.
- Splash UI: Flutter screen shows `kh_logo.svg` centered on white; native splash color `#24536C` (pubspec.yaml) → visible teal→white flash.

---

## 2. Screens in slice

### 2.1 SplashScreen — screens/splash/screens/splash_screen.dart:13-20, controller splash_controller.dart
- White Scaffold, `AppImages.appLogo` (kh_logo.svg) width 248. No loader, no version, no error/offline state. Logic above.

### 2.2 LoginPage — screens/login/login_page.dart, login_controller.dart
- Uses **Vazir** font via local Theme override (login_page.dart:31-34); the rest of the app is IranYekan. Mostly `uicat` components (`UICatTextField`, `UICatButton`, `UICatDialog`).
- Layout: centered, 24 px horizontal padding, scrollable. Logo (margin 45 h) → 32 → title "ورود" (`fa24Semibold`, secondary1000) → 32 → field "نام کاربری یا کد پرسنلی" hint "برای مثال ۱۷۶۵", clear button, LTR text aligned end → 24 → field "رمز عبور" obscured, LTR → 32 → `UICatButton` "ورود" xlarge fill full-width with loading → 12 → text button "فراموشی رمز عبور" with arrowLeft2 → optional "تغییر سرور".
- Validation (login_controller.dart:105-116): empty only. Errors "نام کاربری یا کد پرسنلی خود را وارد کنید." / "رمز عبور خود را وارد کنید."; cleared on typing. No length/format check. No show-password toggle. No "remember me" (token persists anyway). Fields disabled while loading.
- States: loading (button spinner + fields disabled), 403 sheet, silent failure for other errors (see 1).

### 2.3 HomeScreen — screens/home/screens/home_screen.dart, controller/home_controller.dart
- AppBar: `CustomAppBar(title: intl.inspection, showHelpButton, showProfileButton)` → title becomes Khodro45 logo (110 w) because title == `intl.inspection` (custom_appbar.dart:86-92). Leading: `Icomoon.profileCircle` 32 → Profile. Action: headphone 32 → support sheet.
- Before session loads: full-screen `Loading.large()` (Lottie loading.json 250 w).
- Custom tab bar (Row, not TabBar): `TabBarItem` (home/widget/tab_bar_items.dart): label 12 px, count pill (r16, primary if selected else black_80 #52575C, white 12 px), 2 px underline primary when selected, padding 12/12. Tabs are left-packed in a Row, not full-width, not scrollable.
  - 0 "بارگذاری مدارک" (only supervisors) count `documentCompletion`
  - 1 "کارشناسی" count = ready + inProgress
  - 2 "تایید نهایی" (only supervisors) count `finalApproval`
- Non-supervisor: selected index forced to 1 (home_controller.dart:45-47).
- Body: `IndexedStack` of three `TabBarScreen`s (hidden tabs replaced by SizedBox).
- **Badge polling every 10 s** (home_controller.dart:11, 34) calling `TabBadgeDataSource.getTabCount()` (see §4 for cost). Also refreshed on tab tap and after returning from detail screens. Errors swallowed (`debugPrint`).
- Note `/inspection/count/` datasource (`AuctionCountRemoteDataSource`, tab_data_source.dart) exists but is **not used**; repository is wired to `TabBadgeDataSource` (tab_count_repository.dart:6).

### 2.4 TabBarScreen (list per tab) — screens/cars/screens/tab_bar_screens.dart, controller/tab_bar_controller.dart
- Background `#F6F6F6` hardcoded. Column: [status filter bar (tab 1 only)] → 16 → search row → 16 → list → load-more spinner.
- **StatusFilterBar** (cars/widget/status_filter_bar.dart): segmented control, container `black_5` #F9F9FA r12 border dividerColor #D9DADD padding 4. Two pills (r10, AnimatedContainer 150 ms): "در انتظار کارشناسی" (orange #FF793F on cream #FFF5E7, schedule icon) and "در حال کارشناسی" (blue #1098F7 on #E7F5FE, autorenew icon), each with count badge (circle 22, or rounded if >99). Client-side filter only (tab_bar_controller.dart:41-66). Ready filter accepts status 1 or 3. Filter is ignored when a search term is set.
- **Search row**: `SearchInputWidegt` (white filled, dense, search icon, clear X) hint "شناسه قرار ملاقات یا کد پذیرش" + separate `CupertinoButton` "جستجو" (primary, r5, 12 px w500). Search runs on submit or button only. Persian digits converted.
  - Search semantics differ by source: back-office lists pass `search=` to server and then **re-filter client-side** by id / appointment / slug / owner; if nothing kept and query is numeric → `GET back_office/appointments/{id}/` single lookup with status + branch-title guard (back_office_appointments_api.dart:89-163). Final-approval tab also filters client-side again (tab_bar_controller.dart:56-64). Expert ready uses `/inspection/?status=ready&search=`.
- **List**: `SmartRefresher` pull-to-refresh with `AppConstant.classicHeader` texts "برای بروزرسانی پایین بکشید" / "رها کنید" / "در حال بروزرسانی" / "بروزرسانی موفقیت آمیز بود" (core/utils/constant.dart:32-42). ListView padding 11 h. Infinite scroll at exact max extent; page size 15 (`BackOfficeAppointmentsApi.pageSize`) or 10 for onsite (tab_bar_controller.dart:25-34). Merged tab fetches ready + inProgress in parallel and sorts by `createdAt` desc; final approval and doc tabs sort by id desc.
- **States**: initial/loading → opaque `#F6F6F6` overlay with `Loading.large()` (Lottie) covering list; empty / error / filter-empty → same `EmptyStateWidget`: Lottie `Search.json` 200, "موردی یافت نشد!" 18 bold, button "بازخوانی صفحه" with refresh icon (tab_bar_screens.dart:192-198). **Error and empty look identical** (no error message). Load-more: `CupertinoActivityIndicator`. Tab switch cancels in-flight fetch (CancelToken + generation counter).
- **Tap card** (`onTapInspection`, tab_bar_controller.dart:260-292): finalApproval → FinalApprovalScreen (supervisor only). Else if `status == 1` → `PATCH /inspection/{slug}/start_inspection/`; if false, returns silently. Then `SegmentScreen(carName, slug)`. On return: reset search, reload, refresh badges.

### 2.5 CardItem — screens/cars/widget/card_item.dart
- Container `AppConstant.decoration` (white, r8, border #F9F9FA, shadow black 4% blur 4 y2), outer padding 8 v, inner 8 h / 12 top / 8 bottom.
- Rows top→bottom:
  1. `StatusChip` (optional, start-aligned): pill r20, 10×5 padding, 12 px icon + 11 px w600 label. Labels: status 9 → "پایان کارشناسی"; 3 → statusText or "ورود به شعبه"; else statusText or "در حال کارشناسی" / "در انتظار کارشناسی". Colors: ready/entrance cream #FFF5E7 + orange; inProgress #E7F5FE + blue; inspected #F1F9EC + green #62B237 (status_chip.dart).
  2. Logo 32×32 (`Image.network`, fallback segment_logo.svg) + "year | model | brand" RichText LTR, 16 px headlineSmall (w700), separators black_40, ellipsis 1 line.
  3. Appointment icon + `appointment` id (14 px black_80) … Iranian plate graphic 130 w (`plate.svg` / `plate_empty.svg`) with parsed plate text. Plate format expected `"87-پ-324-23ایران"`; letter `v` renders wheelchair icon. Shows "پلاک ثبت نشده" if missing or < 8 chars.
  4. Branch (location icon + branch title) or "در محل" if onsite address.
  5. Profile icon + owner name (or "کاربر") … `createdAt` as Jalali "MM/DD - HH:mm" (no year).
  6. Action row (spacing 16): primary `InspectonButton`; onsite location button 48×48 outlined (opens visit sheet); "گزارش کارشناسی" outlined button (QC report).
- Button visibility rule (tab_bar_screens.dart:118-127): finalApproval → always; documentCompletion → show iff `editable_by_supervisor`, hide if `editable_by_inspector`; other tabs → hide if `editable_by_supervisor`, else show. QC button only on documentCompletion tab. Status chip on all three tabs.
- `InspectonButton` (core/widget/inspection_button.dart): r8, padding 12, 14 px. status 1 or title "بارگذاری مدارک" → filled primary white text; otherwise outlined primary. Label: title ?? (status 5 → "ادامه کارشناسی" else "شروع کارشناسی"). Appium ids via `Semantics(label: buttonId)` e.g. `startInspection1`, `continueInspection2`.
- Final approval button label "تایید نهایی" (FinalApprovalCopy.tabTitle).

### 2.6 VisitDetailsBottomSheet (onsite) — screens/cars/widget/visit_details_bottom_sheet.dart
- `Get.bottomSheet`, white, top radius 16, scroll-controlled, padding 24/24/24/32. Uses its own hardcoded palette (lines 37-43): title #3F6579, label #5D6C7E, icon #8E9098, value #52575C, tonal #F3F5F7, routing bg #ECEDEE, routing text #1B1F24.
- Header "جزییات مراجعه" (fa18Bold) + close 30×30 tonal r8.
- "نام مشتری" row: user icon, label 16 medium, name 18 medium, call square button (tel: `phone`) if phone.
- Divider. "تاریخ مراجعه" row: **uses user icon (vuesaxUser) instead of calendar** (line 211), remaining-time `UICatTag` small: <24 h → "{n} ساعت" red/dismiss; else "{n} روز" warning; hidden if past. Value: Jalali "weekday HH:mm - MM/DD".
- Divider. "آدرس محل": static `FlutterMap` 170 h r8, zoom 16, non-interactive, Map.ir tiles, marker; overlay button "مسیریابی" (Google Maps directions URL, external). Address text 18 regular.
- Primary button "شروع کارشناسی" (always this label, even if in progress) → pops and calls `onTapInspection`.
- Fields: owner, phone, reservation_date, reservation_time, latitude, longitude, address.

### 2.7 QcReportPage — screens/qc_report/qc_report_page.dart
- `CustomAppBar('گزارش کارشناسی')` + `InAppWebView` of backoffice QC report. Injects token into web `localStorage['ls.Authorization']` on load start. Spoofed 2012 Android 4.0 user agent (line 51-52). Spinner until load stop. No error/offline state, no reload.

### 2.8 ProfileScreen — screens/profile/screens/profile_screen.dart, controller/profile_controller.dart
- Background #F6F6F6. **Custom app bar** (not CustomAppBar): white, shadow alpha 10, back arrowRight2 24, centered "حساب کاربری" 16 bold, headphone 32 (support sheet).
- Loading: full shimmer skeleton (header, stats, 2 menu rows, logout row).
- Content (cards are white full-bleed, no radius, shadow alpha 20 blur 8):
  1. Header: 80 avatar circle (#F9FAFB, border secondary100, generic profile icon, no photo), name "first last" 16 bold, "شماره پرسنلی :‌ {username}", rating `UICatTag` (uicat package) with star: ≤2 red, ≤4 warning, else success.
  2. Stats card: header row (inspector icon) "کارشناسی‌های من" + current Jalali month/year, chevron → MyInspectionScreen. Grid 2×2: "تعداد کارشناسی" (`inspected_count.total`), "کارشناسی امروز" (`today`), "کارشناسی مجدد" count + (percent%) in red #F03A47, "میانگین زمان" (`average_inspection_time` secs → H:MM:SS / MM:SS) green <20 min, orange #FF9533 <30 min, red otherwise.
  3. Menu "شعبه‌های من" → BranchScreen.
  4. Menu "به روزرسانی نرم‌افزار" subtitle "نسخه فعلی {version}" — **uses location icon** (profile_screen.dart:577) → update sheet.
- Bottom bar: "خروج از حساب کاربری" red with logout icon → `LogOutDialog` sheet: "آیا اطمینان دارید؟" (primary 18), "آیا می خواهید از حساب کاربری خود خارج شوید؟", buttons "بله" (PrimaryButton) / "فعلا نه" (outlined black_80).
- Update sheet: title "به روزرسانی نرم‌افزار" #3F6579 18 w700, "نسخه‌ی جدید از برنامه دردسترس است.", optional "تغییرات نسخه جدید" bullet list (split by `\n`), Lottie `loading_update.json` 161, buttons "به روزرسانی" (primary) / "انصراف".
- Error state: none (catch → isLoading false, shows empty content).
- Profile load also triggers 2 release-app calls each time screen opens.

### 2.9 BranchScreen — screens/profile/widget/branch_screen.dart
- Same custom app bar ("شعبه‌های من"). Empty: Lottie Location.json 280 + "شعبه فعالی برای شما وجود ندارد.". List: white panel, title 16 bold black_80, address 14 secondary700, UICatDivider separators. Data from profile `branch[]` (title, address).

### 2.10 MyInspectionScreen — screens/profile/my_inspection/*
- Custom app bar "کارشناسی‌های من"; same search row; header row "{Jalali month} {year}" … "{count} مورد".
- Loading: `ShimmerHorizontalLoading(count 4, no button)`. Empty: Lottie Search 200 + "خودرو کارشناسی شده ندارید.". **Error: blank** (SizedBox).
- Card: stopwatch + duration (green <20 min, warning <30, red) … `UICatTag` large: "لغو کارشناسی" (secondary) if `is_canceled`, "کارشناسی مجدد" (dismiss) if `is_re_inspected`, else "کارشناسی شده" (success); dashed divider #D9DFE7; logo + year|model|brand; appointment id; "گزارش کارشناسی" outlined button. No plate, no owner, no date.
- `GET /inspection/my-inspections/?search=&limit=10&offset=` (my_inspection_datasource.dart). Search sent raw (no digit conversion, unlike other lists).

### 2.11 TermAndConditionScreen ("راهنما") — screens/profile/widget/term_and_condition_screen.dart
- Static help text explaining stats: "تعداد کارشناسی" (appointments reviewed since start of month excluding re-inspections), "کارشناسی امروز", "کارشناسی مجدد", "میانگین زمان" (incl. re-inspections, since start of month), "نکات ضبط ویدیو" (video 10–30 s, use HDR). Button "متوجه شدم" (UICat tinted secondary xlarge). Close icon is `Icomoon.bottomSheet`.
- Note: profile label "تعداد کارشناسی" is monthly per this help text, but model field is `inspected_count.total` [Inference: naming ambiguity; check backend].

---

## 3. Design system as implemented

### Two color systems coexist
**A. `AppColors` (core/utils/color.dart) — legacy, used by most custom widgets**
- `primaryColor` **#24536C** (teal-navy, brand). But `primaryColor_50..500` are **orange/cream**: #FFFAF3, #FFF5E7, #FFEBCE, #FFD69D, #FFC26C, #FFAD3B (color.dart:7-12). Misleading names; StatusChip "ready" uses `primaryColor_100` cream.
- Greys: greyColor #424653 (=greyLight=greyColor_0), greyDark #686B75, _15 #D9DADD, _25 #B3B5BA, _50 #E9EDF5, _100 #8E9098, _200 #AEC1DA, _300 #92A7C1, _400 #7B93AF, _500 #6783A0, _600 #8E9098 (dup of _100), _700 #445A74, _800 #30445B, _850 #6883A0, _900 #E5E5E5, _950 #1C2631; grey2 #737373, grey3 #D4D4D4, grey4 #F5F5F5, grey5 #A3A3A3.
- Blue #1098F7 (+ _5 #F3FAFF, _10 #E7F5FE, _20 #CFEAFD, _40 #9FD6FC, _60 #70C1FA, _80 #40ADF9).
- Green #71C544, greenDark #62B237, _50 #F8FCF6, _100 #F1F9EC, _200 #E3F3DA, _300 #51D8AF (odd teal), _400 #C6E8B4, _500=_600=_700 #AADC8F, green2 #06C78D, green3 #03C78D.
- Black #25282B, black_5 #F9F9FA, _10 #E8E8E8, _20 #DBDDE0, _40 #CACCCF, _60 #A0A4A8, _80 #52575C, black2 #262626, black3 #525252, black4 #ACACAC.
- backgroundColor **#F6F6F6** (also hardcoded `0xfff6f6f6` in TabBarScreen, Profile, Branch, MyInspection).
- Red #F03A47, red2 #FF4869, red3 #FFEDF0; borderColor #ADADAD; orange #FF793F; divider #D9DADD; shimmer base #E8E8E8 / highlight #F5F5F5; replacedDefect #FEBEC5; gradients green #40FFAF→#00B86B, red #FF347D→#FF2B2B.

**B. `UICatColors` theme extension (core/utils/theme.dart:21-113) — from in-house `uicat` package (git hamgit.ir/Khodro45/applications/flutter-core)**, accessed via `context.colors.*`. Full 50–1000 scales:
- primary 50 #E9EEF0, 100 #BBCAD1, 200 #9AB0BB, 300 #6C8C9D, 400 #507589, **500 #24536C**, 600 #1A3B4D, 700 #1A3B4D (dup of 600), 800 #142E3B, 900 #0F232D, 1000 #0A171D.
- secondary (cool slate) 50 #F3F5F7, 100 #D9DFE7, 200 #C6D0DC, 300 #ACBACB, 400 #9CADC1, 500 #8398B2, 600 #778AA2, 700 #5D6C7E, 800 #485462, 900 #37404B, 1000 #252B32.
- neutral 50 #F1F1F1 … 500 #767676 … 1000 #1A1A1A. common 50 #FFFFFF, 100 #FEFEFE, 200 #F9FAFB, 300 #F3F5F7, 1000 #000000.
- success 500 #1BA75E (50 #E8F6EF … 1000 #062313); info 500 #0A9EC2; warning 500 #FFC107; error 500 #EF3B2E. Opacity16 tints: secondary #EBEFF3, neutral #E9E9E9, success #EDF9F3, warning #FFF5D7, error #FEF0EF, info #EBF7FA, disable #7D8996.
- Newer screens (login, profile, visit sheet, support sheet) use UICat; older ones (cards, tabs, buttons) use AppColors. Two success greens (#1BA75E vs #71C544/#62B237), two reds (#EF3B2E vs #F03A47 vs #FF4869), two blues (#0A9EC2 vs #1098F7).

### Material ThemeData (theme.dart:18-181)
- `colorScheme`: primary #24536C; **secondary = blackColor #25282B**; onPrimary = black (wrong contrast on teal); error #F03A47; background #E5E5E5; surface white. `secondaryContainer` not set (used by offline banner → Material default).
- Inputs: OutlineInputBorder r4, enabled border #ADADAD, default #424653; label 14 w400; hint 12 w500 black 50%.
- ElevatedButton: bg primary, r8. OutlinedButton: r8, side primary.
- TabBarTheme: label/unselected both 14 bold black_80 (no visual difference).
- TextTheme (all IranYekan): bodyLarge 40 w600 **italic** (bizarre), titleMedium 16 w600 black_80, titleSmall 16 w400, headlineSmall 16 w700, titleLarge 16 w900. Nearly every widget uses `titleMedium.copyWith(fontSize: X, ...)` → typography is ad hoc.
- `colorMap` white-alpha swatch defined, unused.

### Typography
- Fonts: **IranYekan FaNum** 400/500/700/800 (app default, Persian digits); **Vazirmatn** 300–700 (login screen + uicat `fa14Regular`/`fa18Bold` text extensions — [Inference] uicat ext probably sets vazir; not verified); icon fonts `icomoon_inspection`, `icomoon` (uicat).
- Theme asks for w600 and w900 which IranYekan lacks (synthesized/nearest weight).
- Sizes in use: 10 (debug banner), 11 (chips), 12 (tab labels, search button, field labels), 13 (filter pills), 14 (most body/card text, buttons), 16 (card title, menu titles, buttons), 18 (app bar title, sheet titles, empty-state title, stat values, timer), 24 (login title), 110 (legacy 500 page).

### Radii / spacing / shadows / elevation
- Radii: 4 (inputs, customize chip), 5 (search button), 6 (UICatTag), 8 (cards, buttons, chips, close buttons, map), 10–12 (filter bar), 16 (bottom sheets top, preview carousel tiles, tab count pill), 20 (status chip), 40/50 (avatar, FAB, group buttons). Profile cards have **no radius**.
- Spacing: list gutter **11 px** (odd), card inner 8/12, sheet padding 16 or 24, section gaps 12/16/24/32. No spacing tokens.
- Shadows: `AppConstant.decoration` black 4% blur 4 y2; `defultBoxShadow` black 7% blur 20 y4; bottom nav black 7% blur 10 y-3; profile cards black alpha 20 blur 8; app bar elevation 2 shadow 7%.

### Component styles
- App bar (`CustomAppBar`, core/widget/custom_appbar.dart): white, centered title 18 bold, elevation 2, iOS back chevron `arrow_back_ios_rounded` 18, height kToolbarHeight. Title replaced by logo when title equals "اپلیکیشن کارشناسی" or "پروفایل" (string comparison). Car screens show "year | model | make" title. Profile/Branch/MyInspection use a **different hand-built app bar** (arrowRight2 icon, 16 bold title).
- Buttons: `PrimaryButton` (CupertinoButton, primary bg, padding 12/16, title 18 default, optional icon rotated 180°, LTR directionality by default, spinner when loading); `SecondryButton` (OutlinedButton but bg primary unless white); `PrimaryDisableButton` (primary 50% + white 30% text); `SecondryDisableButton` (#6883A0); `InspectonButton`; `UICatButton` (login/help). Destructive red CupertinoButton in delete dialog. Three different press models (Cupertino opacity, Material ripple, GestureDetector none).
- Inputs: `SearchInputWidegt`, `CustomTextField` (label 12 primary above field), `CustomInputEmail` (email validation, unused-looking), `UICatTextField` (login), CategoryWidget search field r8.
- Bottom sheets: mostly white top r16 with title row + 30×30 tonal close (r8, grey #E5E5E5 or #F3F5F7) on the opposite side; `ConfirmationBottomSheet` ("تایید" / "فعلا نه", buttons fixed 174×40, cancel border #847373 brownish); `UICatDialog().showBottomSheet` (title, description, content, primaryButton) for newer flows; chooser sheet `chooseTheImage` (bg #F6F6F6, rows "گرفتن عکس" / "انتخاب عکس از گالری" / "گرفتن ویدیو" / "انتخاب ویدیو از گالری").
- Snackbars: `UICatSnackBar` top position (success/warning/error).
- Icons: mix of icomoon_inspection glyphs, uicat `Icomoon` glyphs, Material icons (bug_report, close_rounded, schedule_rounded, autorenew_rounded, storefront_outlined, arrow_back_ios_rounded, add_photo_alternate_outlined, play_arrow), Cupertino icons (search, photo_camera), and SVG assets (car parts, camera guides, plate, location, profile_icon, appointment, inspector, stopwatch, calling, mapmarker...). ≥4 icon families.
- Animations: Lottie assets (assets/anims): loading.json (global loader 70/100/250), loading_update.json, Wifi.json, call.json, Blocked.json, emptyState.json, check_list_lottie.json, upload_loading.json, Location.json, Search.json (used for all empty states), tap.json, rotate.json, buttonLoading.json. Implicit: StatusFilterBar 150 ms; UploadProgressWidget tween 300 ms; image fade 300 ms; shimmer skeletons (profile, my inspections). **No page transitions** (noTransition).

### Reusable widgets in core/widget (what each does)
| file | widget | role |
|---|---|---|
| app_circle_image.dart | `AppImagesWidget.circle` | cached circular avatar; null → bmw.png placeholder; prints to console |
| bottom_sheet_view.dart | `PreviewBottomSheet` (+ local `IconButton` that shadows Material's) | full-screen media viewer: big image/video, carousel (viewport 0.5, dotted r16 tiles with index badge and title), retry upload (red rotateLeft), "شخصی سازی" (edit → PaintImageScreen), add image, delete (AcceptanceDialog), close; video player with scrubber |
| car_info_widget.dart | `CarInfoWidget` | title + "شماره پذیرش : {carNumber}" row |
| car_status.dart | `CarStatus` | 42 h white bordered status label (1/5/9 text) |
| category_widget.dart | `CategoryWidget` | filter expansion panel + GroupButton chips (radio, r50, selected #FFFAF3 bg primary border) + optional search "نام آیتم" |
| confirmation_bottom_sheet.dart | `ConfirmationBottomSheet`, `showConfirmationBottomSheet` | generic confirm/not-now sheet |
| custom_appbar.dart | `CustomAppBar`, `onTapSupport` | app bar + support sheet |
| custom_bottom_navigation.dart | `CustomBottomNavigatio` | sticky bottom action bar: optional outlined secondary action, 30-min countdown timer (red 18) + primary "end" button; disabled → grey #92A7C1 |
| custom_debug_banner.dart | `CustomDebugBanner` | env/version red strip |
| custom_email_input.dart | `CustomInputEmail` | email field w/ validator "آدرس ایمیل خود را به درستی وارد نمایید" (hint info@carsparency.com: template leftover) |
| custom_error_widget.dart | `CustomErrorWidget` | empty SizedBox (stub) |
| custom_textFiled.dart | `CustomTextField` | labeled TextField |
| empty_state.dart | `EmptyStateWidget` | Lottie Search + title + optional button; wraps itself in `Expanded` (fragile) |
| image.dart | `ImageWidget` / `image()` | universal renderer: asset/network/SVG/Lottie/File/bytes/IconData by extension/mime; network falls back to SVG |
| image_loading_service.dart | `ImageLoadingService` | progressive image: blurred low-res `/60/80/fit/` placeholder then full; default.png on error |
| inspection_button.dart | `InspectonButton` | card CTA (see 2.5) |
| internt_connection_bottom_sheet.dart | `InternetConnectionBottomSheet` | legacy offline sheet |
| loading_service.dart | `Loading` (.small 70, .medium 100, .large 250) | Lottie loader |
| optioanl_image_widget.dart | `OptionalImageWidget` | orange "اختیاری" ribbon |
| outline_button.dart | `SecondryButton` | outlined/primary button |
| primary_button.dart / primary_button_disable.dart / secondry_disable_button.dart | buttons | see above |
| safe_cached_network_svg_image.dart | `SafeCachedNetworkSVGImage` | web-safe network SVG |
| search_empty_state.dart | `SearchEmptyState` | Lottie Search + "موردی یافت نشد!" |
| search_input.dart | `SearchInputWidegt` | search field (Appium id `searchTextField`) |
| shimmer_loading_horizontal.dart | `ShimmerHorizontalLoading` | card skeleton (uses **blue #1098F7** blocks inside shimmer) |
| svg_service.dart | `SvgServices` | IconData or SVG asset, default size 32 |
| timer_widget.dart | `InspectionTimerWidget` | mm:ss countdown from `inspection_start_at` + 30 min, stops at 00:00 (no overtime display) |
| uicat_chips.dart | `UICatChips` | local chip (fill/outline, default/deactive/selected) |
| uicat_tag.dart | `UICatTag` (local) | tag; **name clashes** with uicat package `UICatTag` used in profile/my_inspection (different API: `isFilled`, `UICatTagType.secondary`) |
| upload_progress_widget.dart | `UploadProgressWidget` | circular % progress; creates a new AnimationController every rebuild with `vsync: Navigator.of(context)` (leak) |
| yes_no_dialog.dart | `AcceptanceDialog` | delete confirm "حذف عکس/ویدیو" / "آیا از حذف این مورد اطمینان دارید؟" / "حذف تصویر / ویدئو" (red) / "فعلا نه" |

### UI defects / inconsistencies that date the look
1. Two parallel color systems (AppColors vs UICat), duplicate/contradictory tokens, `primaryColor_xxx` = orange.
2. Hardcoded hex everywhere (#F6F6F6 ×5, #3F6579, #847373, #FF9533, #222222, #FF4869, visit-sheet palette).
3. Typography by `copyWith` on `titleMedium`; theme text styles nonsensical (bodyLarge 40 italic, titleLarge w900).
4. Two fonts (Vazir on login, IranYekan elsewhere).
5. Three app-bar implementations; back icon differs (Material iOS chevron vs Icomoon arrowRight2).
6. Tab bar is a left-packed Row of 12 px labels, not full width; badge on unselected tab is dark grey.
7. List gutter 11 px; cards have tiny 8 px inner padding; card CTA heights vary (12 padding vs 48 box).
8. Empty and error states identical ("موردی یافت نشد!") with a search Lottie for every case; MyInspection error is blank; QC webview and Profile have no error state.
9. Loading is a full-screen Lottie overlay that hides the list on every tab switch/refresh (no skeleton), while MyInspection uses shimmer → inconsistent.
10. Wrong icons: visit date uses user icon; app-version menu uses location icon.
11. Debug banner covers top of screen on staging; double offline banner.
12. Search requires separate tiny "جستجو" button; behavior differs per tab.
13. Date on cards lacks year; brand/model/year order reversed ("year | model | brand") via LTR RichText with LRM marks.
14. No dark mode, no page transitions, no haptic feedback, no pressed states on many GestureDetector rows.
15. Error/validation toasts disabled → silent failures.
16. Profile cards edge-to-edge with no radius vs rounded list cards.
17. Material `colorScheme.onPrimary` black on teal; `secondary` black.

---

## 4. API endpoints touched by this slice

Base = `{env}/api/v2` unless noted. Auth header `Token <token>` (DioClient).

| # | Method | Path | Purpose / key fields | File |
|---|---|---|---|---|
| 1 | POST | `/api/v1/auth/login/` | body `{username, password}` → `{token}`; 403 = disabled account | login_datasource.dart:18 |
| 2 | GET | `/release-app/check/copilot/{device}/{version}/` | no auth → `{should_update, force, download_url}` | release_app_api.dart:27 |
| 3 | GET | `/release-app/retrieve/copilot/{device}/` | no auth → `{latest, supported, latest_version, supported_version, update_url, description}` | release_app_api.dart:42 |
| 4 | POST | `/release-app/report/` | `{app, device, device_id, device_model, version}` | release_app_api.dart:59 |
| 5 | GET | `/inspector/profile/` | `{id, first_name, last_name, email, mobile, username, rate, branch[]{id/pk/branch_id, title/name, address, branch_type{id,title}} (or "branches"), inspected_count{total,today}, average_inspection_time (sec, double), re_inspection{count,percent}, is_supervisor / is_branch_supervisor}` | profile_data_source.dart:15, profile_model.dart |
| 6 | GET | `/back_office/roles/info/` | `{title}`; supervisor if title contains "supervisor" or "سرپرست" | roles_info_api.dart:10-37 |
| 7 | GET | `/back_office/appointments/?page_size=&page=&status=..&ordering=-id&branch=..&search=` | appointment lists; item fields used: `id, status, last_inspection_slug / qc_url / inspection.slug, reservation.branch{id,title/name}, name/owner, logo(str or {url}), trimyear{year, trim{title, model{title, brand{title}}}} or model_full_name, created_time/created_at`; response `{count, next, previous, results}` or list | back_office_appointments_api.dart, appointment_list_mapper.dart |
| 8 | GET | `/back_office/appointments/{id}/` | numeric search fallback | back_office_appointments_api.dart:137 |
| 9 | GET | `/inspection/{slug}/` | **per-item** plate enrichment → `car_detail.plate_number` (N+1 calls per page) | back_office_appointments_api.dart:165-188 |
| 10 | GET | `/inspection/?limit=100&offset=&status=ready&search=` | expert "ready" list, paged client-side up to 500 items, filtered by branch title | inspection_appointments_api.dart:125-177 |
| 11 | GET | `/inspection/onsite-list/?limit&offset&status=ready|in_progress&search` | onsite inspector lists + counts (`limit=1` for counts); items add `address, latitude, longitude, phone, reservation_date, reservation_time, onsite_inspector_id` | tab_bar_datasource.dart:72, tab_badge_data_source.dart:60 |
| 12 | PATCH | `/inspection/{slug}/start_inspection/` | start before entering SegmentScreen (status 1 only) | segments/.../car_segments_datasource.dart:30 (called from tab_bar_controller.dart:284) |
| 13 | GET | `/inspection/my-inspections/?search&limit=10&offset` | history: `duration (sec), is_re_inspected, is_canceled, appointment, slug, logo, model_full_name{brand,model,trim,year}` | my_inspection_datasource.dart:18 |
| 14 | GET | `/inspection/count/` | `{ready_count, in_progress_count, inspected_count}` — **dead** (not wired) | tab_data_source.dart:15 |
| 15 | WEB | `backoffice.{env}.technical45.com/inspection-qc-report/{slug}` | QC report webview, token via localStorage `ls.Authorization` | qc_report_page.dart |
| 16 | GET | `inspection.{env}.technical45.com/static/car_detail_k45.json` | car-detail form config (constant only in slice) | constant.dart:4 |
| 17 | TILE | `map.ir/shiveh/xyz/...?x-api-key=` | onsite visit map | map_ir_config.dart |
| — | — | `https://www.google.com/maps/dir/?api=1&destination=lat,lng` | external routing | visit_details_bottom_sheet.dart:123 |
- Car list legacy fields still parsed in `CarModelResults` (car_model.dart): `inspection_start_at, inspection_end_at, car_detail{…40 fields: mileage, vin_number, plate_number, fuel_type, color, warranty…}, templates_data{total, completed, percent}, car_detail_meta{mileage_range}, car_number, editable_by_inspector, editable_by_supervisor`. `templates_data` (progress %) is parsed but **not shown** on cards. [Inference] a redesign could show inspection progress.
- `cars/data/model/tab_model.dart` (`AllTabModel`, seller, sellrequest) is dead code. `lib/data/**` is entirely commented-out template code (example.com, email login).

### Request volume concern (badge poll every 10 s; home_controller.dart:34)
- Supervisor per poll: doc-completion count + ready count + in-progress count + final-approval count → each a back_office list call with `page_size=1` + 1 plate-enrichment `/inspection/{slug}/` call → ~8 requests / 10 s, plus branch-resolver probes if branch ids < branch titles (resolver re-probes up to `titles × 2 queries × statuses` + a `page_size=200` call; back_office_branch_resolver.dart:23-135).
- Expert per poll: ready count loops `/inspection/?status=ready&limit=100` up to 5 pages + in-progress back_office count. [Inference] heavy server/battery load; also runs while user is on SegmentScreen since Home stays mounted under `Get.to`.

---

## 5. Business rules, statuses, roles, branch resolution

### Two status vocabularies (core/utils/constant.dart:60-110)
- `InspectionStatus` (inspection object, `CarModelResults.status`): **1 ready**, **3 entrance**, **5 inProgress**, **9 inspected**. Text map `statusText()` (utils.dart:236-247): 1 "در انتظار کارشناسی", 5 "در حال کارشناسی", 9 "کارشناسی شده", default "در حال کارشناسی".
- `AppointmentStatus` (back-office appointment): **3 branchEntrance "ورود به شعبه"**, **4 branchInspectionInProgress "در حال کارشناسی"**, **5 inspectionEnded "پایان کارشناسی"**, 23 notPublished "انتشار نشده", 62 auction. `documentCompletionStatuses = [3, 4]`.
- Mapper appointment→inspection status (appointment_list_mapper.dart:58-69): 3→3, 4→5, 5→9, **anything else→9**. Then `_normalizeForInspectionTabs` overwrites to 1 (ready tab) or 5 (in-progress tab) and clears statusText.
- **Collision risk**: the integer 5 means "in progress" for inspections and "inspection ended" for appointments; 3 means entrance in both. A redesign/API refactor must keep the mapping layer.
- `AppointmentTransition` slugs for `POST back_office/appointments/{pk}/change_status/`: `re_inspection` (برگشت به کارشناسی), `not_auctioned` (عدم انتشار), `scheduled_auction` (زمان‌بندی / شروع مزایده). Used by final approval (outside slice).
- `TabBarName { inProgress, completedDocument, ready, inspection, finalApproval }`; `inspection` = UI-only merge of ready + inProgress.

### Tabs → data sources
| Tab | Who | Source |
|---|---|---|
| بارگذاری مدارک (completedDocument) | supervisor only (`showDocumentCompletionTab = isSupervisor`) | back_office status 3 & 4 + branch ids |
| کارشناسی → در انتظار (ready) | supervisor: back_office status 3 + branch ids; expert: `/inspection/?status=ready` + client branch-title filter; onsite: onsite-list `ready` | inspection_appointments_api.dart |
| کارشناسی → در حال (inProgress) | branch users: back_office status 4 + branch ids; onsite: onsite-list `in_progress` | same |
| تایید نهایی (finalApproval) | supervisor only | back_office status 5 + branch ids |
- Comment (inspection_appointments_api.dart:9-18): experts are ACL-denied on back_office status 3 (count always 0), so ready uses `/inspection/`.

### Roles (core/service/inspector_session.dart)
- Loaded on Home init and lazily (`ensureLoaded`) by every datasource: parallel `GET /inspector/profile/` + `GET /back_office/roles/info/`.
- `isSupervisor` = profile `is_supervisor`/`is_branch_supervisor` if present, else role title contains "supervisor"/"سرپرست".
- `isOnsite` = any branch with `branch_type.id == 2`.
- `branchIds` from profile; `branchTitles` (trimmed, deduped).
- Failure → everything null → next call retries.
- Card CTA editability: `editable_by_inspector`, `editable_by_supervisor` (see 2.5).
- Final approval tap is blocked client-side for non-supervisors (tab_bar_controller.dart:268).

### Branch resolution (cars/data/datasource/back_office_branch_resolver.dart)
- Use profile branch ids if count ≥ titles count. Else probe back_office appointments per status (requested + 3 + 4) and per title query (full title, then text before `،`/`,`/`-`), page_size 15, then a page_size 200 sweep; collect `reservation.branch.id` whose title fuzzy-matches (normalize Arabic ي/ك → Persian ی/ک, collapse spaces, contains-either-way or shared comma segment). Results merged into session. Single in-flight future shared.
- Expert ready filter (inspection_appointments_api.dart:179-227): exact/suffix-segment match on branch title; **hardcoded preference: if any assigned branch is "شیخ بهایی", only that branch is shown** (comment: staging test accounts). [Inference] this is production code and would hide other branches for a real multi-branch inspector assigned to شیخ بهایی.
- Search results for back-office lists are also guarded by branch title match.
- [Inference] branch scoping is partly enforced client-side (UI layer), not only by server ACL; a redesign must not drop these filters without backend confirmation.

### Other rules
- Inspection timer: 30 min from `inspection_start_at` (custom_bottom_navigation.dart:74; timer_widget.dart).
- Media rules (core/service/app_media_picker.dart): images ≤ 20 MB, must be ~4:3 (±0.1) when `checkImageSize`; compressed to WebP q95; camera photos/videos saved to gallery album (`k45-...`); video 10–30 s (camera max 30 s), ≤ 100 MB. Upload hard cap 100 MB (DioClient).
- Plate format `NN-letter-NNN-NNایران`; `v` letter = disabled-person plate (wheelchair icon).
- Ratings: ≤2 red, ≤4 amber, >4 green. Avg time: <20 min green, <30 amber, ≥30 red.
- Support: phone 1485, ext ۲۸۰ (hardcoded in several places).
- AI confidence → tag type (extension.dart:192-198): ≤0.8 dismiss, ≤0.9 warning, else success.

---

## 6. Surprising / risky / must-preserve for a redesign

Must preserve (behavior a redesign could easily break):
1. **Offline queue + pending banner + non-dismissible offline sheet** (OfflineRetryInterceptor). Inspectors work in branches with weak network; uploads replay automatically. Queue is in-memory only (lost on kill): consider persisting.
2. **Lifecycle quiet window** (3 s) and the white status-bar strip hack (main.dart:235-243) fixing Android black bars after lock/unlock / camera return.
3. **Status mapping layer** (appointment 3/4/5 → inspection 3/5/9 → normalized 1/5) and client-side branch filters.
4. **Role gating**: supervisor-only tabs, onsite list, editable flags, final-approval guard.
5. **Start-inspection PATCH before entering SegmentScreen** only for status 1.
6. **Forced/optional update gate** on splash and install report after login (release-app API, app key `copilot`).
7. **401 → logout everywhere**; 403 login → disabled-account sheet with 1485 call.
8. Appium test ids via `Semantics(label:)` / `.id()` (`profileButton`, `backButton`, `callButton`, `completedDocumentTab`, `inspectionTab`, `finalApprovalTab`, `searchTextField`, `startInspectionN`, `continueInspectionN`, `QC Report {id}`, `button{Title}`, `car_image_N`, `edit_image`, `delete_image`, `close_image`, `add_image_2`, `camera`, `gallery`, `video`, `videoGallery`). Changing them breaks the automated test suite [Inference: Appium suite exists outside repo].
9. Debug affordances: `CustomDebugBanner` (env + version) and Talker FAB; dev server switcher. Keep for staging but move out of the content area.
10. Persian digits (FaNum font) + `toEnNumber` normalization on all inputs; RTL with explicit LTR islands (plate, car name, timer, numbers).
11. Image upload Sentry tracking keyed on `PATCH .../add_image/` and `.../add_defect/` paths.

Risky / surprising:
- Hardcoded third-party tokens and Sentry DSN in source; `Sentry debug=true` in prod; logs can be forced on in prod with `LOG_API`.
- Plain token in GetStorage; token injected into webview localStorage; QC webview uses a spoofed 2012 user agent.
- `errorToastsEnabled = false` silences all API and media-validation feedback, including wrong password.
- Badge polling every 10 s with N+1 requests and up to 5-page ready sweeps; continues under pushed screens.
- Hardcoded "شیخ بهایی" branch preference.
- Duplicate offline banner; dead datasources (`/inspection/count/`), dead models (tab_model.dart), dead `lib/data/**`, legacy dialogs in utils.dart; `AppConstant.classicHeader` is a static field evaluated with `navigatorKey.currentContext!` (crash risk if accessed before first frame); `AppConstant.noImage` points to a carexpert.com.au Porsche photo.
- `showFilePicker` ignores `allowMultiple` result (utils.dart:30-55) — multiple files collected but `action` never called.
- `UploadProgressWidget` leaks AnimationControllers; `PreviewBottomSheet` defines a local `IconButton` class shadowing Material's.
- Login silently returns on network error while `isLoading` resets; Sentry user set before auth success.
- `Splash` fixed 2.5 s delay after network check.
- Two widgets named `UICatTag` (local vs package) with different enums.
- Stats label "تعداد کارشناسی" described as monthly in help, field named `total`.

Redesign opportunities tied to data already available [Inference]:
- Show `templates_data.percent` progress and `inspection_start_at` elapsed time on in-progress cards.
- Separate error vs empty vs offline states; inline retry.
- Full-width segmented tabs with counts; persistent search with live filter; skeleton loaders instead of Lottie overlay.
- One token set (UICat scales) + named text styles; drop AppColors orange-named primaries.
