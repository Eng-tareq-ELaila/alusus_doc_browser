# 📚 DocBrowser

تطبيق ويب متكامل (Full-Stack) مبني بلغة **Alusus** يتيح تحميل ملفات Markdown من روابط عامة، تخزينها في قاعدة بيانات SQLite، البحث فيها، وعرضها كـ HTML داخل المتصفح.

---

## ✨ المميزات

- **تحميل وثائق Markdown** من أي رابط عام (مثل ملفات GitHub Raw)
- **تخزين دائم** في SQLite بدون إعداد خادم قواعد بيانات خارجي
- **بحث فوري** في العنوان والمحتوى
- **عرض HTML** مُوَلَّد تلقائياً من Markdown مع جدول محتويات جانبي
- **لوحة مدير** محمية بكلمة مرور لإضافة الوثائق وحذفها
- **واجهة نظيفة** مبنية بـ WebPlatform بدون HTML/CSS خارجي

---

## 🏗️ هيكل المشروع

```
DocBrowser/
├── Main.alusus              ← نقطة الدخول: يستورد كل الوحدات ويشغّل الخادم
│
├── db/
│   └── Database.alusus      ← طبقة قاعدة البيانات (SQLite)
│
├── core/
│   └── Core.alusus          ← وظائف مشتركة (JSON، Requests، Markdown، Session)
│
├── api/
│   └── Api.alusus           ← نقاط نهاية REST API (Backend Endpoints)
│
└── ui/
    ├── BrowsePage.alusus    ← صفحة التصفح العامة  GET /
    └── AdminPage.alusus     ← لوحة المدير         GET /admin
```

---

## 📁 وصف الوحدات

### `db/Database.alusus` — طبقة قاعدة البيانات
تُهيّئ قاعدة البيانات وتُنشئ جدول `documents` وتوفّر:

| الدالة | الوصف |
|---|---|
| `initDb()` | تهيئة قاعدة البيانات وإنشاء الجدول |
| `sqlEsc(s)` | هروب اقتباسات SQL لمنع Injection |
| `upsertDoc(...)` | إضافة وثيقة أو تحديثها إذا كانت موجودة |
| `deleteDoc(id)` | حذف وثيقة بالمعرّف |
| `searchDocs(keyword)` | بحث بالكلمة المفتاحية أو جلب كل الوثائق |
| `fetchDoc(id)` | جلب وثيقة كاملة مع محتواها |

---

### `core/Core.alusus` — الوظائف المشتركة
تحتوي على أربعة أقسام:

**JSON Helpers** — بناء ردود HTTP بصيغة JSON آمنة:
- `sendOk(conn, data)` — رد 200 ناجح
- `sendErr(conn, msg)` — رد 400 بخطأ
- `rowsToJson(rows)` — تحويل صفوف SQLite إلى JSON array

**Request Helpers** — معالجة طلبات HTTP:
- `readBody(conn)` — قراءة body الطلب كاملاً
- `jStr(obj, key)` — استخراج نص من كائن Json مع إزالة الاقتباسات
- `qsParam(qs, key)` — استخراج معامل من query string
- `unquote(s)` — إزالة الاقتباسات المحيطة بقيمة rawValue

**Markdown Helpers** — معالجة ملفات Markdown:
- `extractTitle(md)` — استخراج أول عنوان `# H1` كعنوان للوثيقة
- `buildToc(md)` — بناء جدول محتويات JSON من H1/H2/H3

**Session Management** — إدارة جلسة المدير:
- `newSession()` — إنشاء token جديد بناءً على الوقت الحالي
- `clearSession()` — مسح الجلسة (تسجيل خروج)

---

### `api/Api.alusus` — REST API
| الطريقة | المسار | الوصف |
|---|---|---|
| `POST` | `/api/login` | تسجيل دخول المدير |
| `POST` | `/api/logout` | تسجيل خروج |
| `POST` | `/api/docs` | تحميل وثيقة Markdown من رابط URL |
| `DELETE` | `/api/docs` | حذف وثيقة بالمعرّف |
| `GET` | `/api/docs?q=keyword` | بحث أو جلب كل الوثائق |
| `GET` | `/api/docs/view?id=...` | جلب وثيقة وتحويلها لـ HTML |

---

### `ui/BrowsePage.alusus` — صفحة التصفح  `GET /`
الصفحة العامة التي تعرض:
- شريط تنقل علوي مع رابط لوحة المدير
- حقل بحث فوري (live search أثناء الكتابة)
- قائمة بطاقات تعرض عنوان الوثيقة وتاريخ الإضافة
- عند النقر على بطاقة: يُعرَض المحتوى HTML مع جدول محتويات جانبي
- زر "← Back to list" للرجوع لقائمة الوثائق

---

### `ui/AdminPage.alusus` — لوحة المدير  `GET /admin`
تعرض نموذج تسجيل دخول أولاً، وبعده:
- **بطاقة تحميل وثيقة**: حقل URL + زر Load
- **قائمة الوثائق المُحمَّلة** مع زر حذف لكل وثيقة
- رسائل حالة (نجاح/فشل) ملوّنة
- زر Logout في الهيدر

---

## 🔄 تدفق البيانات

```
المستخدم يكتب URL
        ↓
   POST /api/docs
        ↓
  Sle.Net.Request.get()   ← تنزيل ملف Markdown من الإنترنت
        ↓
  extractTitle()           ← استخراج العنوان من أول H1
        ↓
  upsertDoc()              ← حفظ في SQLite
        ↓
  رد JSON {ok, id, title}
        ↓
  refreshList()            ← تحديث قائمة الوثائق في UI
```

```
المستخدم ينقر وثيقة
        ↓
  GET /api/docs/view?id=…
        ↓
  fetchDoc()               ← جلب من SQLite
        ↓
  MarkdownTranslator       ← تحويل Markdown → HTML
  buildToc()               ← بناء جدول محتويات
        ↓
  رد JSON {html, toc, …}
        ↓
  HtmlText(html)           ← عرض في الواجهة
```

---

## 🚀 تشغيل التطبيق

```bash
alusus Main.alusus
```

ثم افتح المتصفح على:
- `http://localhost:8080` — صفحة التصفح
- `http://localhost:8080/admin` — لوحة المدير

---

## 🔐 بيانات تسجيل الدخول الافتراضية

| المستخدم | كلمة المرور |
|---|---|
| `admin` | `12345` |

> ⚠️ يُنصح بتغيير هذه البيانات في بيئة الإنتاج وتخزينها في متغيرات البيئة.

---

## 🛠️ المكتبات المستخدمة

| المكتبة | الاستخدام |
|---|---|
| `WebPlatform` | خادم HTTP + واجهة المستخدم |
| `Sqlite` | قاعدة البيانات المحلية |
| `Json` | تحليل وبناء JSON |
| `MarkdownTranslator` | تحويل Markdown إلى HTML |
| `Sle (Srl/Net)` | تنزيل ملفات من الإنترنت |
| `Srl/StringBuilder` | بناء نصوص JSON بشكل آمن |

---

## 📝 ملاحظات تقنية

- **الـ URL يُستخدم كمعرّف فريد** للوثيقة — تحميل نفس الرابط مرتين يُحدّث الوثيقة.
- **الجلسة في الذاكرة فقط** — تنتهي عند إعادة تشغيل الخادم.
- **حجم body الطلب** محدود بـ 64KB.
- **بحث LIKE** في SQLite غير حساس لحالة الأحرف للحروف الإنجليزية.

---

*مبني بـ [Alusus Programming Language](https://alusus.org)*
# alusus_doc_browser
