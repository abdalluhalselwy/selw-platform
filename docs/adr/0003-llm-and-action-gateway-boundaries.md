# ADR-0003: LLM Gateway and Action Gateway Boundaries

- **الحالة:** Proposed؛ بانتظار اعتماد المستخدم.
- **التاريخ:** 2026-11-20
- **السياق:** جميع استدعاءات النماذج تمر عبر LLM Gateway؛ وكل أثر خارجي يتطلب Action Gateway وموافقة/Audit. قرارات المزودين والـ queue والرصد مفتوحة.
- **القرار المقترح — LLM Gateway:** نقطة داخلية موحدة لـ `generate`, `stream`, structured output وtool calling وفق إمكانات مزود تم التحقق منها. يمر كل طلب بسياق tenant/actor/task/skill والسياسات؛ تُفحص العضوية والميزانية وحدود المؤسسة قبل الاستدعاء؛ تُقلل PII؛ الأدوات allowlisted؛ retries محدودة؛ fallback لا يتجاوز سياسة بيانات المؤسسة؛ تُتحقق المخرجات بمخطط؛ claims الرقمية تُحسب بالكود وتُسند لمصادر أو توسَم افتراضاً. تسجيل `usage_events` يكون منقحاً، ولا يحفظ prompts/outputs افتراضياً.
- **القرار المقترح — Action Gateway:** حالات الإجراء `draft → pending_approval → approved → executing → succeeded | failed | cancelled | expired`. تُحفظ حمولات الإجراء بإصدارات immutable في `action_versions` مع snapshot وhash. كل سجل `approvals` يشير إلى إصدار محدد؛ relation actions:approvals واحد-إلى-متعدد لدعم إعادة الموافقة وتأكيد L3. يدخل `scheduled_time` وtarget account ضمن canonical payload/hash. قبل التنفيذ يُعاد التحقق من hash والإصدار والهوية/المؤسسة/الصلاحية والانتهاء/الحدود ومفتاح idempotency. أي تعديل ينشئ إصداراً جديداً ويبطل صلاحية الموافقات على الإصدارات السابقة. L2/L3 دون موافقة تلقائية؛ L3 بحدود صارمة وتأكيد ثانٍ.
- **البدائل:** استدعاء provider من Skill مباشرة، أو منح النموذج صلاحية الكتابة الخارجية، أو تجاوز البوابة عبر Make (مرفوضة لتعارضها مع حدود المشروع).
- **العواقب:** فصل التخطيط عن الفعل، وتحسين التتبع والميزانية؛ يلزم تنفيذ reconciliation لحالات timeout، ومراجعة حقوق الموافقة وقواعد فصل الطالب عن الموافق.
- **NOT VERIFIED:** واجهات LangGraph الحالية، دعم interrupt/checkpointer؛ APIs/providers والتسعير والـ limits؛ Data Processing Agreements؛ تكاملات المنصات وسلوك idempotency لديها. لا اختيار مزود أو queue في هذا القرار.
- **المراجع:** `AGENTS.md` §§6.1–6.6, 9, 12, 16.
