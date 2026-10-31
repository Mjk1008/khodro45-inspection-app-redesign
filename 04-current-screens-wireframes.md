# Current screens as text wireframes (no screenshots exist)

Derived from the source code, so layout, sizes and copy are accurate to the code but not seen on a device. RTL: the first item in a row is on the **right**. Colours are today's. Use these to understand what is being replaced; do **not** copy the look.

## Home: list card (one car)
```
┌───────────────────────────────────────────┐ white, radius 8, faint shadow, bg #F6F6F6
│ ( در انتظار کارشناسی )  pill 11px, cream  │
│ [logo 32]  1402 | 206 | پژو     16px bold │  "year | model | brand" (reversed order, LRM hack)
│ 📅 شناسه 123456            [ plate 130w ] │
│ 📍 شعبه ...                               │
│ 👤 نام مالک                  12/08 - 14:30│  no year
│ [ شروع کارشناسی ]  [📍]  [ گزارش کارشناسی ]│  buttons 12px padding, mixed heights
└───────────────────────────────────────────┘
Tabs above: Row of 12px labels, left-packed, count pill (dark grey if unselected), underline 2px
Filter (tab 2): segmented "در انتظار کارشناسی" (orange) | "در حال کارشناسی" (blue)
Search: field + separate tiny "جستجو" button
Empty = error = same Lottie + "موردی یافت نشد!" + "بازخوانی صفحه"
```

## Inspection hub (SegmentScreen)
```
AppBar: car name + help      bg #F6F6F6, padding 24
[ شماره پذیرش : 123456 ]  [status chip]
┌ [icon 56] مدارک                    ‹ ┐  supervisor only (hidden for inspector)
┌ [icon 56] اطلاعات خودرو            ‹ ┐
┌ [icon 56] جزئیات خودرو             ‹ ┐  supervisor only
┌ [icon 56] تصاویر + ویدیوها   ۶۵٪   ‹ ┐  only progress number on the page
┌ [icon 56] کارکرد                   ‹ ┐
┌──────────┐ ┌──────────┐   2-col grid, server segments (موتور، بدنه و لاستیک …)
│ icon     │ │ icon     │   ★ rate · ✓ n (components WITHOUT defect, incl. unopened) · ✗ n
│ title    │ │ title    │
└──────────┘ └──────────┘
[ لغو کارشناسی ] red card, supervisor only
Bottom bar: ⏱ 18:32 (red, 18px)        [ پایان کارشناسی ]
Empty/error: blank white
```

## Photos grid
```
AppBar: car name
chips: همه | بیرونی | داخلی | …
[ ===== عکاسی سریع ===== ]  full-width primary button
           ── یا ──
نمای جلو *                         (title 18)
┌────────┐ ┌────────┐     2 columns, 180px tall, dotted border, red when required-missing
│ ✎(32px)│ │  guide │     uploaded: thumbnail + 32px edit chip top-left
│ thumb  │ │  icon  │     uploading: scrim + % ring · failed: 28px retry icon
└────────┘ └────────┘
Bottom bar: ⏱  [ ثبت اطلاعات ]  (disabled until required done)
```

## Guided camera (landscape)
```
[flash]            ┌──────────────────────────┐          (empty 56px spacer)
                   │  viewfinder 4:3          │
[back]             │  white outline guide     │          [skip if can_skip]
                   │      ( shutter 88 )      │
                   │  pill: "نمای جلو | 3/20" │
                   └──────────────────────────┘
Video slots (360, engine-video): shutter red, 30 s countdown
Errors (size/ratio/duration): silent
```

## Segment: component and defect
```
chips (categories) + search
▼ بدنه و لاستیک   (3)            accordion cards, icon tile 40px, orange count badge
   [ عکاسی سریع نواقص ماشین ]    body segment only, opens the diagram (replaces this page)
   ▸ درب جلو راست (1)       🛡
        توضیحات [__________]     disabled once any defect is on
        ○ خط و خش   ●(red dot)  ✎    switch coloured by severity_color
        نکات [____]   [+ add media 40px dashed]   (3-col thumbs)
        شدت آسیب  ▢▢▢▢▢   مساحت آسیب  ▢▢▢▢▢    (after a photo; body & tire only)   [ تایید ]
[ ثبت اطلاعات ]  → validate; 4xx → same page in "incomplete only" mode
```

## Car diagram
```
AppBar: نواقص ماشین  ⓘ(24px)
top-down car (CustomPaint, full width, 6,952-line painter)
  19 panels tappable (min 32px), 2 mirrors, 4 wheels; markers 24px severity colour; replaced panels pink
legend: قابل انتخاب | غیرقابل انتخاب | شیشه
لیست اجزا: components not on the map (tap leaves the diagram)
tap panel → sheet: choose component → choose defect → TRACE screen (finger-draw on a single panel, grey bg)
   → photo sheet (photo required, annotation editor forced) → severity/area rails
```

## AI QC panel
```
AppBar: AI45 logo
analyzing: spinner (orange), "در حال بررسی تصاویر آپلود شده", 10 s countdown (+4×5 s)
results: "تصاویر نیازمند بررسی"
┌ [thumb 120×90]  chips: عدم تطابق تصویر | پلاک نمایان | وجود شخص در تصویر (advisory) ┐
│ [آپلود مجدد] [تایید است] [افزودن لوگوی پلاک]   buttons 36px, text 11px                │
bottom: [ ادامه ]  or  [ بررسی همه تصاویر الزامی است ] (looks enabled, does nothing)
failed: "بررسی تصاویر ناموفق بود" [تلاش مجدد] [ادامه بدون بررسی]
```

## Final approval (supervisor)
```
AppBar: تایید نهایی
مشکلات عکس‌های کارشناسی      [thumb 56] title                 [تایید] 36px → [تایید شد ✓]
مدارک                       [thumb 56] برگه سبز · reason · AI value · [ویرایش/آپلود مجدد] [ثبت]
تایید تریم و آپشن            question sheet auto-opens; mismatch card; «ثبت اصلاح روی کارشناسی»
sticky: [ انتشار ] orange (disabled until all acked)
        [ عدم انتشار ]  [ برگشت به کارشناسی ]   (look identical)
        "برای مشاهده قیمت به بک آفیس مراجعه کنید"
انتشار → no confirm → auction starts in 35 s
```

## Login
```
logo · «ورود» 24px · [نام کاربری یا کد پرسنلی] · [رمز عبور] (no show toggle)
[ ورود ] · فراموشی رمز عبور › · (staging) تغییر سرور
wrong password: no message
```
