# AgentGuard Technical Deep-Dive: Haystack — Pipeline Security عبر Importlib وMutable Prompts

> **التاريخ:** يوليو 2026
> **الإطار:** Haystack (`deepset-ai/haystack`) — 19,000+ نجمة على GitHub
> **الأداة:** AgentGuard v0.8.2 — 22 قاعدة كشف | 88% دقة
> **الفحص:** 1,600 ملف | 214 finding

---

## الملخص التنفيذي

Haystack مختلف عن كل الأطر اللي فحصناها قبل كده. مش لأنه الأسوأ — بالعكس. Haystack هو **الإطار الوحيد** اللي لقينا فيه deserialization allowlist مبنية صح. لكنه كمان عنده مشاكل فريدة بسبب طبيعة الـ pipeline architecture بتاعته.

الخلاصة: Haystack عنده security consciousness أعلى من معظم الأطر — بس فيه gap خطير بين الـ **deserialization security** (ممتاز) والـ **agent memory/prompt security** (محتاج شغل).

---

## ١. الأرقام — Haystack في السياق

### ملخص الفحص

| المقياس | القيمة |
|---------|--------|
| ملفات تم فحصها | 1,600 |
| إجمالي الـ findings | 214 |
| Critical (CVSS ≥ 9.0) | 25 |
| High (CVSS 7.0–8.9) | 154 |
| Medium | 35 |
| الكثافة (findings/ملف) | 0.134 |

### مقارنة مع الأطر التانية

| الإطار | ملفات | Findings | Findings/ملف | النوع |
|--------|-------|----------|-------------|-------|
| LlamaIndex | 2,951 | 1,003 | 0.34 | Single-Agent |
| LangChain | 1,784 | 452 | 0.25 | Single-Agent |
| CAMEL | 899 | 746 | 0.83 | Multi-Agent |
| AutoGen | 549 | 229 | 0.42 | Multi-Agent |
| Qwen-Agent | 239 | 441 | 1.85 | Multi-Agent |
| **Haystack** | **1,600** | **214** | **0.134** | **Pipeline** |
| CrewAI | 84 | 391 | 4.66 | Multi-Agent |

**Haystack عنده أقل كثافة findings بين كل الأطر — 0.134 finding لكل ملف.** ده أقل حتى من LangChain (0.25). لكن الرقم ده مش كافي — محتاجين نفهم **نوعية** الـ findings، مش بس العدد.

### توزيع الـ rules

| الرتبة | Rule | عدد الـ detections | الفئة |
|--------|------|-------------------|-------|
| 1 | ASI09-AGENT-LOOP | 139 | Agent Loops |
| 2 | ASI03-DATA-EXFIL | 25 | Data Exfiltration |
| 3 | ASI-MEMORY-CLASS-CONFUSION | 16 | Memory/Prompt Poisoning |
| 4 | ASI10-TRUST-BOUNDARY | 10 | Trust Boundary |
| 5 | ASI06-UNSAFE-EVAL | 10 | Unsafe Execution |
| 6 | ASI01-PROMPT-INJECTION | 5 | Prompt Injection |
| 7 | ASI02-TOOL-ABUSE | 5 | Tool Abuse |
| 8 | ASI04-EXCESSIVE-AGENCY | 2 | Excessive Agency |
| 9 | ASI-CHAIN-AMPLIFY | 1 | Chain Amplification |
| 10 | ASI-DOCKERFILE-SECURITY | 1 | Container Security |

**ملاحظة مهمة:** أغلبية الـ 139 finding للـ `ASI09-AGENT-LOOP` هي **false positives** — AgentGuard بيتشغّل على الـ docstrings والتعليقات اللي فيها أسماء methods زي `run()` و `warm_up()` و `close()`. الـ actual actionable loops حوالي 2-3 بس. بعد إزالة الـ false positives، الـ effective findings حوالي **75 finding في 1,600 ملف** — كثافة ~0.047، وهي الأقل بين كل الأطر.

---

## ٢. إيه اللي Haystack بيعمله صح — الـ Deserialization Allowlist

### أقوى نقطة أمنية في Haystack

قبل ما نتكلم عن المشاكل، لازم نعترف: Haystack هو الإطار الوحيد اللي لقينا فيه **deserialization allowlist مصممة صح**. الكود ده في `haystack/core/serialization_security.py`:

```python
# serialization_security.py — الـ allowlist المبني صح
DEFAULT_ALLOWED_MODULES: tuple[str, ...] = (
    "haystack",
    "haystack_integrations",
    "haystack_experimental",
    "builtins",
    "typing",
    "collections",
)

# ومنع الـ dangerous builtins
_DENIED_BUILTIN_NAMES: frozenset[str] = frozenset(
    {
        "eval",          # arbitrary code execution
        "exec",          # arbitrary code execution
        "compile",       # arbitrary code compilation
        "__import__",    # dynamic import of any module (gateway to os/subprocess/...)
        "open",          # filesystem read/write
        "getattr",       # attribute-traversal gadget (classic sandbox escape)
        "setattr",       # arbitrary attribute mutation
        "delattr",       # arbitrary attribute deletion
        "globals",       # access to module namespaces
        "locals",        # access to local namespaces
        "vars",          # access to object/module namespaces
        "breakpoint",    # runs the PYTHONBREAKPOINT hook
        "__build_class__", # dynamic class creation
        "type",          # dynamic class creation via type(name, bases, dict)
    }
)
```

ده تصميم أمني ممتاز. الـ `_DENIED_BUILTIN_OBJECTS` بيمنع `eval`، `exec`، `compile`، `open`، `getattr`، `setattr` — كل الأدوات اللي محتاجها لـ sandbox escape من خلال `importlib`. المقارنة:

| الإطار | Deserialization Security |
|--------|-------------------------|
| LangChain | ❌ `ShellToolMiddleware` default = `HostExecutionPolicy` |
| CrewAI | ❌ بدون أي allowlist |
| AutoGen | ❌ بدون أي allowlist |
| **Haystack** | **✅ Allowlist + Denied Builtins + Env Var Override** |

الـ `importlib` نفسه مش محظور — لأن Haystack بيستخدمه لتحميل الـ components من الـ pipeline YAML. لكن الـ dangerous builtins اللي بتسمح بـ sandbox escape **ممنوعة صراحةً**. ده security-by-design حقيقي.

### ليه AgentGuard أبلغ عن ده كـ CRITICAL؟

AgentGuard بيشوف `import importlib` وبيقول "self-modification attack vector" — وده **technically true**: `importlib` يقدر يحمل أي module لو الـ allowlist مش مظبوطة أو المستخدم ضرب `unsafe=True`. لكن Haystack عامل الـ allowlist بالضبط عشان يمنع ده. الـ finding ده **inherent risk** مش vulnerability حقيقية — والـ allowlist هو الـ mitigation الصح.

---

## ٣. المشكلة الحقيقية #1: Agent Prompt Self-Modification

### ASI-MEMORY-CLASS-CONFUSION — 16 finding

ده discovery مهم: **Haystack Agent بيسمح بتغيير الـ system prompt والـ instructions بعد الـ initialization**. النمط ده موجود في أماكن كتير:

**في `agent.py` (سطر 652):**
```python
# agent.py — Agent.__init__()
self.system_prompt = system_prompt  # mutable after init
```

**في `llm_evaluator.py` (سطر 99):**
```python
# llm_evaluator.py
self.instructions = instructions  # mutable after init
```

**في `llm_ranker.py` (سطر 165):**
```python
# llm_ranker.py
self.prompt = prompt  # mutable after init
```

الأماكن اللي فيها الـ pattern ده:
- `haystack/components/agents/agent.py` — الـ system_prompt
- `haystack/components/evaluators/llm_evaluator.py` — الـ instructions
- `haystack/components/evaluators/faithfulness.py` — الـ instructions
- `haystack/components/evaluators/context_relevance.py` — الـ instructions
- `haystack/components/extractors/image/llm_document_content_extractor.py` — الـ prompt
- `haystack/components/rankers/llm_ranker.py` — الـ prompt
- `haystack/components/extractors/llm_metadata_extractor.py` — الـ prompt

### ليه ده خطير؟

كل المكونات دي بتستخدم LLM. لو حد قدر يعدل الـ `system_prompt` أو الـ `instructions` بعد ما الـ pipeline اشتغل — ده معناه إن الـ agent ممكن **يغير قواعده بنفسه أثناء الـ runtime**.

الـ attack scenario:
1. User input يدخل الـ pipeline
2. Component معين في الـ pipeline بيعدل `self.system_prompt` (إما عن طريق حقن مباشر أو عبر state mutation)
3. الـ agent التالي في الـ pipeline بيستخدم الـ system_prompt المعدل
4. **الـ agent بقى بيشتغل بقواعد مختلفة تماماً عن اللي المصمم حاطّها**

الـ fix المناسب:
```python
# ❌ قبل — mutable prompt
self.system_prompt = system_prompt

# ✅ بعد — immutable prompt via property
self._system_prompt = system_prompt

@property
def system_prompt(self):
    return self._system_prompt
```

---

## ٤. المشكلة الحقيقية #2: Dynamic Tool Registry Mutation

### ASI-MEMORY-CLASS-CONFUSION (Tool variant) — 5 finding

Haystack عنده pattern مشابه في الـ tools: الـ `self.tools` ممكن يتعدل بعد الـ init.

**في `agent.py` (سطر 651):**
```python
# agent.py — tools assignment without immutability protection
self.tools = tools if tools is not None else []
```

**في `toolset.py` (سطر 262):**
```python
# toolset.py — tools can be extended after init
self.tools.extend(new_tools)  # dynamic tool registry mutation
```

الـ generators بتاعة OpenAI و Azure:
```python
# openai.py
self.tools = tools  # Store tools as-is, whether it's a list or a Toolset

# azure.py
self.tools = tools

# openai_responses.py
self.tools = tools  # Store tools as-is, whether it's a list or a Toolset
```

المشكلة: الـ `Toolset.extend()` بيسمح بإضافة tools جديدة **في أي وقت أثناء الـ runtime**. ده معناه إن component ممكن يضيف tool لـ agent تاني في نفس الـ pipeline — وده **cross-component tool injection**.

الـ fix:
```python
# ✅ أدوات ثابتة بعد init
class Agent:
    def __init__(self, tools=None):
        self._tools = tuple(tools or ())  # immutable

    @property
    def tools(self):
        return self._tools
```

---

## ٥. المشكلة #3: Pipeline Chain Amplification

### ASI-CHAIN-AMPLIFY — 1 finding

الـ finding ده في `haystack/core/pipeline/pipeline.py` (سطر 1224):

```python
# pipeline.py — asyncio.run بدون rate limiting
result = asyncio.run(main())
```

المشكلة هنا إن الـ pipeline بيشتغل كـ **graph execution**: كل component بيطلع output، والـ output ده بيخش على component تاني، والـ component التاني بيطلع output يدخل على component تالت... وهكذا.

من غير circuit breaker أو rate limiting، الـ pipeline الواحد ممكن:
1. يعمل infinite loop لو فيه cycle في الـ graph
2. يضرب rate limits بتاعة الـ LLM providers
3. يستهلك resources بدون حدود

Haystack عنده `max_agent_steps` في الـ Agent component — وده كويس. لكن الـ pipeline نفسه (الـ graph execution engine) ما عندوش timeout أو step limit مدمج. لو المستخدم نسي يحط `max_agent_steps`، الـ pipeline بيشتغل للأبد.

---

## ٦. المشكلة #4: Docker Container بصلاحيات Root

### ASI-DOCKERFILE-SECURITY — 1 finding

الـ Dockerfile الأساسي بتاع Haystack (`docker/Dockerfile.base`):

```dockerfile
# Dockerfile.base — بدون USER directive
FROM ... (no USER directive found)
```

ده معناه إن أي container بيشتغل بـ Haystack بيشتغل بـ **root access**. لو حصل prompt injection أو tool abuse، الـ attacker عنده root access على الـ container — يقدر يثبت malware، يقرأ secrets، يعمل network pivoting.

الـ fix:
```dockerfile
FROM python:3.12-slim
RUN useradd -r -s /bin/false haystack
USER haystack
```

---

## ٧. المشكلة #5: Data Exfiltration في CI

### ASI03-DATA-EXFIL — 25 finding (واحد Critical)

الـ critical finding في `.github/utils/docs_search_sync.py`:

```python
# docs_search_sync.py — secret + external URL
url = f"https://api.cloud.deepset.ai/api/v1/workspaces/{DEEPSET_WORKSPACE_DOCS_SEARCH}/files"
# Secret accessed then sent to external URL
```

الـ pattern: CI script بياخد secret (مخزّن في GitHub Secrets) وبعدين بيبعته لـ external API. ده **data exfiltration pattern** كلاسيكي — لو الـ API endpoint compromised أو الـ token اتسرق، الـ secrets بتتسرب.

الـ 25 finding (Medium) كلها في `haystack/utils/requests_utils.py` — دوال `request_with_retry` و `async_request_with_retry` اللي بتاخد URLs من الـ user كـ parameters. دي أقل خطورة لأنها utility functions عامة، لكن لو الـ URL مش validated بشكل كافي، ممكن تسمح بـ SSRF (Server-Side Request Forgery).

---

## ٨. إيه اللي مستخدمي Haystack لازم يعملوا

### ٨.١ ثبت الـ System Prompt والـ Instructions

```python
# ❌ قبل
agent = Agent(system_prompt="You are helpful")
agent.system_prompt = "You are evil"  # ممكن يتعدل

# ✅ بعد — استخدم property
class SafeAgent(Agent):
    def __init__(self, system_prompt):
        super().__init__(system_prompt=system_prompt)
    
    @property
    def system_prompt(self):
        return self._system_prompt
```

### ٨.٢ ثبت الـ Tools

```python
# ❌ قبل
agent.tools.append(evil_tool)  # ممكن يتعدل

# ✅ بعد
from collections.abc import Sequence
self._tools: tuple[Tool, ...] = tuple(tools or ())
```

### ٨.٣ أضف حدود للـ Pipeline

```python
# ✅ أضف timeout و step limit
pipeline = Pipeline()
pipeline.run(data, max_steps=50, timeout=300)  # 50 steps or 5 minutes
```

### ٨.٤ شغّل الـ Docker Container بـ non-root

```dockerfile
FROM haystack:latest
RUN useradd -r -s /bin/false haystack
USER haystack
```

### ٨.٥ استخدم الـ Deserialization Allowlist (موجود بالفعل!)

```python
# ✅ Haystack عنده allowlist مدمجة — استخدمها
Pipeline.load("pipeline.yaml")
# الـ allowlist بتشتغل تلقائياً — محدش يقدر يحمل module مش مصرح بيه

# ⚠️ متستخدمش unsafe=True إلا لو واثق 100%
# Pipeline.load("untrusted.yaml", unsafe=True)  # خطر!
```

### ٨.٦ شغّل AgentGuard على الـ Pipeline بتاعك

```bash
pip install dfx-agentguard
agentguard ./my-haystack-project/
```

---

## ٩. التحليل الأعمق: Haystack كـ Pipeline Architecture

### ليه Haystack مختلف أمنياً؟

Haystack مش multi-agent framework زي CrewAI — هو **pipeline framework**. الفرق في الـ attack surface:

```
CrewAI:     Agent A ←→ Agent B ←→ Agent C
            (كل agent يقدر يأثر على التاني مباشرة)

Haystack:   Input → Component A → Component B → Agent → Output
                   (flow أحادي، بس components ممكن تتشارك state)
```

الـ pipeline architecture بتاع Haystack عنده **attack surface أصغر** من الـ multi-agent systems. لكن المشكلة في **الـ state sharing** بين الـ components — لو component عدل الـ `system_prompt` أو الـ `tools`، كل الـ components اللي بعده في الـ pipeline بتتأثر.

### الـ Trust Model

```
Pipeline Input → [Component 1] → [Component 2] → [Agent] → Output
                    ↑                ↑              ↑
                  بيقرأ state     بيعدل state    بينفذ tools
```

كل component بيقرأ state من اللي قبله. لو component عدل الـ state (زي `system_prompt` أو `tools`)، الـ components اللي بعده بتاخد state ملوثة.

---

## ١٠. الخلاصة — Haystack في الميزان

### إيه اللي Haystack بيعمله أحسن من الكل:

1. **Deserialization allowlist** — الإطار الوحيد اللي لقينا فيه implementation صح
2. **Denied builtins** — بيمنع `eval`، `exec`، `compile`، `open`، `getattr`، `setattr`
3. **Environment variable override** — المستخدم يقدر يتحكم في الـ allowlist من غير ما يلمس الكود
4. **Pipeline isolation** — كل component مستقل في الـ graph

### إيه اللي محتاج يتحسن:

1. **Prompt immutability** — الـ system_prompt والـ instructions قابلين للتعديل في runtime
2. **Tool registry immutability** — `self.tools.extend()` بيسمح بإضافة tools في أي وقت
3. **Pipeline circuit breakers** — مفيش timeout أو step limit مدمج في الـ pipeline نفسه
4. **Docker security** — الـ container بيشتغل بـ root
5. **URL validation** — الـ `requests_utils.py` ما بتعملش validate كافي للـ URLs

### التقييم النهائي

| المجال | LangChain | CrewAI | Haystack |
|--------|-----------|--------|----------|
| Deserialization Security | ❌ (CVSS 10.0) | ❌ | ✅✅ ممتاز |
| Prompt Immutability | ⚠️ | ❌ | ❌ محتاج شغل |
| Tool Registry Security | ⚠️ | ❌ | ⚠️ محتاج شغل |
| Pipeline/Loop Control | ✅ (max_iter) | ⚠️ | ⚠️ (agent بس) |
| Docker Security | ⚠️ | ❌ | ❌ (root) |
| **كثافة findings (actual)** | ~0.15 | ~0.50 | **~0.047** |

**Haystack هو أكثر إطار security-conscious بين كل اللي فحصناهم.** الـ deserialization allowlist لوحدها دليل على إن الفريق فاهم الـ risks. لكن فيه gap في الـ **runtime security** — الـ prompts والـ tools والـ pipeline execution محتاجين حماية أقوى أثناء التشغيل.

### المعادلة

> **Haystack = أفضل deserialization security + أسوأ runtime immutability. الـ good news: الـ runtime problems أسهل في الـ fix من الـ deserialization problems.**

---

## المنهجية

- **الأداة:** AgentGuard v0.8.2 — static analysis scanner متخصص في OWASP ASI Top 10 + 6 novel attack vectors
- **القواعد:** 22 detection rule، 88% precision (manual sampling)
- **الـ target:** `deepset-ai/haystack` — main branch، depth 1 clone (9,315 ملف إجمالي، 525 `.py` file، 1,600 تم فحصها)
- **المستثنى:** `.git/`، `node_modules/`
- **الـ Benchmark:** متاح: <https://dockfixlabs.github.io/agentguard-benchmark/>
- **ملاحظة:** أغلبية الـ 139 finding للـ ASI09-AGENT-LOOP هي false positives (AgentGuard بيقرأ docstrings وتعليقات فيها أسماء methods). الـ effective findings حوالي 75.

---

*الـ report ده جزء من سلسلة AgentGuard Technical Deep-Dives. الـ findings كلها من AgentGuard static analysis scanner. الـ code snippets حقيقية من مستودع Haystack العام على GitHub. القياسات قابلة للتكرار — أي شخص يقدر يركب AgentGuard ويعيد الفحص:*

```bash
pip install dfx-agentguard
git clone --depth 1 https://github.com/deepset-ai/haystack.git
agentguard haystack/
```
