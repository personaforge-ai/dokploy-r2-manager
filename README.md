# Dokploy — Cloudflare R2 / S3 Web Manager

حزمة **Docker Compose** جاهزة للنشر على [Dokploy](https://dokploy.com) لتوفير واجهة ويب لإدارة **Cloudflare R2** (وأي endpoint متوافق مع S3) بدون AWS وبدون تشغيل MinIO كخدمة.

---

## 1. المشروع المختار ولماذا

**[Storage UI](https://github.com/hahahumble/storageui)** (`hahahumble/storageui`)

| معيار | Storage UI | cloudlena/s3manager | subratomandal/s3explorer |
|--------|------------|---------------------|----------------------------|
| GitHub ⭐ (تقريبي) | ~174 | ~1058 | ~30 |
| آخر نشاط (2026-09) | نشط | نشط | نشط |
| Docker image رسمية | `hahahumble/storageui` | `cloudlena/s3manager` | `ghcr.io/subratomandal/s3explorer` |
| Cloudflare R2 | `STORAGE_n_PROVIDER=r2` + `ACCOUNT_ID` | endpoint يدوي | موثّق في README |
| تسجيل دخول التطبيق | `AUTH_USERNAME` / `AUTH_PASSWORD` | **لا** (من يصل للـ URL يستخدم صلاحيات S3 من الـ env) | `APP_PASSWORD` + جلسات |
| Multi-bucket | `STORAGE_1`…`STORAGE_50` + من الـ UI | كل الـ buckets في الحساب | حتى 100 اتصال (SQLite) |
| Upload / Download / Delete | نعم | نعم | نعم + zip |
| Rename / folders / search | مدعوم في المنتج | محدود مقارنةً | rename + folders |
| Read-only | `STORAGE_n_READ_ONLY=true` | لا على مستوى التطبيق | عبر صلاحيات token |
| Database / volume | **لا** عند ضبط R2 من env فقط | لا | **نعم** — `/data` SQLite |
| ترخيص | Apache-2.0 | Apache-2.0 | MIT |
| مناسب Dokploy | container واحد، port 3000، non-root `bun` | بسيط لكن **بدون login** | أقوى أمانًا لكن يعتمد volume + إعداد اتصالات من الـ UI |

**الاختيار:** Storage UI لأنه يجمع **login للتطبيق**، **دعم R2 صريح في التوثيق**، **read-only per bucket**، **compose بسيط بدون DB**، وصورة Docker مثبتة بإصدار (`0.2.2`).  
`s3manager` أشهر لكنه لا يحقق شرط «تسجيل دخول آمن» على مستوى الواجهة.  
`s3explorer` ممتاز للأمان المتقدم لكنه يحتاج volume دائم ويُ configure الاتصال غالبًا عبر الـ UI بعد التشغيل.

---

## 2. روابط رسمية

- **GitHub:** https://github.com/hahahumble/storageui  
- **Docker Hub:** https://hub.docker.com/r/hahahumble/storageui  
- **توثيق المتغيرات:** https://storageui.dev/docs/environment-variables  

**الصورة المثبتة في هذا المستودع:** `hahahumble/storageui:0.2.2`

---

## 3. إنشاء Cloudflare R2 API Token

1. [Cloudflare Dashboard](https://dash.cloudflare.com) → **R2 Object Storage**.  
2. **Manage R2 API Tokens** → **Create API token**.  
3. اختر نطاقًا:
   - **Admin Read & Write** — للرفع والحذف والتعديل (Production كامل).  
   - **Object Read only** — للتصفح والتحميل فقط (مع `STORAGE_1_READ_ONLY=true` على التطبيق).  
4. يمكن تقييد الـ token على **bucket واحد** إن أردت.  
5. احفظ **Access Key ID** و **Secret Access Key** (Secret يظهر مرة واحدة).  
6. **Account ID** من Overview في R2 (لـ `STORAGE_1_ACCOUNT_ID`).

Endpoint الافتراضي لـ R2:

```text
https://<ACCOUNT_ID>.r2.cloudflarestorage.com
```

مع `PROVIDER=r2` يكفي `STORAGE_1_ACCOUNT_ID`؛ أو استخدم `STORAGE_1_ENDPOINT` يدويًا.

---

## 4. الصلاحيات المطلوبة (R2)

| عمل في الواجهة | صلاحية Token مقترحة |
|----------------|---------------------|
| Browse + Download | Object Read (أو Read & Write) |
| Upload / Delete / Rename | Object Read & Write على ذلك الـ bucket |
| إنشاء bucket من الواجهة | Admin (Storage UI قد لا يحتاجها إن bucket واحد مُعرّف مسبقًا) |

**أقل امتياز:** token مقيد على bucket واحد + Object Read & Write فقط إن كنت تثق بالمستخدمين بعد login التطبيق.

---

## 5. Environment Variables في Dokploy

1. Dokploy → **Projects** → **Create Service** → **Compose**.  
2. **Provider:** GitHub → هذا المستودع (انظر §7).  
3. **Compose path:** `docker-compose.yml`  
4. **Environment:** انسخ من [`.env.example`](./.env.example) وعبّئ القيم (Dokploy يكتب `code/.env` بجانب الـ compose).

**إلزامي للإنتاج:**

| Variable | الغرض |
|----------|--------|
| `NEXT_PUBLIC_APP_URL` | `https://your-domain.example` (نفس الدomain في Dokploy) |
| `AUTH_USERNAME` / `AUTH_PASSWORD` | login الواجهة |
| `AUTH_SECRET` | `openssl rand -hex 32` — جلسات آمنة |
| `STORAGE_1_PROVIDER` | `r2` |
| `STORAGE_1_ACCOUNT_ID` | Cloudflare Account ID |
| `STORAGE_1_BUCKET` | اسم الـ bucket |
| `STORAGE_1_ACCESS_KEY_ID` | R2 access key |
| `STORAGE_1_SECRET_ACCESS_KEY` | R2 secret |

**تعيين أسماء S3 الشائعة → Storage UI:**

| S3 (دليل عام) | Storage UI |
|---------------|------------|
| `S3_BUCKET` | `STORAGE_1_BUCKET` |
| `S3_ACCESS_KEY_ID` | `STORAGE_1_ACCESS_KEY_ID` |
| `S3_SECRET_ACCESS_KEY` | `STORAGE_1_SECRET_ACCESS_KEY` |
| `S3_REGION=auto` | غير مطلوب لـ `provider=r2` |
| `S3_ENDPOINT` | `STORAGE_1_ENDPOINT` أو `ACCOUNT_ID` |

لا تضع credentials في Git — فقط في Dokploy Environment.

---

## 6. Deploy على Dokploy

1. ادفع هذا المجلد إلى مستودع GitHub (مثلاً `personaforge-ai/dokploy-r2-manager` أو monorepo subfolder — إن كان subfolder، اضبط **Root Directory** / مسار الـ compose حسب Dokploy).  
2. Compose service → **Deploy**.  
3. تأكد أن الخدمة **healthy** (healthcheck على port 3000).  
4. **لا** hacen expose عام على 3000 إن Dokploy يوجّه Traefik فقط — الـ compose يستخدم `expose: "3000"`؛ Dokploy يربط الدomain بالحاوية.

---

## 7. ربط Domain

1. Dokploy → الخدمة → **Domains** → أضف `r2-manager.example.com`.  
2. فعّل **HTTPS** (Let's Encrypt عبر Traefik).  
3. حدّث `NEXT_PUBLIC_APP_URL=https://r2-manager.example.com` وأعد **Deploy** (قيمة `NEXT_PUBLIC_*` تُقرأ عند التشغيل؛ راجع توثيق Storage UI إن غيّرت الدomain لاحقًا).

---

## 8. اختبار Upload / Download

1. افتح الدomain → سجّل دخول بـ `AUTH_USERNAME` / `AUTH_PASSWORD`.  
2. اختر **R2 Production** (أو الاسم في `STORAGE_1_NAME`) من الشريط الجانبي.  
3. **Upload** ملفًا صغيرًا → تحقق في Cloudflare R2 dashboard.  
4. **Download** نفس الملف من الواجهة.  
5. **Delete** اختياري على ملف تجريبي.

---

## 9. تغيير الـ Bucket

**طريقة 1 — env (يتطلب redeploy):**  
عدّل `STORAGE_1_BUCKET` (و مفاتيح token إن لزم) في Dokploy Environment → Deploy.

**طريقة 2 — bucket ثانٍ:**  
فعّل `STORAGE_2_*` في `.env.example` (نفس الشكل) → Deploy.

**طريقة 3 — من الواجهة:**  
Storage UI يسمح بإضافة اتصالات من الـ UI (credentials تبقى على السيرver). في الإنتاج، يُفضّل الاكتفاء بـ env للاتصالات الثابتة.

---

## 10. Read-only

```env
STORAGE_1_READ_ONLY=true
```

مع R2 token **Object Read** فقط لطبقة دفاع مزدوجة.

---

## 11. اعتبارات أمنية

- **لا** تنشر `.env` ولا R2 secrets في Git.  
- استخدم **AUTH_*** قويًا + **AUTH_SECRET** عشوائي.  
- قيّد R2 token على bucket واحد وأقل صلاحية ممكنة.  
- الواجهة خلف HTTPS فقط (Dokploy/Traefik).  
- اختياري: **Cloudflare Turnstile** على صفحة login (`TURNSTILE_SITE_KEY` + `TURNSTILE_SECRET_KEY`).  
- `ALLOW_PRIVATE_ENDPOINTS` — اتركه غير مفعّل إلا إن احتجت endpoints خاصة من الـ UI.  
- من يملك login التطبيق يملك ما يسمح به token R2 — عالج ذلك بـ RBAC (حسابات منفصلة / tokens منفصلة) إن لزم.

**ملاحظة:** Storage UI لا يخزّن credentials في المتصفح؛ الاتصالات من env تُحمّل على السيرver فقط ([التوثيق](https://storageui.dev/docs/environment-variables)).

---

## 12. Reverse proxy

لا Nginx في هذا الـ compose. Dokploy (Traefik) يتولى TLS والدomain؛ الحاوية تستمع على `3000` داخليًا.

---

## 13. Persistence

لا volume مطلوب عند استخدام R2 عبر env فقط. إذا أضفت اتصالات من الـ UI، قد تحتاج مراجعة توثيق Storage UI للإصدارات الأحدث — هذا الـ stack مُحسّن لـ **اتصال R2 ثابت من Environment**.

---

## هيكل المستودع

```text
dokploy-r2-manager/
├── docker-compose.yml
├── .env.example
├── .gitignore
└── README.md
```

---

## ترقية الصورة

راجع [tags على Docker Hub](https://hub.docker.com/r/hahahumble/storageui/tags)، غيّر `image:` في `docker-compose.yml` إلى إصدار جديد، اختبر على staging، ثم Deploy.
