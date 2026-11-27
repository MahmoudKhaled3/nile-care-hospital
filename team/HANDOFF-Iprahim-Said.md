# Nile Care - تقرير تسليم عضو الفريق (`Iprahim Said`)

| الحقل | القيمة |
|---|---|
| الاسم والـbrief | Iprahim Said — `team/03-Iprahim-Said.md` (Azure Disaster Recovery & Cloud Security / B6) |
| تاريخ التسليم ونسخة الملفات أو commit | 2026-11-27 — Pre-flight package (`work/iprahim-azure-preflight`, pre-provisioning) |
| البيئة المستخدمة | Windows + Azure CLI `2.93.0` + `Azure for Students` (`$100` credit, 365 days) |
| الوقت المتوقع للباقي | `<PENDING>` (يعتمد على موافقة تغيير الـ region/SKU واعتماد Gate 3) |

| البند المطلوب | الحالة | المطلوب يحصل | اللي حصل فعلاً | ملف التسليم أو الدليل | الناقص أو الغلط ومين هيعالجه |
|---|---|---|---|---|---|
| 1. B6 §1 Pre-flight (اشتراك، Region، SKU، Quota، Public IP، تكلفة) | `DONE` | فحص اشتراك `Azure for Students` وتوفر `uaenorth` و`Standard_B1s` والـ quota والتكلفة المتوقعة قبل إنشاء أي موارد | الاشتراك اتفعّل (`$100`), والـ providers اتسجلت (`Registered`). في `uaenorth` كل أحجام `Standard_B*` (منها `Standard_B1s`) عليها `NotAvailableForSubscription`. تم فحص بدائل واتلاقى `swedencentral` متاح بدون قيود على `Standard_B2ats_v2` (`$0.00972/hr`) وبـ quota كافية (`6` vCPUs و`3` Standard Public IP). لم يتم إنشاء أي موارد (`$0.00` مصروف). | `team/azure-preflight.md` | مطلوب موافقة **محمود** على تعديل التصميم إلى `swedencentral` + `Standard_B2ats_v2` قبل التنفيذ |
| 2. إنشاء `nilecare-dr`, `vnet-dr` (`172.16.0.0/16`), `snet-dr` (`172.16.10.0/24`), `nsg-dr`, `cloud-dr` | `BLOCKED` | بناء الشبكة و`cloud-dr` بعد موافقة الـ pre-flight وGate 3 | متوقف بناءً على Stop condition (`Standard_B1s` في `uaenorth` غير متاح للاشتراك) وبانتظار Gate 3 | `<PENDING>` (`azure-resources.txt`) | موافقة **محمود** على `swedencentral` + `Standard_B2ats_v2` واكتمال **Gate 3** |
| 3. إعداد Ubuntu 24.04 + strongSwan + MariaDB replica + restic + Blob immutability | `NOT RUN` | تجهيز السيرفر والـ VPN والـ replica والنسخ المحمي | `<PENDING>` | `<PENDING>` | يتنفذ بعد بند 2 ومع **يوسف** (`emr01`) |
| 4. Azure Monitor + Budget + Deallocate | `NOT RUN` | ضبط التنبيهات وإغلاق الـ VM خارج وقت الاختبار | `<PENDING>` | `<PENDING>` | يتنفذ بعد إنشاء الموارد |
| 5. Gate 4 (IPsec + MariaDB) وGate 5 (Wazuh agent) | `NOT RUN` | ربط النفق مع `fw01` والـ agent مع `siem01` | `<PENDING>` | `<PENDING>` | تنسيق مع **محمود** و**يوسف** و**شاكر** بعد Gate 3 |

---

## خطوات إعادة التنفيذ

1. استخدام `Azure CLI 2.93.0` وتسجيل الدخول على اشتراك `Azure for Students`.
2. التأكد من تسجيل `Microsoft.Compute`, `Microsoft.Network`, `Microsoft.Storage` (`Registered`).
3. فحص quotas وSKUs في `uaenorth` و`swedencentral` عبر CLI / Compute Resource SKUs REST API والأسعار عبر Azure Retail Prices API (التفاصيل والأوامر المنقحة في `team/azure-preflight.md`).
4. **فرق عن التصميم في `A-design-package.md` / `B6-cloud.md` (مقترح بانتظار الاعتماد):**
   - `uaenorth` غير متاح لأي حجم `Standard_B*` على اشتراك `Azure for Students` الحالي (`NotAvailableForSubscription`).
   - البديل المختبر والمتاح بدون قيود: region **`swedencentral`** وحجم **`Standard_B2ats_v2`** (2 vCPU, 1 GB RAM, `$0.00972/hr`).

---

## نقطة الدمج

| السؤال | الإجابة |
|---|---|
| مين هيستلم ومحتاج يبدأ بإيه؟ | **محمود**: مراجعة `team/azure-preflight.md` واعتماد استخدام `swedencentral` + `Standard_B2ats_v2` (أو تحديد بديل). **شاكر**: توثيق الملاحظة المبدئية لـ Ch. 6 / Ch. 8. |
| أي Gate وفصل أو Figure هيتأثروا؟ | **Gate 4**, Chapter 6 (`§6.1`, `§6.2`, `§6.6`), Figure 6.1, وDeliverable D (PDPL cross-border section — بيانات اصطناعية فقط). |
| فين نلاقي نسخة سابقة لو التحديث فشل؟ | لم تُنشأ أي موارد على Azure بعد؛ الريبو على `main` كما هو. |
| شاكر. هل الأدلة اتفتحت واتطابقت مع البنود؟ | `<PENDING>` |
| محمود. هل المخرجات متوافقة مع التصميم واتجربت في بيئتها المطلوبة؟ | `<PENDING>` |
| القرار وتاريخه | `<PENDING>` |
