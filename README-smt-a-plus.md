# NQ/ES SMT A+ Setup — אינדיקטור TradingView (Pine Script v6)

אינדיקטור **סלקטיבי מאוד** שמנהל את כל מחזור החיים של סטאפ SMT איכותי בין **NQ ל‑ES**:

```
Liquidity Sweep → SMT → Displacement → MSS/CISD → FVG/IFVG → Retest → Entry → Stop/TP → Result
```

הקובץ: [`nq-es-smt-a-plus-setup.pine`](nq-es-smt-a-plus-setup.pine)

> **עיקרון מנחה:** SMT לבדו **אינו** כניסה. כניסה מסומנת **רק** אחרי שכל השרשרת הושלמה **וגם** המחיר חזר (Retest) לאזור הכניסה. עדיף לפספס עסקה מאשר להציג הרבה אותות חלשים.

---

## 1. ארכיטקטורה

### 1.1 מכונת מצבים (State Machine)
לכל רגע נתון מנוהל **סטאפ פעיל אחד** (Long או Short) דרך המצבים הבאים. כל מעבר מתבצע **רק על נר סגור** (`barstate.isconfirmed`), ולכן אין Repaint.

| State | משמעות | תנאי מעבר קדימה |
|---|---|---|
| `IDLE` | ממתין | Sweep שנוצר קרוב ל‑SMT באותו כיוון → זריעת סטאפ |
| `SMT_CONFIRMED` | Sweep+SMT זוהו | Displacement בכיוון העסקה תוך `maxSmtToDisp` נרות |
| `DISPLACEMENT_CONFIRMED` | תנועת עוצמה | MSS ו/או CISD תוך `maxDispToMss` נרות |
| `STRUCTURE_SHIFT_CONFIRMED` | שינוי מבנה | קיים FVG/IFVG → חישוב Entry/Stop/TP/Score/Grade |
| `WAITING_FOR_RETEST` | הסטאפ מדורג ומצויר | המחיר חוזר לאזור ומגיע ל‑Entry |
| `ENTRY_TRIGGERED` | כניסה | ניהול TP/SL, ואז Cooldown ואיפוס |
| `INVALIDATED` / `EXPIRED` | פסילה/פקיעה | Cooldown ואיפוס ל‑IDLE |

מצבי פסילה מיידית: אין Displacement/MSS בזמן, אזור בוטל לפני Retest, המחיר הגיע ליעד לפני הכניסה, Bias חזק הפוך, או הסטאפ ישן מדי.

### 1.2 מבנה נתונים
- **`type Setup`** — אובייקט יחיד (`var Setup S`) המחזיק את כל נתוני הסטאפ: כיוון, מצב, רמת Sweep, זמני SMT/Displacement/MSS, אזור, Entry/Stop/TP, Score ו‑Grade.
- **`type Gap`** — מערך של FVG‑ים עם מעקב אחרי היפוך (IFVG) ותוקף.
- **מערכים** ל‑Lines/Labels וניהול רמות נזילות מדורגות, עם מחיקת אובייקטים ישנים כדי לא לעבור את מגבלות TradingView.

### 1.3 הגדרות הרכיבים
- **SMT (Swing)** — פיבוטים של הגרף משמשים כעוגן זמן; באותו נר נדגם מחיר הנכס המשווה (NQ↔ES). דיברגנס = בדיוק אחד מהשניים יצר Low/High חדש (XOR), עם סף `ATR × tolerance` נפרד לכל נכס.
- **SMT (Liquidity Level)** — נכס אחד לוקח נזילות מעבר לרמה (PDL/Asia/London/...) והשני לא לוקח את הרמה המקבילה שלו.
- **Sweep** — ברירת מחדל: Wick חוצה רמה + Close חוזר פנימה. אפשרויות נוספות: Close+Reclaim (תוך N נרות), Any‑break.
- **Displacement** — גוף נר `≥ ATR × factor` **או** `≥ ממוצע גוף × factor`, עם Close בחלק הנכון של הנר.
- **MSS** — סגירה מעבר לפיבוט פנימי אחרון (Lower‑High ללונג / Higher‑Low לשורט).
- **CISD** — סגירה מעבר ל‑Open של רצף הנרות ההפוך האחרון שקדם לתנועה החדה.
- **FVG** — גאפ תלת‑נרי מסונן (`≥ ATR% / ticks`). **IFVG** — FVG שנפרץ והחליף תפקיד (תמיכה↔התנגדות).

### 1.4 מניעת Repainting
1. כל מעברי המצב והציור מתבצעים ב‑`barstate.isconfirmed` (נרות סגורים בלבד).
2. פיבוטים מאושרים רק אחרי `pvtRight` נרות — התווית מופיעה בזמן האישור, לא בזמן ההיווצרות.
3. כל `request.security()` משתמש ב‑`barmerge.lookahead_off`; רמות יום נגזרות מ‑`[1]`.
4. סטאפ שהופעל אינו משנה דירוג בדיעבד (רק מצב TP/SL מתעדכן), וסטאפים כושלים נשמרים בהיסטוריה לבדיקה.

---

## 2. התקנה

1. פתחו גרף פיוצ'רס עם ווליום אמיתי — מומלץ `CME_MINI:NQ1!` או `CME_MINI:ES1!`.
2. בחרו טיים‑פריים **3 דקות** (או 1/5 דקות).
3. `Pine Editor` → הדביקו את תוכן `nq-es-smt-a-plus-setup.pine` → **Add to chart**.
4. האינדיקטור מזהה אוטומטית: על גרף NQ הוא משווה ל‑ES ולהפך.

### הגדרות מומלצות ל‑NQ/ES (ברירות המחדל)
| נושא | ערך |
|---|---|
| Pair | `NQ1!` מול `ES1!` (למיקרו: `MNQ1!` / `MES1!`) |
| Chart TF | 3 דקות |
| Mode | **Strict** (A+ בלבד) |
| Pivot L/R | 3 / 3 · Internal 2 / 2 |
| ATR | 14 · SMT tolerance 0.05 ATR |
| Sweep | Wick + Close חזרה |
| Bias | EMA · 15m ו‑1H חייבים להסכים · 4H בונוס |
| RR | Min 2 · Target 3 |
| Sessions | NY AM + London (שעון `America/New_York`, DST אוטומטי) |

לצפייה גם ב‑A/B: שנו `Mode → Flexible` והתאימו `Flexible minimum score`.

### תצוגה נקייה — למה לפעמים יש "הרבה סימונים"?
האינדיקטור מפריד בין **שכבות אבחון** (כל SMT, כל Sweep, כל FVG) לבין **הסטאפ המדורג** (הסיגנל האמיתי).
בטיים‑פריים נמוך (1 דקה) NQ/ES מתפצלים כמעט בכל פיבוט זעיר → מאות תוויות SMT ורמות FVG. לכן:

- **ברירות המחדל כבר "רגועות":** `Standalone SMT labels` = Off, `Sweep markers` = Off, ו‑`FVG boxes: only significant` = On (מסתיר גאפים זעירים).
- **עבדו ב‑3 או 5 דקות** (טווח היעד של האינדיקטור), לא ב‑1 דקה.
- ב‑**Strict Mode** רק סטאפ **A+** מצויר (קווי Entry/Stop/TP + תווית ציון + משולש כניסה). B/C נשמרים ב‑Dashboard/Debug בלבד.
- לתצוגה מינימלית לחלוטין: כבו גם `FVG / IFVG boxes` ו‑`Liquidity level lines`, והשאירו רק את הסטאפים המדורגים.

> בצילום לדוגמה נראה סטאפ **SHORT בציון 69 = Grade B** (HTF Bias = Neutral/Mixed, Sweep על Swing‑High בלבד). זו **התנהגות תקינה** — האינדיקטור דירג אותו כבינוני; ב‑Strict הוא לא יצויר כסיגנל.

---

## 3. טבלת Inputs (עיקרי)

| קבוצה | Input | ברירת מחדל | תפקיד |
|---|---|---|---|
| Symbols | Symbol 1 / 2 | NQ1! / ES1! | זוג ה‑SMT |
| Symbols | Auto‑detect | On | בחירת נכס משווה לפי הגרף |
| SMT | SMT method | Swing | Swing / Liquidity Level / Both |
| SMT | Pivot L/R | 3 / 3 | אישור פיבוט (מונע Repaint) |
| SMT | Min deviation (×ATR) | 0.05 | סף רעש ל‑Lower‑Low/Higher‑High |
| Liquidity | PDH/PDL, Asia, London, Prev‑Sess, Equal, Swing | On | רמות נזילות ודירוגן |
| Sweep | Sweep definition | Wick | Wick / Close+Reclaim / Any |
| HTF Bias | Method | EMA | EMA / Market Structure / Prem‑Disc / Manual |
| HTF Bias | TF1/TF2/TF3 | 15 / 60 / 240 | שני הראשונים חובה, השלישי בונוס |
| HTF Bias | Agreement rule | TF1&TF2 | חובה שיסכימו / רוב / רק TF1 |
| Displacement | Min body ×ATR | 0.8 | סף גודל גוף |
| Displacement | Max SMT→Disp | 5 | חלון זמן |
| Structure | Requirement | MSS or CISD | MSS/CISD/שניהם |
| Structure | Require close | On | Wick בלבד אינו מספיק |
| FVG/IFVG | Min FVG ×ATR | 0.10 | סינון גאפ זניח |
| FVG/IFVG | Only from displacement | On | אזור כניסה רק מ‑Displacement |
| FVG/IFVG | Entry inside zone | Midpoint 50% | Proximal/Midpoint/Distal |
| Timing | Max wait retest | 15 | חלון Retest |
| Timing | Setup max age | 40 | פקיעה |
| Timing | Cooldown | 20 | מרווח בין סטאפים |
| Timing | Allow re‑entry | Off | כניסה חוזרת אחת |
| Stop | Method | Beyond sweep | מיקום הסטופ |
| Stop | Buffer ×ATR | 0.10 | מרווח ביטחון |
| TP | Min / Target / Max RR | 2 / 3 / 6 | יעדי סיכוי‑סיכון |
| Grading | Mode | Strict | Strict (A+) / Flexible |
| Grading | Min A+/A/B | 85 / 75 / 60 | ספי ציון |
| Sessions | Timezone | America/New_York | DST אוטומטי |
| Sessions | Windows | Asia/London/NY‑AM/NY‑PM | ניתן לעריכה מלאה |
| Sessions | Setups allowed in | NY AM + London | סינון סשן |
| Display | הצג/הסתר כל רכיב | On | Lines/Boxes/Labels/BG/Dashboard |
| Dashboard | Debug rows | Off | פירוט ניקוד וסיבת פסילה |

---

## 4. מערכת הדירוג (100 נקודות)

| קטגוריה | מקס' | לוגיקה |
|---|---|---|
| HTF Bias | 15 | 15m=+5, 1H=+7, 4H=+3 כשתואם לכיוון |
| Liquidity Location | 15 | PDH/PDL 15 · Asia 13 · London 11 · Equal 10 · Swing/Prev‑Sess 8 |
| Sweep Quality | 10 | Wick 10 · Close+Reclaim 7 · Any 3 |
| SMT Quality | 15 | לפי חשיבות הרמה שנשברה |
| Displacement | 10 | נר חזק + FVG 10 · עם FVG 8 · בלי 4 |
| Structure | 10 | MSS+CISD 10 · MSS 8 · CISD 7 |
| Entry Zone | 10 | IFVG 10 · FVG מ‑Displacement 8 · אחר 2 |
| Session Timing | 5 | NY KZ 5 · London 4 · אחר 2 |
| Risk/Reward | 5 | ≥3R → 5 · 2–2.99R → 3 |
| Confluence | 5 | Premium/Discount + Volume expansion + יעד נזילות אמיתי |

### קטגוריות
- **A+ (85–100):** כל תנאי החובה קיימים — SMT, Sweep, Displacement, MSS/CISD, FVG/IFVG, Retest, RR≥2, סשן פעיל, ולא נגד Bias חזק. **גם אם הציון ≥85, חוסר בתנאי חובה מוריד מ‑A+.**
- **A (75–84):** ליבה מלאה, חסר Confluence אחד (למשל FVG במקום IFVG, או רק 2R).
- **B (60–74):** אזהרה בלבד — SMT/Displacement/MSS חלשים או Bias מעורב.
- **C (<60):** לא מוצג כברירת מחדל (רק ב‑Debug).

**Strict Mode** מציג A+ בלבד. **Flexible Mode** מציג גם A/B לפי הספים והטוגלים.

---

## 5. Alerts

צרו Alert על האינדיקטור. תנאים זמינים דרך `alertcondition`:
`Bullish/Bearish SMT detected` · `A+/A Long/Short setup ready` · `Entry retest reached` · `Setup invalidated` · `TP1/TP2 reached` · `Stop reached`.

**ההתראה החשובה ביותר** היא כאשר הסטאפ הושלם והמחיר חזר לאזור הכניסה (`... setup ready` / `Setup entry triggered`).

בנוסף, קריאת `alert()` דינמית שולחת הודעה מלאה כשמופעל Retest. כדי לקבל אותה: ב‑Create Alert בחרו **Condition = "Any alert() function call"**. דוגמת הודעה:

```
A+ LONG | NQ/ES SMT | Score 91 | PDL Sweep | Entry 21450 | SL 21425 | TP 21525 | RR 3.0
```

---

## 6. מגבלות ידועות (וקירובים לפי מגבלות Pine)

Pine מודד רק OHLC + ווליום. לכן מושגים כמו *Institutional Order Flow / True Manipulation / Market‑Maker Intent* **אינם נמדדים ישירות** — האינדיקטור מזהה **פרוקסי מדיד** ולא "כוונה".

1. **התאמת SMT** נעשית על פיבוטי הגרף כעוגן זמן ודגימת הנכס המשווה באותו נר (קירוב מדיד, מתאים ל‑NQ/ES שנעים כמעט יחד). אין השוואת מחיר מוחלט — רק מבנה יחסי.
2. **Bias — Market Structure** ממומש כפרוקסי EMA (מחיר מול EMA + שיפוע), לא HH/HL מלא, כדי לשמור על יציבות ואי‑Repaint. EMA / Premium‑Discount / Manual ממומשים מלא.
3. **סטאפ פעיל אחד בכל רגע** — כדי לשמור סלקטיביות. בזמן שסטאפ מתפתח, סטאפ מנוגד לא ייזרע עד לפתרונו (עד `setupMaxAge`).
4. **Equal Highs/Lows** נגזרים משני פיבוטים סמוכים בטולרנס; אינם מתאפסים אוטומטית עד לזיהוי זוג חדש.
5. **Dashboard/Debug** משקפים את הסטאפ הפעיל הנוכחי; לאחר איפוס יוצגו ערכי "‑".
6. אין תלות ב‑UTC קבוע — הסשנים ב‑`America/New_York` וה‑DST מטופל אוטומטית.
7. דורש ווליום אמיתי (חוזים, לא CFD) לניקוד ה‑Confluence של ה‑Volume.

**האינדיקטור הוא כלי ניתוח בלבד ואינו ייעוץ השקעות.**

---

## 7. בדיקה ושיפור לאחר Backtest

1. **ידני על 1m/3m/5m** — ודאו שרקעי הסשנים מתחלפים בשעות ה‑NY הנכונות, ושרמות PDH/PDL/Asia/London משורטטות נכון.
2. **Repaint check** — הריצו על היסטוריה, ואז Replay: ודאו שהתוויות מופיעות רק בנר האישור ולא זזות אחורה.
3. **כיול Displacement** — אם יש מעט סטאפים, הקטינו `Min body ×ATR` ל‑0.6; אם רועש מדי, העלו ל‑1.0.
4. **חלונות זמן** — התאימו `Max SMT→Disp` ו‑`Max Displacement→MSS` לטיים‑פריים (קטן יותר → הגדילו מעט).
5. **RR ריאלי** — אם A+ נדיר מדי, בדקו אם היעד הקרוב לא מגיע ל‑2R; שקלו `Target selection = Fixed RR only` להשוואה.
6. **Bias** — השוו EMA מול Manual בימי חדשות; שקלו לרכך ל‑"Only TF1 decides" בטרנד חזק.
7. **סטטיסטיקה** — הפעילו `Debug` ורשמו Score/Grade/סיבת פסילה לאורך שבוע; חדדו את הספים לפי אחוזי הצלחה בפועל.
8. **שיקול דעת אנושי שנותר נדרש:** חדשות בעלות אימפקט, שינוי משטר תנודתיות, ורוחב Spread — אלה אינם נמדדים ויש לשקול ידנית לפני כניסה.
