# المرحلة 1 — تقرير مراجعة المعمارية والأمن وتصميم قاعدة البيانات

- **الفرع:** `docs/phase-1-architecture-security-design`
- **الأساس:** `AGENTS.md` §§3–5, 6, 13–16، وحالة الشجرة عند `57790b25edb9fe8f37c47eada4f03312bbf43af1`.
- **الحالة:** تسليم وثائقي مقترح؛ بانتظار اعتماد المستخدم. لا كود تطبيق ولا migrations.

## 1. ملخص ما أُنجز

أُعد ERD مفاهيمي يشمل الجداول المطلوبة، مع composite FKs إلزامية، إصدارات actions وموافقات متعددة، ومصفوفة RLS لكل جدول.
وُثقت Multi-Tenancy/RBAC، عزل LangGraph checkpoints كـ `NOT VERIFIED`، تصميم LLM/Action Gateways، ونموذج التهديدات.
أضيفت ADR-0001 إلى ADR-0005 وتقريرا المرحلة 0 والمرحلة 1؛ القرارات المفتوحة بقيت دون حسم.

## 2. النتائج والتصنيفات

| التصنيف | النتيجة | الدليل |
|---|---|---|
| `PASS` | ERD يتضمن organizations/memberships/projects ومحتوى/ذاكرة وplans/subscriptions وorg_budgets/quotas وinvitations وplatform_admins/impersonation وusage/integrations/actions/versions/approvals/audit وRBAC | `docs/architecture/data-model-multitenancy-rbac-rls.md` قسم ERD والجداول |
| `PASS` | علاقات parent-child tenant محددة كـ composite FK `(org_id,parent_id)` إلى `UNIQUE(org_id,id)` في كل parent، دون علاقات منطقية بديلة | الملف نفسه، «قواعد FK المركب الإلزامية» |
| `PASS` | `action_versions` immutable؛ snapshot/hash يشملان `scheduled_time` وtarget account؛ approvals متعددة مربوطة بإصدار محدد | ERD والـ gateway doc وADR-0003 |
| `PASS` | جدول RLS يغطي كل الجداول المذكورة ويكتب صراحةً «ممنوع» في UPDATE/DELETE لـ `audit_events`, `rbac_permissions`, `role_permissions` | مصفوفة RLS في `docs/architecture/data-model-multitenancy-rbac-rls.md` |
| `PASS` | صُممت Cross-Tenant tests على CRUD وDB/API وpool/Storage/Realtime/Cache/Queue/search وLangGraph عند توفرها | الملف نفسه، قسم Cross-Tenant test plan |
| `PASS` | خيارات Supabase schema exposure vs RLS-only، SSE vs Realtime، ودالة واحدة محتملة `SECURITY DEFINER` عُرضت دون حسم | `docs/adr/0005-supabase-exposure-realtime-and-membership-function.md` |
| `PASS` | threat model يتناول XSS وعرض output وتسريب البيانات، وسباق الميزانية وحجز reservation ثم settlement | `docs/architecture/security-threat-model.md` و`docs/architecture/llm-and-action-gateways.md` |
| `PASS` | خيارات موافقة صاحب الطلب على L2/L3 معروضة دون اختيار | `docs/architecture/data-model-multitenancy-rbac-rls.md` قسم RBAC؛ gateway doc |
| `PASS` | أُنشئ تقرير المرحلة 0، مع توضيح أن التقرير لم يكن موجوداً سابقاً وأنه موثق بأثر رجعي من commit معروف | `docs/audit/phase-0-read-only-audit.md` |
| `WARNING` | التصاميم مفاهيمية ولا يوجد schema فعلي أو migrations؛ المصادقة/RLS/قيود FK لم تُنفذ | المستودع عند commit الأساس لا يحوي تطبيقاً أو قاعدة أو migrations |
| `WARNING` | تقرير المرحلة 0 الأصلي لم يوجد على الفرع/الشجرة الأساسية؛ التقرير المضاف يثبت فقط الملفات التي ظهرت في commit المحدد | `docs/audit/phase-0-read-only-audit.md`؛ Git tree commit الأساس |
| `NOT VERIFIED` | واجهات LangGraph/checkpointer وعزل checkpoints بين المؤسسات لم تُراجع رسمياً أو تختبر | ERD doc وgateway doc يعلّمانها بوضوح `NOT VERIFIED` |
| `NOT VERIFIED` | تفاصيل Supabase Data API وRLS/Realtime/Storage/pooling/grants لم تتحقق من الوثائق الرسمية الحالية أو في بيئة فعلية | ADR-0002 وADR-0005؛ لا مشروع Supabase قائم |
| `NOT VERIFIED` | لا توجد اختبارات فعلية، بما فيها RLS/Cross-Tenant/XSS/budget reservation/idempotency/security | لا تطبيق أو test suite أو بيئة DB حالية |
| `NOT VERIFIED` | قرارات القسم 16، ومنها Data Residency والتشفير/provider/payment/queue/observability/email/retention، لم تُحسم | ADR-0004 يعرض خيارات ويؤجلها للمرحلة 2 |

## 3. قرارات مطلوبة من المستخدم

1. اعتماد أو طلب تعديل ERD/Multi-Tenancy/RBAC ومصفوفة RLS، مع العلم أنها design proposals وليست سياسات منشورة.
2. اختيار اتجاه ADR-0005 لاحقاً بعد مراجعة الوثائق الرسمية: Data API exposure، SSE/Realtime، وهل توجد حاجة واستثناء مقبول لدالة `SECURITY DEFINER` واحدة.
3. تحديد سياسة self-approval لـ L2/L3؛ الخيارات موثقة دون حسم.
4. تأكيد إغلاق المرحلة 1 أو إرجاع ملاحظات. القرارات العشرة في §16 تبقى للمرحلة 2 ولا يُبدأ بها الآن.

## 4. المخاطر المفتوحة

- `org_id` في context ليس إثباتاً مستقلاً للهوية؛ يجب منع client DB credentials والتحقق من العضوية في API/RLS.
- grants وSupabase Data API وRealtime قد تغير سطح الوصول؛ لا تعتمد مصفوفة RLS قبل التحقق العملي من الأدوار والسياسات.
- خلط tenant context في connection pool أو checkpoints قد يسبب تسريباً عابراً.
- تخزين محتوى approval والـ LLM state/output يتطلب قرارات حماية/retention/redaction.
- عرض model output يحتاج حماية XSS وPII، ولم توجد UI للاختبار.
- reservation/settlement يحتاج معاملات ذرية وreconciliation للتعطل؛ لم يُختبر.

## 5. اقتراح المرحلة التالية ومعايير قبولها

**المقترح بعد اعتماد المرحلة 1 فقط:** المرحلة 2 — حصر المتطلبات والثغرات والقرارات المفتوحة.

**معايير القبول المقترحة للمرحلة 2:** معالجة قرارات `AGENTS.md` §16 العشرة بخيارات ومزايا/عيوب وتوصية وأثر؛ تحقق رسمي حديث من الوثائق/الأسعار/الحدود عند انطباقها؛ عرض Data Residency والتشفير وprovider/DPA والدفع/queue/observability/email/integrations/coverage/retention للموافقة؛ عدم اتخاذ قرار مفتوح منفرداً؛ وقائمة فجوات واضحة مع `NOT VERIFIED` حيث يلزم.

## الملفات التي أُنشئت أو حُدّثت على الفرع

- `docs/adr/0001-modular-monolith-and-tenant-ownership.md`
- `docs/adr/0002-database-tenant-context-and-rls.md`
- `docs/adr/0003-llm-and-action-gateway-boundaries.md`
- `docs/adr/0004-open-decisions-deferred-to-phase-2.md`
- `docs/adr/0005-supabase-exposure-realtime-and-membership-function.md`
- `docs/architecture/data-model-multitenancy-rbac-rls.md`
- `docs/architecture/llm-and-action-gateways.md`
- `docs/architecture/security-threat-model.md`
- `docs/audit/phase-0-read-only-audit.md`
- `docs/audit/phase-1-architecture-security-design.md`

**توقف:** لا انتقال إلى المرحلة 2 قبل اعتماد المستخدم.
