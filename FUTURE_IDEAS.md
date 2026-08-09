# Future implementation ideas

Things worth doing eventually, deliberately set aside for now. Not a bug list — see git history/issues for that. Add a short "why parked" note when setting something aside, and remove/move it here to done once implemented (or delete if abandoned).

## Open

- **Alternatives as a carousel** — instead of the current show/hide-all toggle for alternative options on a day, consider a swipeable carousel UI for browsing alternatives. (Raised 2026-08-07, not yet scoped.) Confirmed 2026-08-08: this is a real UI rebuild (new interaction behavior across every alt-group in the app, needs a live browser check afterward), deliberately deferred as heavier work — not something to pick up casually.
- **Time-to-spend estimates for shrines/temples/attractions, all 36 days** — recommended duration at every shrine/temple/attraction (not shops/restaurants/cafes) across the whole trip, so it's clear how long to budget per stop. Raised originally during the Day 5 redesign work, deferred as its own follow-up scope. Confirmed 2026-08-08: mechanically trivial to add per place, but the research volume is large (~40-70 individual places need a lookup) — deliberately deferred, not something to pick up casually.
- **Hands-free auto-sync from app to this repo** — a one-tap in-app button that automatically commits current in-app edits (V2Store overlay, custom events, done-checks) straight to `index.html` in this GitHub repo, no manual export/download step. Would need a Firebase Cloud Function (watching the Firestore trip doc) that calls the GitHub API to commit changes, plus logic to safely translate overlay data into valid static HTML. Explicitly deferred 2026-08-07 because it adds a backend + a GitHub write token to manage, and a bad auto-merge could break the live site. Revisit if manually reconciling "app says fixed, repo says not fixed" (see CLAUDE.md's in-app-edits-vs-repo note) becomes a recurring pain. Simpler fallback considered and also parked: a button that exports the overlay as a downloadable JSON file to hand to Claude manually.
- **Color-coding tied to area, not day** — cards/accents currently get one color per *day* (`--dc` on the day section). Idea is to instead color by *area* (neighborhood/region), so a color change signals "you've moved to a new area" — including within a single day that crosses multiple areas. (Corrected wording 2026-08-07 — original note was ambiguous.) Doc marks this "future update," not urgent. Note: a lighter version of "where am I" was implemented 2026-08-08 as a plain city-name tag per day header (not area color-coding) — see Done section.
- **Walking-segment tickets with a real route on the map (remaining instances)** — pattern proven and implemented for Day 4 (Takeshita Street, Cat Street, 2026-08-08) and Day 10/Kamikochi (Kappa Bridge → Myojin Bridge, 2026-08-09): map link switched to a Google Maps walking-directions URL (`maps/dir/?api=1&origin=...&destination=...`) instead of a single-pin search. **Lesson from the Kamikochi fix:** use raw `lat,lon` coordinates for origin/destination, not place-name text — Google Maps couldn't reliably geocode long parenthetical place names (e.g. "Kappa Bridge (河童橋) (Kamikochi 上高地)") and silently fell back to a single point instead of drawing the route; verified live in browser before/after. At least one more instance still uses the old pattern: Day 6 Kamakura, "Kotoku-in → Zeniarai Benten / Genjiyama Park" still has the start/end baked into the title text with a plain search-query map link. Worth a full sweep for any other remaining instances before calling this done everywhere.
- **Trip Expenses tab** — full spec pasted below from the original planning doc, unimplemented. No expense-tracking feature exists in the app today (only a currency converter).
- **Attach a new custom ticket as an alternative to an existing ticket** — when adding a new event via the "+" flow, offer a toggle ("this is an alternative") plus a picker to choose which existing ticket (static or custom) it's an alternative to. Raised 2026-08-09. Not easy: alternatives today are a static-HTML structure (`.grp`/`.alts`/`.alttoggle`, tagged by `nwTagGroups`/`nwPromoteInGroup`) that custom (`jt_cbk`) events don't participate in at all — they always render as independent `.customcard`s via the boot-time injection pass (`nwBuildCustomCard`). Making a custom event attach into an existing alt-group needs: a new `altOf` field on the custom-event schema, a day-scoped picker UI listing current cards to attach to, and render-pass logic to insert the new card into the target's `.alts` container (creating one if the target isn't already a group) and update the "N alternatives" toggle. Comparable scope to the already-parked "Alternatives as a carousel" idea — don't pick up casually.
- **Two-way delete on pre-planned schedule cards (hide vs. permanently delete)** — today, deleting a static schedule card (the trash-icon `rowdel` button) only ever hides it (`nwHideEvent` → `V2Store.hide`, recoverable from More → Hidden events) — there's no way to remove it from that recovery list. Raised 2026-08-09: add a real choice between "Hide (recoverable)" and "Delete permanently" when removing a card. Not easy: `nwConfirm` is a single yes/no modal, so this needs a new 3-option confirm UI (Cancel/Hide/Delete), a new "permanently deleted" flag distinct from `hidden` in `V2Store`, and a filter added to the Hidden-events recovery screen so permanently-deleted items don't reappear there. (Note: custom (`jt_cbk`) events are the opposite today — `delBooking` always fully deletes with no hide option at all; if this gets built, decide whether custom events get the same two-choice treatment too.)

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

- **Day 2 (Sept 8)** — "Nihonbashi Bridge" and the "Nihonbashi Muromachi + Mitsukoshi" department store stop were flagged as redundant/expensive-irrelevant; still both present, unchanged.
- **Day 2 dinner note fused into a bar card** — the card titled "Dinner in Ginza (kept light) then Bar High Five" mixes a dinner note into what's really a bar entry; original ask was to separate or drop the dinner note entirely.
