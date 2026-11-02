# פרויקט הלכה ממקורה (Halakha MiMekora) — מהדורת שרת-לקוח מופרדת ומאובטחת (v12.2)

אתר חופשי לבירור ההלכה שבו **מערכת א' (מנוע ה-AI להבנת שאלות הלכתיות והיסק עקרוני)** כפופה ומחויבת באופן מוחלט ל**מערכת ב' (קרנל אכיפה דטרמיניסטי לנאמנות למסורת הפוסקים, מניעת תכנים זרים, מניעת הזיות AI ומניעת ביטחון יתר)** — מותאם ל-4 מסורות הפסיקה (ספרדי, מרוקאי, אשכנזי ותימני) ב-3 רמות החמרה (עיקר הדין, לכתחילה, שעת הדחק/בדיעבד).

* **אתר חי ב-GitHub Pages:** https://archimedesnsos.github.io/halakha-mimekora/
* **מאגר הקוד הציבורי ב-GitHub (ממשק לקוח נקי בלבד):** https://github.com/archimedesnsos/halakha-mimekora

---

## 1. ארכיטקטורת הפרדת שרת-לקוח (Zero-IP-Exposure) ושרת ה-Cloudflare Worker

כדי להגן באופן מוחלט על הקניין הרוחני (IP) של ארכיטקטורת ארכימדס ועל מפתח ה-API של המערכת (ולמנוע כל אפשרות של הנדסה לאחור מתוך הדפדפן או מתוך מאגר ה-GitHub הציבורי), המערכת הופרדה לשתי שכבות מבודדות:

1. **צד הלקוח הציבורי (`index.html` — 10.9KB בלבד, עולה ל-GitHub Pages):**
   * מכיל אך ורק את ממשק המשתמש (UI): תיבת השאלה, כפתורי בחירת 4 העדות, שאלות לדוגמה בלחיצה ותצוגת התשובה ומראי המקומות.
   * **0% חשיפת IP ופרטיות מלאה:** אין בקובץ זה שום לוגיקה של ארכימדס, שום פרומפטים, שום חוקי סינון, שום מאגר הלכתי ושום מפתח API. כמו כן הוסר כפתור ההגדרות (`⚙️`) וכתובת ה-Endpoint קודדה ב-Base64 כך שלא נחשף שום פרט מזהה בטקסט הקוד.
2. **צד השרת הסגור (`archimedes_halakha_backend_worker.js` — v4.0, רץ על Cloudflare Workers):**
   * יושב בשרת ענן מאובטח (`muddy-mode-halakha-kernel`) ב-Cloudflare Workers ומכיל את כל המוח של מערכת א' ומערכת ב':
     * **שלב 1 — נרמול ובידוד שלילה (`normalizeQuery`):** תיקון שגיאות כתיב, תרגום 32 מונחי סלנג ללשון השו"ע ובידוד מילים בשלילה (`¬Negated`).
     * **שלב 2 — שער סינון תחום וחומת אש אורתודוקסית (`checkFirewallAndDomainGate`):** חסימה דטרמיניסטית של שאלות חולין (`OUT_OF_SCOPE`), שאלות היסטוריות/ביוגרפיות, תכנים נוצריים/משיחיים (`BLOCKED_FOREIGN_THEOLOGY`) ופסיקות רפורמיות/קונסרבטיביות (`BLOCKED_NON_ORTHODOX`), והעברה לשאלת חכם אנושי (`POSEK_ESCALATION`) במראות טהרה ופיקוח נפש.
     * **שלב 3 — שליפת RAG קנונית מ-Sefaria API (`fetchSefariaCanonicalRAG`):** שליפה חיה מתוך השולחן ערוך, המשנה ברורה והרמב"ם במאגר ספריא.
     * **שלב 4 — לולאת משוב מתקנת עד 3 ניסיונות מול Gemini ואכיפת מערכת ב' (`auditDraftBySystemB`):** בדיקת תקינות מבנה הפסק ל-4 העדות ו-3 רמות ההחמרה, נעילת ציטוט המקור בחתימת `SHA-256`, ואכיפת הערת ענווה הלכתית למניעת ביטחון יתר.

---

## 2. אבולוציית הטיפול במפתחות ה-API של Google Gemini ופתרון הכשלים ההנדסיים

במהלך פיתוח הפרויקט נפתרו מן השורש שורה של אתגרים טכניים מול שרתי Google Gemini ו-Cloudflare:

1. **מעבר ממפתח חשוף בצד-לקוח לסוד מוצפן בשרת (`GEMINI_API_KEY` Secret):**
   * במקום שהמשתמש יזין מפתח בדפדפן או שהמפתח ישב בקוד גלוי, המפתח הוגדר כמשתנה סביבה מוצפן (`Secret`) בתוך הגדרות ה-Cloudflare Worker (`Settings -> Variables and Secrets`).
2. **תמיכה כפולה במפתחות Google AI Studio (`AIza...`) ובמפתחות Google Cloud / Vertex AI Express בני 53 תווים (`AQ...`):**
   * פונקציית `callGeminiWithAutoFallback` בשרת מזהה ותומכת אוטומטית הן בנקודת הקצה הרגילה (`generativelanguage.googleapis.com`) והן בנקודת הקצה של Vertex AI (`aiplatform.googleapis.com/v1/publishers/google/models`).
3. **גילוי מודלים דינמי (`discoverAvailableGeminiModels` מול `ModelService.ListModels`):**
   * לאחר שגוגל הסירה מודלים ישנים (כגון `gemini-1.5-flash` שהחזיר שגיאת `404 Not Found`), השרת מבצע שאילתת `ListModels` בזמן אמת מול `v1beta` ו-`v1`, מאתר את המודלים הפעילים בחשבון (כגון `gemini-2.5-flash`, `gemini-2.5-flash-lite`, `gemini-3-flash-preview`), שומר אותם במטמון הזיכרון של השרת, וכולל מנגנון Fallback אוטומטי ללא `responseMimeType` במידת הצורך.
4. **מנקה ומתקן JSON אוטומטי לעברית תורנית (`extractCleanJson` + `normalizeAndHealDraft`):**
   * בעברית תורנית קיימים ראשי תיבות עם גרשיים כפולים באמצע מילה (`שו"ע`, `רמ"א`, `או"ח`, `תרנ"ד`). כאשר מודל השפה פלט אותם בתוך מחרוזת JSON, הפקודה `JSON.parse()` קרסה וגרמה למערכת ליפול בטעות לתבנית הבהרה קבועה של שאלות מטבח (*"האם הכלי בן יומו"*).
   * **הפתרון ב-v4.0:** הפרומפט מנחה שימוש בגרש בודד (`שו'ע`, `רמ'א`), הפונקציה `extractCleanJson` מנקה עטיפות Markdown (` ```json `) ומתקנת בביטוי רגולרי (`replace(/([א-ת])"([א-ת])/g, "$1'$2")`) כל מרכאות כפולות שנותרו בתוך מילים עבריות, ו-`normalizeAndHealDraft` משלימה אוטומטית כל שדה חסר ב-4 העדות. כמו כן, תבנית ההבהרה הותאמה דינמית לנושא השאלה.
5. **חיסון כתובות URL מפני שיבוש אוטומטי בהעתקה:**
   * כל כתובות ה-URL בקוד השרת נבנות בחיבור מחרוזות מפוצל (`"https://" + "generativelanguage..."`) ללא Template Literals (`${...}`), כדי ששום ממשק צ'אט או עורך Markdown לא יהפוך אותן לקישורים שבורים בעת העתקה.

---

## 3. צינור הפרסום והסנכרון הכפול האוטומטי (`publish_to_github.bat` — v12.2)

הסקריפט `publish_to_github.bat` שבתיקיית הדרייב מסנכרן בלחיצה כפולה אחת גם את GitHub Pages וגם את ה-Cloudflare Worker **ללא צורך ב-Node.js / Wrangler**, באמצעות PowerShell ו-`.NET HttpClient` המובנים ב-Windows:

1. **שלבים 1–4 (סנכרון נקי ל-GitHub Pages מחוץ לדרייב):**
   * מוחק ומקים מחדש את תיקיית הביניים המקומית `%LOCALAPPDATA%\HalakhaMiMekora_GitStage`.
   * מעתיק אליה **אך ורק את 4 הקבצים הציבוריים** (`index.html`, `README.md`, `.nojekyll`, `.gitignore`).
   * מריץ `git checkout --orphan main` ו-`git push -u origin main --force` כך שכל היסטוריית הקומיטים הישנה ב-GitHub נמחקת לחלוטין ואף קובץ פיתוח אינו נחשף בהיסטוריה.
2. **שלב 5 (העלאה בינארית ישירה ל-Cloudflare Worker באמצעות `.NET MultipartFormDataContent`):**
   * קורא את אסימון ה-API של Cloudflare מהקובץ המקומי `%LOCALAPPDATA%\halakha_cf_api_token.txt` (שנשמר מחוץ לדרייב ומחוץ ל-GitHub).
   * אורז את `metadata.json` (`main_module: "worker.js"`) ואת הקובץ `archimedes_halakha_backend_worker.js` באמצעות המחלקה `System.Net.Http.MultipartFormDataContent` של `.NET` (הפותר לחלוטין שגיאות ציטוט בשורת הפקודה ושגיאת `multipart: NextPart: EOF`) ומשגר אותם בבקשת `PUT` ישירה ל-REST API של Cloudflare.

---

## 4. רשימת הקבצים בתיקיית הפרויקט ב-Google Drive

* **`index.html`** — ממשק הלקוח הציבורי הנקי (10.9KB) העולה ל-GitHub Pages.
* **`archimedes_halakha_backend_worker.js`** — קוד השרת המאובטח (v4.0) העולה ל-Cloudflare Worker.
* **`halakha_mimekora.html`** — קובץ מעבדת הפיתוח וה-CI/CD המקומית (1.5MB, כולל 173/173 בדיקות אוטומטיות וסטודיו הרחבת המאגר — נשאר בדרייב הפרטי בלבד).
* **`publish_to_github.bat`** — סקריפט הסנכרון הכפול (v12.2) ל-GitHub Pages ול-Cloudflare Worker.
* **`README.md` + `Halakha_MiMekora_Work_Plan.pdf` + תיקיית `DOCS`** — תיעוד הפרויקט ומסמכי האפיון וההוכחה ההנדסית.
* **`_backup_archive (גיבוי וארכיון גרסאות)`** — ארכיון מסודר של כל הגרסאות הקודמות.
