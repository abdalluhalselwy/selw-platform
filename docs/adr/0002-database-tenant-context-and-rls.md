# ADR-0002: Tenant Context, Composite Keys, and RLS

- **الحالة:** Proposed؛ تصميم سياسات لا SQL منفذ.
- **التاريخ:** 2026-11-20
- **السياق:** `AGENTS.md` §5 يطلب عزل قاعدة البيانات والتطبيق، سياق المؤسسة لكل transaction، RLS على كل جدول، وتمرير `org_id` صراحةً. لا توجد migrations أو قاعدة بيانات في الشجرة الحالية.
- **القرار المقترح:** تفعيل RLS على كل جدول مكشوف، وأدوار DB محدودة. API يتحقق من JWT والعضوية والصلاحية ثم يمرر `org_id` الموثق إلى Service/Repository ويضبط سياقاً محدود العمر داخل transaction. سياسة tenant تسمح فقط بتطابق المؤسسة مع السياق وعضوية المستخدم النشطة. غياب السياق أو العضوية يفشل مغلقاً (deny). على كل child table توجد `org_id NOT NULL` وعلاقة FK مركبة إلزامية `(org_id, parent_id)`؛ ولكل parent table قيد `UNIQUE (org_id, id)`. الجداول ذات أكثر من parent تفرض composite FK مستقلاً لكل parent. لا تمنح الواجهة أو العميل DB credentials.
- **استثناء محتمل غير محسوم:** قد تتطلب سياسات العضوية دالة واحدة `SECURITY DEFINER` لتفادي recursion. القرار والضوابط والبدائل معروضة في ADR-0005 ولا تعتمد هنا.
- **البدائل:** RLS وحده مع client access عبر Data API؛ مخطط خاص مع سحب صلاحيات `anon` و`authenticated`; JWT claims وحدها؛ فلاتر التطبيق وحدها (غير كافية كطبقة وحيدة حسب مبدأ الدفاع المتعدد). المقارنة التفصيلية في ADR-0005.
- **العواقب:** حماية متعددة الطبقات وسلامة parent-child عبر tenants؛ يلزم اختبار أدوار DB وtransaction pooling والسياق ومراجعة schema لكل migration.
- **اختبارات قبول التنفيذ:** schema scan يفشل لأي جدول مكشوف بلا RLS أو استثناء صريح؛ Cross-Tenant CRUD؛ غياب/تبديل السياق؛ عضوية ملغاة؛ اتصال معاد الاستخدام؛ والتحقق من كل FK مركب.
- **NOT VERIFIED:** سلوك Supabase Data API وRLS/roles/connection pool، وفعالية إعدادات RLS/Storage/Realtime في بيئة النشر لم تُراجع حالياً مع الوثائق الرسمية ولم تُختبر عملياً.
- **المراجع:** `AGENTS.md` §§5.1–5.5, 13, 16.
