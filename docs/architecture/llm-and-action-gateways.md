# تصميم LLM Gateway وAction Gateway

- **الحالة:** تصميم مفاهيمي مقترح؛ لا تنفيذ، ولا مزود أو Queue معتمد.
- **المراجع:** `AGENTS.md` §§4.4, 6, 7, 9, 11–13.
- **NOT VERIFIED:** واجهات LangGraph وPostgres checkpointer/interrupt، مزودو النماذج/واجهاتهم وأسعارهم وحدودهم، وسلوك Supabase Auth/Data API/Realtime. يجب مراجعة الوثائق الرسمية الحالية قبل أي تنفيذ.

## التدفق وحدود المسؤولية

```text
FastAPI: JWT → org membership/permission → Service
  → Orchestrator/Skill (thin; no direct provider SDK)
  → LLM Gateway → provider adapter → schema/grounding/redaction → usage event

External write request
  → Action Gateway → policy/budget/connector check
  → immutable action_version(snapshot + hash)
  → approvals for that exact version
  → revalidate approval/hash/expiry/limits → integration adapter
  → idempotent execution/reconciliation → append-only audit
```

## LLM Gateway

1. بوابة واحدة داخلية لعمليات `generate`, `stream`, structured output وtool calling وفق دعم provider الذي يتم التحقق منه. تمنع Skills/routers من استدعاء provider مباشرة.
2. الطلب يحمل `org_id`, actor, task, skill/version, risk level, content locale, allowed tools, data policy و`correlation_id`. لا يثق في معرفات من prompt أو model output.
3. قبل الاتصال: تحقق authorization وplan/quota/budget. عند 80% تحذير وعند الحد إيقاف صارم وفق AGENTS. احجز budget قبل dispatch لتفادي السباق ثم سوِّ الاستهلاك الفعلي بعد الرد؛ تفاصيل الحجز في «ميزانية متزامنة» أدناه.
4. أقل بيانات ممكنة: تصنيف/تقليل PII، عدم تمرير سجلات غير لازمة، وعزل المحتوى الخارجي كبيانات غير موثوقة. الأدوات allowlist، ومصادر URL تحتاج SSRF controls.
5. adapter provider-neutral، secrets عبر آلية ستُختار لاحقاً، timeout/retry محدودان للأخطاء القابلة للإعادة. لا fallback إلى منطقة/provider محظور. يسجل provider/model المستخدمان فعلياً.
6. تحقق structured output، إصلاح محدود ثم فشل آمن. الأرقام تُحسب في الكود، وclaims المادية تمر عبر Claim Verifier/provenance أو وسم افتراض.
7. `usage_events`: tenant/task/skill/provider/model/token counts/estimated cost/latency/outcome، مع redaction. لا تحفظ prompt/output افتراضياً؛ تفعيل tracing rich يحتاج قرار retention/consent/region.
8. Streaming للردود المناسبة، والمهام الطويلة ذات status/progress؛ اختيار SSE أو Realtime مفتوح في ADR-0005.
9. أخطاء آمنة، لا stack trace أو بيانات tenant في الرد؛ budget exhausted يرفض بأمان.

## سباق الميزانية: reservation ثم settlement

- قراءة remaining budget ثم الاستدعاء بلا حجز تتيح لطلبات متزامنة أن تتجاوز الحد. التصميم المقترح: عملية ذرية على `org_budgets`/quota تحجز تقديراً أعلى محافظاً قبل provider request، مع reservation ID ومدة صلاحية؛ إذا لا رصيد كافٍ يرفض قبل الاتصال.
- بعد الاستجابة، يسجل usage event ويُسوّى الحجز على التكلفة/الوحدات الفعلية. الفرق يعاد للرصيد؛ الفشل قبل الإرسال يحرر الحجز، والـ timeout حالة غير محسومة تبقى محجوزة مؤقتاً حتى reconciliation/expiry منضبط.
- idempotency على reservation/settlement لمنع الخصم أو الإرجاع المكرر. تُعالج concurrent updates عبر transaction/locking أو آلية ذرية مناسبة بعد تحقق DB، ويُراقب مجموع reserved+consumed.
- إشعار 80% يعتمد استهلاكاً محجوزاً ومسوّى بوضوح. المصدر الرسمي للتسعير وأسلوب تقدير التكلفة وخصائص transaction لم تتحقق: `NOT VERIFIED`.

## Action Gateway ونسخ الإجراءات والموافقات

- `actions` كيان workflow؛ `action_versions` تحمل snapshot immutable وhash لكل نسخة. كل تعديل ينتج version جديداً، ولا يحدث payload معتمد في مكانه.
- canonical hash يغطي كامل payload المؤثر، بما فيه صراحةً `scheduled_time` والمنصة/`target_account_id` والمعلمات التي تحدد الفعل. تثبت قواعد canonicalization والـ hash algorithm عند التنفيذ، وتخزن النسخة/الخوارزمية للـ verification.
- `approvals` relation واحد-إلى-متعدد مع `actions`, وكل approval مربوط بـ `action_version_id` و`approved_hash`, و`approval_kind` (مثلاً موافقة أساسية أو تأكيد L3 الثاني). الموافقة لا تسري على إصدار لاحق. L3 confirmation سجل/فاعل منفصل وفق سياسة تعتمد لاحقاً.
- الحالات: `draft → pending_approval → approved → executing → succeeded | failed | cancelled | expired`. الإصدارات السابقة تظل محفوظة ومؤرشفة؛ تعرض الحالة الحالية للنسخة فقط.
- قبل الموافقة: identity/org/membership/permission، connector/scopes، risk classification، payload validation، account/destination، scheduled time، سقف/عدد، expiry. تعرض للموافق لقطة النسخة الفعلية ومستوى المخاطر.
- قبل التنفيذ: إعادة تحقق من الموافقة على نفس version/hash، الصلاحية/الانتهاء، صلاحيات actor والحدود، connector state، ومفتاح idempotency. أي تعديل/اختلاف يبطل الموافقات ويعود إلى approval للنسخة الجديدة.
- L2/L3 لا auto-approval. L3 يطلب limits صارمة وتأكيداً ثانياً. سياسة السماح لصاحب الطلب بالموافقة على إجرائه **قرار مفتوح**: حظر كامل؛ أو استثناء L2 فقط مع منع L3؛ أو سياسة حسب org/action مع افتراضي تقييدي. مزاياها/مخاطرها موثقة في ERD doc ولا حسم.
- adapter رسمي فقط وبأقل scopes؛ لا يسمح للنموذج بكتابة خارجية. network timeout = unknown/reconcile، وليس إعادة إرسال عمياء. retries محدودة؛ cancel/expire state transitions.
- Audit append-only للأحداث مع actor/source/org/correlation ID والنتيجة المنقحة؛ لا credentials ولا PII كاملة.

## LangGraph state isolation

أي checkpoints محفوظة تعامل كبيانات tenant. اربطها بسياق org/task موثوق، وامنع تحميل/استئناف state بناء على thread ID وحده. يلزم اختبار الوصول العابر، ومعالجة retention/redaction للـ state.
**NOT VERIFIED:** واجهات LangGraph الحالية وآليات checkpoint namespace/tenant isolation والـ Postgres checkpointer لم تُراجع؛ لا ضمان أمني من thread prefix وحده.

## بوابات القبول قبل التنفيذ

Contract tests لكل provider/adapter، unit tests للميزانية/reservation/settlement/state machine/hash/version/idempotency، negative tests للتوقيع والـ approval replay وexpired approval، Cross-Tenant tests، prompt injection/output leakage tests، وإخفاقات/timeout reconciliation في sandbox. كلها `NOT VERIFIED` لعدم وجود تطبيق أو بيئة.
