# QA Review — ParkLog v1.2.0
**תאריך:** 2026-03-08
**גרסה נבדקת:** 1.2.0
**מבצע הסקירה:** מנהל צוות בדיקות
**סטטוס כולל:** ✅ מוכן לפרסום — עם שיפורים מומלצים

---

## 1. סיכום מנהלים

המערכת נמצאת ברמת בשלות טובה לפרויקט בגודל זה. ה-changelog של v1.2.0 מראה שהצוות תיקן בעיות קריטיות (שימור timestamp של ה-offline queue, focus trap במודאלים, XSS ב-commandcenter). הקוד נקי, מתועד ב-JSDoc, ועמיד ב-design system. רשימת הבדיקות ב-CLAUDE.md מקיפה ומפורטת אבל סובלת ממספר פערים מבניים שמפחיתים את שימושיותה.

---

## 2. ממצאים לפי קטגוריה

---

### 🔴 CRITICAL (2)

---

**🔴 C-01 — config.js חושף Apps Script URL אמיתי ב-repo**

📍 מיקום: `config.js` שורה 12
🔍 תיאור: `APPS_SCRIPT_URL` מכיל URL אמיתי וממשי המגיע ישירות ל-Apps Script Production:
```
https://script.google.com/macros/s/AKfycbzd96MPqGSA.../exec
```
ה-CLAUDE.md מציין ש"URL alone cannot bypass Apps Script validation" — זה נכון טכנית, אך יש לכך השלכות:

1. **Rate abuse**: כל מי שיש לו גישה ל-repo יכול לשלוח אלפי POST requests ולמלא את ה-Sheets בזבל — Apps Script אינו מגביל קצב בצורה עמידה.
2. **CORS allowlist**: ב-Code.gs שורה 24 מוגדר `ALLOWED_ORIGINS` אך `jsonResponse()` שורה 692 **אינה מחזירה CORS headers** — הכתוב ב-CLAUDE.md על "Apps Script validates Origin header" אינו מיושם בפועל בקוד.
3. **Enum enumeration**: ניתן להריץ `action=searchVehicle&placa=AA` עד `ZZZ` ולמפות לוחיות.

💥 השפעה: כל אחד עם גישה ל-repo יכול לקרוא ולכתוב נתונים ב-Google Sheet, ולהציף את המערכת.
✅ פתרון מוצע:
- הוסף `config.js` ל-`.gitignore` ותדאג שה-repo לא חושף אותו ציבורית.
- ממש את ה-CORS headers בפועל ב-`jsonResponse()` — כרגע האכיפה לא קיימת.
- שקול הוספת Bearer token פשוט בשדה header, שנשמר ב-Apps Script Properties ולא ב-frontend.

---

**🔴 C-02 — race condition בשמירה כפולה מהירה**

📍 מיקום: `vault-entry.js` שורות 172, 190–191
🔍 תיאור: `submitCooldown` נועד למנוע שמירה כפולה, אך יש חלון זמן:
```javascript
submitCooldown = true;           // נקבע כאן
submitBtn.disabled = true;       // נקבע כאן
// ... await DataStore.saveEntry(...)  ← מה קורה אם לוחצים ENTER פה?
```
`keydown` listener בשורה 121 בודק `!submitBtn.disabled`, אבל בין `handleSubmit()` שנקרא מה-click לבין ה-await, אירוע ENTER יכול לקרוא שוב ל-`handleSubmit()` לפני שה-`submitCooldown` נבדק — כי ה-event handler קורא ל-`handleSubmit()` ישירות ולא בודק `submitCooldown` בנפרד מ-`submitBtn.disabled`.

לעומת זאת, `handleSubmit()` בשורה 172 כן בודק:
```javascript
if (submitCooldown || submitBtn.disabled) return;
```
אך ב-test מהיר מאוד (< 1ms) — שניהם עשויים להיות false כשה-event השני נכנס.

💥 השפעה: ייתכן שתיים כניסות כפולות בשניות ספורות, יוצרות duplicate בשט.
✅ פתרון: קבע `submitCooldown = true` כבר בתחילת `handleSubmit()`, לפני ה-validation — כך כל קריאה שנייה נחסמת מיד.

---

### 🟡 WARNING (7)

---

**🟡 W-01 — Apps Script: לא מוגן מ-queuedAt injection**

📍 מיקום: `Code.gs` שורות 172–177
🔍 תיאור: `createEntry()` מאמין ל-`data.queuedAt` שנשלח מה-frontend ומשתמש בו כ-timestamp:
```javascript
if (data.queuedAt) {
  const parsed = new Date(data.queuedAt);
  now = isNaN(parsed.getTime()) ? new Date() : parsed;
}
```
משתמש זדוני יכול לשלוח `queuedAt: "2020-01-01"` ולרשום כניסה בתאריך עבר.
✅ פתרון: הגבל queuedAt לטווח סביר, למשל: לא יותר מ-24 שעות אחורה. הוסף validation:
```javascript
const MAX_QUEUE_AGE_MS = 24 * 60 * 60 * 1000;
if (data.queuedAt) {
  const parsed = new Date(data.queuedAt);
  const age = Date.now() - parsed.getTime();
  now = (!isNaN(parsed) && age >= 0 && age < MAX_QUEUE_AGE_MS) ? parsed : new Date();
}
```

---

**🟡 W-02 — Apps Script: Linear search ב-Sheets לא עמיד לנפח**

📍 מיקום: `Code.gs` שורות 185–197 (createEntry), 131–148 (searchVehicle), 488–499 (updateNotes)
🔍 תיאור: כל הפונקציות מבצעות `getDataRange().getValues()` ואז לולאה לינארית על כל השורות. Google Sheets לא מאינדקס. עם 500+ רכבים, כל חיפוש סורק שורות רבות ב-server-side GAS.
💥 השפעה: ב-test הביצועים ב-CLAUDE.md יש דרישה "table with 500+ entries renders without freeze" — אך האטה תגיע **בצד Apps Script**, לא בצד ה-table. Lookup עשוי לקחת 3–5 שניות ב-Sheets גדולים.
✅ פתרון: שקול להשתמש ב-`TextFinder` של Apps Script שמהיר יותר מלולאה ידנית, או הוסף שמירת map עם `CacheService` (Apps Script Cache).

---

**🟡 W-03 — רשימת בדיקות ב-CLAUDE.md: אין עדיפויות ואין שיוך לקבצים**

📍 מיקום: `CLAUDE.md` — כל פרק הבדיקות
🔍 תיאור: רשימת הבדיקות מוגדרת כ-checklist פשוט של checkboxes. הבעיות המרכזיות:

1. **אין עדיפות** — בדיקת "clipboard iOS" ובדיקת "XSS injection" נראות זהות ברשימה.
2. **אין שיוך לקובץ** — אין ציון איזה קובץ/component כל בדיקה בודקת.
3. **אין הפרדה: automated vs manual** — חלק מהבדיקות (placa format, KPI calculations) ניתנות לאוטומציה ב-unit tests, אחרות (clipboard iOS) הן manual בלבד.
4. **אין הגדרת "מוכן לפרסום"** — כמה בדיקות חייבות לעבור לפני merge? כולן?

💥 השפעה: מפתח שמריץ את הרשימה לפני release ידלג על בדיקות criticals בטעות בגלל שהן נראות כמו שאר הבדיקות.
✅ פתרון: ראה סעיף 4 — המלצות מבניות.

---

**🟡 W-04 — אין בדיקה לטיפול ב-Apps Script quota exhaustion**

📍 מיקום: `CLAUDE.md` — Integration Tests / Performance Tests
🔍 תיאור: Google Apps Script ל-free account מוגבל ל-6 דקות ריצה ביום ו-20,000 קריאות ביום. רשימת הבדיקות לא כוללת:
- מה קורה כש-Apps Script מחזיר שגיאת 429/quota?
- מה קורה כש-Apps Script מחזיר HTML (הדף של "service unavailable") במקום JSON?

כרגע `apiGet` בודק `response.ok` אך אם Apps Script מחזיר HTML עם status 200 (כפי שקורה לפעמים עם "service unavailable"), `response.json()` יזרוק exception ב-catch שישלח למשתמש "שגיאת שרת".
✅ פתרון: הוסף בדיקת `typeof data` לפני שימוש בתגובה + הוסף integration test ספציפי לתגובת quota.

---

**🟡 W-05 — VaultEntry: אין live region לשגיאת placa**

📍 מיקום: `vault-entry.html` שורה 63
🔍 תיאור: `#placa-error` מוגדר `role="alert"` אך ב-`showError()`:
```javascript
placaError.textContent = message;
placaError.classList.remove('hidden');
```
שינוי `textContent` על אלמנט שהיה `hidden` (display:none) לא תמיד מופעל על ידי screen readers — כי ה-DOM node היה מוסתר בזמן שה-content שינה.
✅ פתרון: מומלץ להחזיק את האלמנט ב-DOM עם `visibility:hidden` / `height:0` במקום `display:none`, או להשתמש ב-`aria-describedby` על ה-input field.

---

**🟡 W-06 — אין בדיקה לסנכרון גרסאות בין frontend ל-Apps Script**

📍 מיקום: `CLAUDE.md` — Integration Tests
🔍 תיאור: ב-v1.2.0 נוסף שדה `queuedAt` לפרוטוקול ה-POST. אם מישהו עדכן את ה-frontend אך שכח לעדכן את ה-Apps Script (פריסה נפרדת), המערכת תפסיק לשמר timestamps.
✅ פתרון: הוסף בדיקה לפרוטוקול הגרסאות — ה-`ping` response יכלול `version: "1.2.0"` מ-Apps Script, ו-frontend יבדוק שהגרסאות תואמות.

---

**🟡 W-07 — VaultEntry: resetForm() לא מאפס את selectedTipo**

📍 מיקום: `vault-entry.js` שורה 484–493
🔍 תיאור: `resetForm()` מנקה placa, notes, errors — אך לא מאפס את `selectedTipo` לברירת המחדל (`auto`). אם עובד רשם אופנוע ולחץ שמור, הטופס הבא יהיה עם "מוטו" כברירת מחדל — בלי אינדיקציה ויזואלית כי ה-state נשאר ב-toggle.

למרות שה-toggle-button מראה visually את הבחירה, זה עלול לגרום לרישום שגוי של הרכב הבא.
✅ פתרון: ב-`resetForm()` הוסף איפוס של ה-toggle:
```javascript
selectedTipo = CONFIG.DEFAULT_VEHICLE_TYPE;
// + reset UI toggle
```

---

### 🔵 SUGGESTION (8)

---

**🔵 S-01 — הוסף property "blocking" לבדיקות ב-CLAUDE.md**

כל בדיקה צריכה לציין אם היא `blocking: true` (חוסמת פרסום) או `blocking: false` (nice-to-have). זה יאפשר לצוות לרוץ רק ה-blocking tests לפני כל release, ואת השאר אחת לשבוע.

---

**🔵 S-02 — פצל את רשימת הבדיקות ל-3 רמות תדירות**

רשימת הבדיקות כולה רשומה תחת "run before every release" — אך חלק מהן כבדות (compatibility tests על דפדפנים מרובים, performance tests על 500+ records). מומלץ לפצל:

- **Pre-commit**: בדיקות fast/automated בלבד (validation logic, i18n keys)
- **Pre-release**: כל הבדיקות הפונקציונליות וה-security
- **Weekly/Monthly**: Compatibility tests, Performance tests עם נתונים אמיתיים

---

**🔵 S-03 — הוסף smoke test אוטומטי ל-Apps Script**

Apps Script לא נבדק ב-CI/CD. מומלץ ליצור Apps Script function בשם `runSmokeTest()` שמבצעת:
1. `createEntry` עם plate test
2. `searchVehicle` לאותה plate → מצפה ל-`isNew: false`
3. ניקוי — מחיקת שורת הtest

תריץ את זה ידנית אחרי כל deploy של Apps Script.

---

**🔵 S-04 — הוסף בדיקה: "מה קורה כשה-sessionStorage מלא"**

`sessionStorage` יכול להגיע ל-5MB limit. עם אלפי רשומות (סמן קיצוני), `sessionStorage.setItem()` יזרוק `QuotaExceededError` ויפיל את ה-`addToSession()` ללא שגיאה גלויה.
מומלץ לעטוף ב-try/catch עם fallback graceful.

---

**🔵 S-05 — הוסף data-testid attributes לאלמנטים ב-HTML**

כרגע, אם תחליטו לכתוב E2E tests ב-Playwright/Cypress בעתיד, תצטרכו להשתמש ב-selectors שבירים (`.ve-submit`, `#submit-btn`). מומלץ להוסף `data-testid` לאלמנטים קריטיים:
```html
<button id="submit-btn" data-testid="submit-entry-btn" ...>
<input id="placa-input" data-testid="placa-input" ...>
```

---

**🔵 S-06 — הוסף בדיקת "גרסת Apps Script לא עדכנית"**

כשהמשתמש טוען את CommandCenter ועושה ping, הוסף בדיקה שה-response כולל גרסה. אם הגרסה לא תואמת, הצג banner: "Dashboard version mismatch — please contact admin".

---

**🔵 S-07 — בדיקת Regression חסרה: "clipboard fallback ב-non-HTTPS"**

ב-`vault-entry.js` שורות 338–358, `navigator.clipboard.writeText()` עובד רק ב-HTTPS. ה-fallback עם `document.execCommand('copy')` קיים, אך אין בדיקה שמאמתת שהוא עובד. ה-CLAUDE.md מציין בדיקה ל-iOS clipboard אך לא ל-HTTP environments (dev/localhost).

---

**🔵 S-08 — הוסף "definition of done" לכל test suite**

בסוף כל קטגוריית בדיקה ב-CLAUDE.md, הוסף שורת סיכום:
```
✅ Pass criteria: X מתוך Y בדיקות חייבות לעבור לפני release
```
כרגע לא ברור אם 95% מספיק, או שצריך 100%.

---

## 3. ניתוח רשימת הבדיקות הקיימת (CLAUDE.md)

### מה עובד טוב

**כיסוי מקיף יוצא דופן** — הרשימה מכסה 8 קטגוריות (usability, security, functional, integration, performance, accessibility, compatibility, localization) + regression. זה רמה גבוהה לפרויקט indie/client.

**Regression tests מפורטים לפי קובץ** — הסעיף "After any change to X" הוא pattern מעולה שמונע regressions. רוב הצוותים לא עושים את זה.

**Security tests מציאותיים** — הבדיקות ל-XSS injection, server-side validation, ו-`.env` ב-git הן ישירות לענין ורלוונטיות לסביבת deployment האמיתית.

**Offline queue — 3 בדיקות ספציפיות** — timestamps, persistence across reload, order preservation — אלה edge cases שרוב הצוותים מדלגים עליהם. הם חשובים מאוד לסביבה של חניון (עובד שנכנס לאזור ללא אות).

### פערים מרכזיים שמצאתי

**פער 1: אין בדיקה לטיפול בשגיאות Apps Script Quota**
כנ"ל ב-W-04. Apps Script עם free account יכול להגיע ל-quota ב-יום עמוס. אין בדיקה לתרחיש הזה.

**פער 2: אין בדיקה לגודל data ב-KPI list modal**
הבדיקות ל-KPI card modal ב-CommandCenter בודקות שהוא נפתח/נסגר, אך לא שהוא מציג נתונים נכונים כשיש 50+ items ברשימה (overflow, scroll, truncation).

**פער 3: אין בדיקה לאינטגרציה בין שני Tabs פתוחים**
אם VaultEntry ו-CommandCenter פתוחים בו-זמנית (Tab A + Tab B):
- VaultEntry שומר entry ← cache מנוקה
- CommandCenter ב-Tab B — cache שלו **לא** מנוקה
- המשתמש רואה נתונים ישנים עד לרענון ידני

**פער 4: הבדיקה "Duplicate prevention" היא weak**
הבדיקה אומרת "save same plate twice quickly → only one new-vehicle record". אך היא לא מגדירה "quickly" — תוך כמה ms? האם הבדיקה היא frontend (cooldown 3s) או server-side (atomicity)?

**פער 5: אין בדיקה ל-RTL + overflow**
הבדיקות ל-RTL קיימות, אך אין בדיקה שלוחית ארוכה (`ABC-DEF-123`, 10 תווים) לא שוברת את ה-layout בעברית RTL, במיוחד בטבלה של CommandCenter.

**פער 6: אין בדיקת error handling ל-updateNotes כשה-record לא קיים**
ב-Code.gs שורה 500: `throw new Error('Record not found: ' + id)` — מה קורה ב-frontend כשמגיע error כזה? אין בדיקה שמאמתת שה-notes modal מציג הודעת שגיאה נכונה.

---

## 4. המלצות מבניות לרשימת הבדיקות

### הצעה לפורמט מעודכן

במקום checklist פשוט:
```
- [ ] בדיקה כלשהי
```

הצע מבנה עשיר יותר:
```markdown
| בדיקה | קובץ | סוג | Blocking | אוטומטי |
|-------|------|-----|----------|---------|
| Plate entry flow | vault-entry.js | Usability | ✅ | ❌ |
| XSS plate field | vault-entry.js | Security | ✅ | ✅ |
| iOS clipboard | vault-entry.js | Compat | ❌ | ❌ |
```

### הצעה לseverity tagging

הוסף prefix לכל בדיקה:
- `[P0]` — Critical, חוסם פרסום
- `[P1]` — חשוב, אך לא חוסם
- `[P2]` — Nice to have

### הצעה: הפרד Automated מ-Manual

**Automated Tests** (ניתן לאוטומציה עם Vitest/Playwright):
- Placa validation (format, length, uppercase)
- t() interpolation ב-i18n
- KPI calculation logic (עם mock data)
- Cache TTL behavior

**Manual Tests** (חייבים בדיקה אנושית):
- iOS clipboard
- RTL visual layout
- Offline behavior ב-Android
- Screen reader (VoiceOver/NVDA)

---

## 5. ניתוח Code Quality

### חוזקות בקוד

- **JSDoc מלא** — כל פונקציה public מתועדת עם @param, @returns, @throws ✅
- **XSS protection** — שימוש עקבי ב-`textContent` (לאחר תיקון v1.2.0) ✅
- **separation of concerns** — `sheets.js` כ-abstraction layer הוא design pattern נכון ✅
- **Freeze config** — `Object.freeze(CONFIG)` מונע שינוי accidental ✅
- **Rate limiting** — `rateLimit()` ב-sheets.js ✅
- **Offline queue** — `localStorage` + auto-retry עם timestamp preservation ✅

### נקודות שמצריכות תשומת לב

- **`searchVehicle` ב-`handleSubmit` לא נקרא לפני submit** — המשתמש יכול לדלג על ה-lookup ולהגיש ישירות. הכניסה תיכנס ל-Apps Script שיעשה את ה-lookup שם — זה בסדר מבחינה פונקציונלית, אך הבאדג' לא יוצג.
- **`innerHTML` ב-`renderSessionList`** — שורה 295: `sessionItems.innerHTML = ''` — safe (ריקון), אבל כדאי לתעד שזה intentional.
- **`document.execCommand('copy')` deprecated** — ה-fallback ב-`copySessionPlates` משתמש ב-API שהוצא משימוש. יש לתעד את הסיכון ולוודא שבדיקת ה-clipboard נעשית גם על דפדפנים שלא תומכים ב-`execCommand`.

---

## 6. סיכום

```
📊 סיכום ממצאים
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔴 Critical:    2 (C-01, C-02)
🟡 Warnings:    7 (W-01 עד W-07)
🔵 Suggestions: 8 (S-01 עד S-08)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

🔴 חייב לתקן לפני פרסום:
  → C-01: CORS לא מיושם, URL ב-config.js חשוף
  → C-02: Race condition אפשרי ב-submit כפול

🟡 מומלץ לתקן בגרסה הקרובה:
  → W-01: queuedAt injection validation
  → W-02: Linear scan performance ב-Apps Script
  → W-07: resetForm לא מאפס selectedTipo

📋 פערים ברשימת הבדיקות:
  → אין עדיפות (P0/P1/P2)
  → אין הפרדה automated/manual
  → חסרות 6 בדיקות ספציפיות (ראה סעיף 3)
  → אין definition of done ברורה

✅ מוכן לפרסום: כן — בתנאי שיטופלו C-01 ו-C-02
```

---

*דוח זה הוכן על בסיס קריאת קוד מלאה של כל קבצי הפרויקט.*
*גרסה: 1.2.0 | תאריך: 2026-03-08*
