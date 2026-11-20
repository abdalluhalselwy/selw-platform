# ADR-0001: Modular Monolith and Tenant Ownership

- **الحالة:** Proposed؛ بانتظار اعتماد المستخدم، وليس قرار تنفيذ.
- **التاريخ:** 2026-11-20
- **السياق:** المستودع التأسيسي لا يحتوي تطبيقاً أو schema أو اختبارات. المنصة المقصودة SaaS متعددة المؤسسات. `AGENTS.md` §§4–5 يعتمد FastAPI Modular Monolith وPostgreSQL عبر Supabase وعزل tenant على طبقتين.
- **القرار المقترح:** الالتزام بالاختيارات المعتمدة دون تغييرها: Next.js/TypeScript، FastAPI Modular Monolith، PostgreSQL/Supabase، Supabase Auth، Redis، LangGraph، Docker/CI. تكون Services وRepositories ضمن حدود modules ولا تستعلم وحدة مباشرةً عن جداول وحدة أخرى. PostgreSQL هو مصدر الحقيقة، وكل كتابة للتطبيق عبر FastAPI. كل tenant-scoped row يحمل `org_id`; وكل علاقة بصف أب داخل tenant تفرض composite foreign key `(org_id, parent_id)` إلى `UNIQUE (org_id, id)` في الجدول الأب.
- **البدائل:** Microservices (غير ملائمة للبداية وفق rationale المعتمد)، shared-table access بين الوحدات (اقتران غير مرغوب)، قاعدة/Schema مستقلة لكل مؤسسة (تغيير بنيوي غير معتمد ويحتاج ADR وموافقة).
- **العواقب:** وحدات قابلة للفهم والاختبار دون تغيير التقنية المعتمدة؛ يلزم تدقيق migrations والقيود المركبة وحدود الخدمات قبل التنفيذ.
- **NOT VERIFIED:** لا توجد شيفرة أو قاعدة حالية لإثبات هذه الحدود أو التحقق من إعدادات Supabase/connection pooling.
- **المراجع:** `AGENTS.md` §§2, 4.1–4.4, 5, 14–15.
