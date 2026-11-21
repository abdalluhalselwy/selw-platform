# نموذج التهديدات الأمني — تصميم أولي

- **الحالة:** تقييم تصميمي أولي، وليس اختبار اختراق أو تقييم امتثال.
- **النطاق:** منصة SaaS Multi-Tenant كما تحدد `AGENTS.md`؛ لا يوجد تطبيق فعلي.
- **NOT VERIFIED:** إعدادات الخدمات الخارجية، البيئة، تدفقات البيانات الفعلية، الثغرات التشغيلية، والالتزامات القانونية. يلزم تحديث النموذج بعد الاختيارات والتنفيذ.

## الأصول وحدود الثقة

**الأصول:** بيانات المؤسسة والمشاريع والملف والحقائق والمحادثات ومخرجات النماذج؛ JWT والجلسات؛ provider/OAuth credentials؛ action snapshots/versions/hashes/approvals؛ ميزانيات وreservations وusage؛ checkpoints؛ audit؛ backups؛ مفاتيح التشفير؛ سلامة إجراءات النشر والرسائل والميزانية.

**حدود الثقة:** browser/Next.js؛ FastAPI؛ Postgres/Supabase (DB/Data API/Auth/Storage/Realtime حدود منفصلة)؛ Workers/Queue/LangGraph؛ LLM providers وMake/connectors؛ المحتوى الخارجي/المستخدم؛ platform support.

## التهديدات والتخفيف والأدلة المطلوبة

| التهديد | الأثر | ضوابط تصميمية | اختبار/دليل مطلوب |
|---|---|---|---|
| Tenant IDOR/org ID tampering | تسريب/تغيير tenant آخر | membership check في API، org explicit، RLS، composite FKs على كل parent | Cross-Tenant CRUD عبر DB/API؛ أي عبور BLOCKER |
| Data API أو grants أوسع من المقصود/RLS bypass | تسريب مباشر | مقارنة private schema + revoke `anon/authenticated` ضد exposed schema + RLS؛ قرار مفتوح ADR-0005 | تحقق grants/schema وnegative tests بالأدوار الفعلية؛ `NOT VERIFIED` |
| اتصال DB بصلاحية مفرطة أو context leakage في pool | تجاوز العزل | restricted role، tenant context transaction-bound، fail closed، لا DB credentials للعميل | اختبار production role، وسيناريو pooled B ثم A وغياب context |
| RBAC escalation أو self-approval | فعل غير مصرح | granular permissions، approval capability منفصلة، خيارات self-approval مفتوحة، platform admins معزولون | مصفوفة denial لكل role، concurrent approval tests |
| Prompt injection أو تسرب عبر agent/checkpoint | إفشاء بيانات أو استئناف state غير tenant | untrusted input framing، least tools، org/task scoping، لا write capability للنموذج | injection suite + cross-tenant checkpoint resume/load tests؛ LangGraph details NOT VERIFIED |
| اختلاق أسعار/إحصائيات/ادعاءات | تضليل أو مخالفة | provenance/grounding، arithmetic in code، Claim Verifier، schema validation، safe failure | evals تشمل نقص المصدر والرفض؛ لا eval suite حالياً |
| تغيير المحتوى/الحساب/الوقت بعد approval | تنفيذ غير معتمد | immutable action_versions؛ hash يغطي payload وscheduled_time وtarget account؛ approval tied to version؛ L3 confirmation | غيّر كل hash field، version rollback/replay، expiry/concurrent execution |
| إعادة تنفيذ بعد timeout أو replay | مضاعفة نشر/رسالة/خصم | idempotency، reconciliation، bounded retries، state machine | duplicate requests ومحاكاة timeout/network retries |
| سباق إنفاق budget/quotas | تجاوز plan limits أو تكلفة | pre-call reservation، settlement على actual usage، idempotent release/reconcile | ضغط متزامن عند بقاء وحدات قليلة؛ إثبات ألا يتجاوز حد المؤسسة |
| XSS في عرض مخرجات النموذج | تنفيذ script أو سرقة جلسة/بيانات أو فعل باسم المستخدم | اعتبر model output غير موثوق؛ escaping/contextual encoding، لا HTML خام أو sanitization بمكتبة معتمدة بعد التحقق، CSP، URL scheme allowlist، عدم تخزين output كـ trusted markup | اختبارات stored/reflected XSS في Markdown/HTML/links/RTL mixed text، CSP verification؛ لا واجهة للاختبار الآن |
| تسريب بيانات عبر عرض output | إفشاء PII/سر/tenant data في UI أو prompt | output screening/redaction قبل العرض، provenance، عدم دمج outputs غير موثوقة في HTML، authorization لكل resource | test secrets/PII/cross-tenant markers في rendered output، snapshot/e2e |
| SSRF من URL fetcher | وصول إلى metadata/internal network | allowlist schemes؛ منع private/link-local/metadata IPv4/IPv6؛ redirect/DNS revalidation؛ timeout/size cap | SSRF suite؛ لا fetcher حالي |
| تسرب provider/OAuth secrets | استيلاء على integration أو فاتورة | secret manager قرار مفتوح، لا Git/DB plaintext/prompt/log، least OAuth scopes وrotation | secret scan، log redaction، revoke/rotation drill |
| Webhook forgery/replay | حالة integration مزيفة | signature verification، replay window، idempotency، adapter contract | invalid signature/replay tests؛ provider-specific details NOT VERIFIED |
| Audit/log/trace/backup PII exposure | كشف بيانات عبر الدعم/مزود telemetry | redaction، append-only audit، محدودية الوصول والاحتفاظ/الإقامة خيارات مفتوحة | log sample review، access review، backup restore tests |
| impersonation misuse | وصول دعم باسم العميل | platform_admin منفصل، session reason/scope/time، موافقة عميل مسبقة، Audit لكل فعل | عدم بدء/تمديد الجلسة دون approval صالح؛ اختبارات انتهاء/إلغاء |
| rate/cost exhaustion | تعطيل الخدمة أو خسارة مالية | rate limits per user/org، reservation/quotas، hard stop، retries capped، alerts | concurrency/load/abuse tests |
| supply chain/CI compromise | نشر كود أو سر مخترق | PR review، secret/dependency/SAST gates، least CI permissions | workflows وتقارير scan قبل الدمج؛ CI غير موجود |
| حذف/احتفاظ/نسخ غير متسق | بقاء بيانات مطلوب حذفها أو فقدان استعادة | lifecycle/retention/export policy، backup scope/restore drill | delete/export/restore test؛ policy قرار مفتوح |

## حدود التقييم والبوابات

لا يوجد كود يمكن فحصه بـ SAST/DAST، ولا DB لتجربة RLS، ولا واجهة لتقييم XSS. جميع الاختبارات أعلاه متطلبات مستقبلية `NOT VERIFIED` وليست PASS. قبل external writes يجب اعتماد RLS/approvals ثم اختبار hash/version/idempotency/reservation. قبل تشغيل واجهة يجب اختبار XSS وعرض output. قبل الإطلاق يلزم security review، Cross-Tenant suite، اختبار استعادة، ومراجعة قانونية.
