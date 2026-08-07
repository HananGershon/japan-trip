# Future implementation ideas

Things worth doing eventually, deliberately set aside for now. Not a bug list — see git history/issues for that. Add a short "why parked" note when setting something aside, and remove/move it here to done once implemented (or delete if abandoned).

## Open

- **Alternatives as a carousel** — instead of the current show/hide-all toggle for alternative options on a day, consider a swipeable carousel UI for browsing alternatives. (Raised 2026-08-07, not yet scoped.)
- **Hands-free auto-sync from app to this repo** — a one-tap in-app button that automatically commits current in-app edits (V2Store overlay, custom events, done-checks) straight to `index.html` in this GitHub repo, no manual export/download step. Would need a Firebase Cloud Function (watching the Firestore trip doc) that calls the GitHub API to commit changes, plus logic to safely translate overlay data into valid static HTML. Explicitly deferred 2026-08-07 because it adds a backend + a GitHub write token to manage, and a bad auto-merge could break the live site. Revisit if manually reconciling "app says fixed, repo says not fixed" (see CLAUDE.md's in-app-edits-vs-repo note) becomes a recurring pain. Simpler fallback considered and also parked: a button that exports the overlay as a downloadable JSON file to hand to Claude manually.
- **"Where am I in Japan" context indicator** — within a day view, show current city/area context so it's clear where a given stretch of the day is happening, especially on days that cross areas.
- **Color-coding tied to area, not day** — cards/accents currently get one color per *day* (`--dc` on the day section). Idea is to instead color by *area* (neighborhood/region), so a color change signals "you've moved to a new area" — including within a single day that crosses multiple areas. (Corrected wording 2026-08-07 — original note was ambiguous.)
- **Walking-segment tickets with a real route on the map** — some walking segments are already merged into one ticket with a text "start → end" description (e.g. the Daibutsu hiking trail), but tapping the map for those just does a Google Maps pin search for the end point — it doesn't draw an actual walking route/polyline from start to end. Idea: when a ticket represents a walk between two points, its map button should open/show the real route (e.g. Google Maps directions link between the two points, or an embedded polyline) instead of a single-pin search.
- **Trip Expenses tab** — full spec pasted below from the original planning doc, unimplemented. No expense-tracking feature exists in the app today (only a currency converter).

  <details>
  <summary>Full PRD (Hebrew, from original planning doc)</summary>

  # PRD: טאב הוצאות וניהול תקציב (Trip Expense Tracker)

  ## 1. סקירה כללית וארכיטקטורה (Overview & Architecture)

  פיצ'ר **טאב ההוצאות** נועד לתת מענה מלא לניהול, תיעוד וחלוקת הוצאות כספיות במהלך טיול קבוצתי או אישי.
  הפיצ'ר משלב בין **הוצאות מתוכננות מתוך הלו"ז** לבין **הוצאות ספונטניות (ידניות)**, ומסנכרן את הנתונים בזמן אמת בין כל המכשירים המחוברים לטיול.

  ### עקרונות ליבה:
  * **אפליקציית ווב עם סנכרון בזמן אמת:** כל הזנה, עריכה או מחיקה משתקפת מיידית אצל כל חברי הקבוצה.
  * **שפה ואייקונים:** שפת הממשק בעברית/אנגלית לפי הגדרת המערכת. **שימוש בלעדי באייקוני SVG** בכל רכיבי ה-UI (ללא אימוג'ים).
  * **מטבע ברירת מחדל:** **יין יפני (JPY - ¥)**, עם תמיכה מלאה בהזנה במטבעות מרובים והמרה אוטומטית למטבע התצוגה של הדוח.

  ## 2. מנגנון סגירת אירועים מבוססי לו"ז (Schedule-Expense Integration)

  בכל אירוע בלו"ז הקיים באפליקציה קיים רכיב לצירוף הוצאה: **`+ Add Expense`**.

  ### לוגיקת התנאים למחיקה אוטומטית מהלו"ז:
  אירוע יימחק מהלו"ז המרכזי ויעבור בלעדית לטאב ההוצאות **רק כאשר מתקיימים שני התנאים הבאים יחד (תנאי AND):**

  1. **תנאי זמן:** הזמן הנוכחי במכשיר עבר את שעת סיום האירוע (`currentTime > event.endTime`).
  2. **תנאי הוצאה:** הוזנה לאירוע הוצאה בפועל (`event.expense != null`).

  ## 3. מבנה סוגי ההוצאות והקטגוריות (Expense Types & Categories)

  כדי למנוע בלבול וכפילויות, יצירת הוצאה מתבצעת בשני מסלולים:

  1. **הוצאה מתוך אירוע קיים בלו"ז:**
     * **שם ההוצאה:** נמשך אוטומטית משם המקום/האירוע בלו"ז.
     * **קטגוריה:** נגזרת אוטומטית מסוג האירוע שסווג בלו"ז (אין צורך לבחור קטגוריה ידנית).
  2. **הוצאה חופשית/ספונטנית (כתיבה חופשית):**
     * **שם ההוצאה:** הקלדה חופשית של המשתמש.
     * **קטגוריה:** בחירה מתוך רשימת הקטגוריות המוגדרות מראש באפליקציה.

  ## 4. תצוגות הדוח בטאב ההוצאות (Report & View Modes)

  הדוח בטאב ההוצאות כולל מנגנון בחירה בין שתי תצוגות עיקריות (Toggle):

  * **תצוגה כרונולוגית לפי תאריכים (By Date - ברירת מחדל):** ההוצאות מקובצות לפי ימי הטיול. תחת כל תאריך מופיעות ההוצאות של אותו יום בצירוף **סיכום יומי כולל** בראש הקבוצה.
  * **תצוגת רשימה מלאה (All Expenses):** רשימה רציפה אחת של כל ההוצאות בטיול לפי סדר כרונולוגי יורד (מהחדש לישן), ללא חלוקה לפי ימים וללא פילטור.

  ### רכיבי כרטיס הוצאה בדוח:
  * **שם המקום / האירוע** (משם הלו"ז או מהכתיבה החופשית).
  * **תאריך ושעה** של ההוצאה/אירוע.
  * **סכום ומטבע:** סכום מומת בולט במטבע הדוח + סכום מקורי בסוגריים (אם נדרש).
  * **אינדיקטור מקור (Badge):** חיווי ויזואלי (אייקון SVG) המציין האם ההוצאה נמשכה מאירוע בלו"ז (`From Schedule`) או שהוזנה ידנית.
  * **פרטי תשלום:** מי שילם ואופן החלוקה בקבוצה.

  ## 5. מבנה טאב ההוצאות (UI Layout)

  ```
  +-----------------------------------------------------------------------+
  |  SUMMARY BAR                                                          |
  |  [SVG Wallet] סך הכל: ¥142,500                    [ Currency: JPY ▾ ] |
  |  המאזן שלך: מגיע לך ¥3,200                                            |
  +-----------------------------------------------------------------------+
  |  VIEW TOGGLE: [ By Date (Default) ]  |  [ All Expenses ]              |
  +-----------------------------------------------------------------------+
  |  EXPENSES FEED                                                        |
  |                                                                       |
  |  -- יום שני, 27 ביולי (סך הכל ליום: ¥32,500) --                        |
  |  +-----------------------------------------------------------------+  |
  |  | [SVG Utensils]  מסעדת ראמן Ichiran                  ¥4,500   |  |
  |  |                 27/07 19:30 | [SVG Calendar] From Schedule   |  |
  |  |                 שולם ע"י דני | חלוקה: שווה (3)                  |  |
  |  +-----------------------------------------------------------------+  |
  |  | [SVG Shopping]  מזכרות ברחוב                         ¥2,000   |  |
  |  |                 27/07 16:15 | הזנה ידנית                        |  |
  |  |                 שולם ע"י חנן | חלוקה: אישי                      |  |
  |  +-----------------------------------------------------------------+  |
  +-----------------------------------------------------------------------+
  |                                                      [ SVG Plus (FAB) ]|
  +-----------------------------------------------------------------------+
  ```

  ### א. סרגל סיכום עליון (Summary Bar)
  * **סך כל הוצאות הטיול:** מחושב במטבע התצוגה הנבחר.
  * **בחירת מטבע תצוגה (Report Currency Selector):** Dropdown הממוקם בתוך הדוח/סרגל הסיכום (עד להקמת טאב הגדרות מרכזי). שינוי המטבע מעדכן מיידית את כל החישובים בדוח.
  * **מאזן אישי (Personal Balance):** סיכום מהיר למשתמש המחובר (למשל: *"מגיע לך ¥3,200"* או *"אתה חייב ¥1,500"*).

  ### ב. כפתור פעולה מהירה (FAB)
  * כפתור **`+`** צף בתחתית המסך להוספה מהירה של הוצאה ספונטנית.

  ## 6. מנגנון חלוקת עלויות ואיזון חובות (Expense Splitting)

  לכל הוצאה ניתן להגדיר חלוקה בין חברי הקבוצה:

  1. **מי שילם? (Paid By):** בחירת המשתמש/ים שביצעו את התשלום בפועל.
  2. **עבור מי? (Split Among):**
     * **שווה בשווה (Equal Split):** ברירת מחדל — חלוקה שווה בין כל/חלק מחברי הקבוצה.
     * **לפי סכום/אחוזים (Custom Split):** הזנת סכום ספציפי לכל משתתף.

  ### מסך התחשבנות (Settle Up View)
  תת-מסך המחשב את צמצום החובות בקבוצה (Simplify Debts Algorithm) כדי למזער את מספר העברות הכספים הנדרשות:
  * תצוגה ברורה: *"דני צריך להעביר למיכל ¥3,500"*.
  * כפתור **"סמן כמשולם" (Mark as Settled)** המאפס את החוב הרלוונטי.

  ## 7. הגדרת קטגוריות ואייקוני SVG

  כל הקטגוריות המוגדרות באפליקציה מיוצגות באמצעות **אייקוני SVG בלבד** (ללא אימוג'ים):

  | קטגוריה | שם באנגלית | אייקון SVG מומלץ |
  | :--- | :--- | :--- |
  | **אוכל ושתייה** | Food & Dining | `Utensils` / `Coffee` |
  | **תחבורה** | Transportation | `Car` / `Bus` / `Train` |
  | **אטרקציות ובילוי** | Entertainment | `Ticket` / `Camera` |
  | **קניות ומזכרות** | Shopping | `ShoppingBag` / `Tag` |
  | **לינה** | Accommodation | `Bed` / `Hotel` |
  | **כללי / אחר** | General | `Receipt` / `CreditCard` |

  ## 8. טיפול במטבעות והמרה (Multi-Currency Handling)

  1. **מטבע בסיס מוביל:** **JPY (יין יפני)**.
  2. **אחסון נתונים:** כל הוצאה שומרת ב-DB את הסכום והמטבע המקוריים שבהם בוצעה העסקה (`originalAmount`, `originalCurrency`).
  3. **חישוב המרה:** המערכת ממירה בזמן אמת את הסכום המקורי למטבע התצוגה הנבחר של הדוח לפי שער חליפין מעודכן.
  4. **תצוגה שקופה:** בכרטיס ההוצאה מוצג הסכום המומת באופן בולט, ומתחתיו הסכום המקורי במטבע שבו שולם.

  ## 9. מבנה נתונים (Data Schema)

  אובייקט JSON מייצג להוצאה בבסיס הנתונים:

  ```json
  {
    "id": "exp_88392",
    "tripId": "trip_japan_2026",
    "sourceEventId": "evt_ramen_101",
    "title": "מסעדת ראמן Ichiran",
    "originalAmount": 4500,
    "originalCurrency": "JPY",
    "categoryId": "food",
    "eventDate": "2026-07-27T19:30:00Z",
    "isFromSchedule": true,
    "paidBy": {
      "user_1": 4500
    },
    "splitWith": ["user_1", "user_2", "user_3"],
    "splitType": "EQUAL",
    "createdBy": "user_1",
    "createdAt": "2026-07-27T19:35:00Z",
    "updatedAt": "2026-07-27T19:35:00Z"
  }
  ```
  </details>

## Content corrections still needed

Real itinerary/content fixes flagged in the original planning doc, not yet applied to the static file. Distinct from feature ideas above — these are "the data is wrong," not "the app should do something new."

- **Ticket titles show category, not place name** — throughout the app, a card's title (`<p class="n">`) often shows a generic label like "Lunch" or "Dinner" instead of the actual restaurant/place name, which is buried lower in the card body. Example: Day 6 lunch card title is "Lunch," actual place "Hama Zen" is inside the details list.
- **Day 4 (Sept 10, Shinjuku/Harajuku)** — lunch is still "Acacia Shinjuku Honten," flagged for replacement, unchanged. Dinner slot exists but is a placeholder ("Light meal") with no actual restaurant name/address filled in.
- **Breakfast repetition** — "Byron Bay Coffee Daimon" and "GAZEBO TOKYO" each appear as breakfast multiple times across different days; original ask was for variety instead of repeats.
- **Day 2 (Sept 8)** — "Nihonbashi Bridge" and the "Nihonbashi Muromachi + Mitsukoshi" department store stop were flagged as redundant/expensive-irrelevant; still both present, unchanged.
- **Day 2 dinner note fused into a bar card** — the card titled "Dinner in Ginza (kept light) then Bar High Five" mixes a dinner note into what's really a bar entry; original ask was to separate or drop the dinner note entirely.
