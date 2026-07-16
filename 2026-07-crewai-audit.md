# AgentGuard Technical Deep-Dive: CrewAI — لماذا Multi-Agent = خطر مضاعف

> **التاريخ:** يوليو 2026
> **الإطار:** CrewAI (`crewAIInc/crewAI`) — 25,000+ نجمة على GitHub
> **الأداة:** AgentGuard v0.8.1 — 22 قاعدة كشف | 88% دقة | 951 finding مؤكد عبر 7 أطر
> **الفحص:** 84 ملف | 391 finding

---

## الملخص التنفيذي

CrewAI مختلف. لما؟ لأن كل إطار agent ثاني فيه agent واحد يشتغل وينتهي الموضوع. CrewAI كله مبني على فكرة إن مجموعة agents يتنسقون مع بعض ويتفوضون ويتبادلون السياق. وده بالضبط اللي بيخلق مشكلة أمنية فريدة — كل agent في الـ crew بيقدر يأثر على اللي بعده، واللي بعده بيأثر على اللي بعده، وما في أي trust boundary حقيقي بينهم.

الأرقام تتكلم:

| المقياس | CrewAI | LangChain | النسبة |
|---------|--------|-----------|--------|
| ملفات ممسوحة | 84 | 1,784 | — |
| إجمالي الـ findings | **391** | **452** | — |
| findings لكل ملف | **~4.7** | **~0.25** | **18.8x** أشد |

391 finding في 84 ملف بس. ده يعني إن كثافة الثغرات في CrewAI أعلى بـ ~19 ضعف من LangChain. وده منطقي — كل ملف في CrewAI فيه orchestration logic، وكل orchestration logic فيه trust boundary.

---

## ١. ليه CrewAI فريد في الخطورة — الـ Multi-Agent Topology

### المشكلة الأساسية: الـ Delegation Chain

CrewAI مبني على pattern اسمه **Hierarchical Process** — فيه agent واحد اسمه "Manager" وده بيوزع المهام على باقي الـ agents. فيه كمان **Sequential Process** اللي الـ agents يشتغلوا واحد بعد الثاني. وده بيخلق ما نسميه **Delegation Chain Attack Surface**.

```
User Input → Manager Agent → Researcher Agent → Writer Agent → Output
                  ↑                  ↑               ↑
            [كل واحد يقدر يسرب أو يحرف]
```

في نظام single-agent، عندك trust boundary واحد بين المستخدم والـ agent. في CrewAI، عندك trust boundary بين كل agent واللي قبله واللي بعده. لو Crew فيه 5 agents، عندك 4 trust boundaries محتاجة validation، مش trust boundary واحد.

### الـ Attack Surface المضاعف

```
عدد الـ trust boundaries = n - 1
حيث n = عدد الـ agents في الـ Crew

مع كل agent زيادة، الخطورة بتتضاعف مش بتزيد خطياً
```

ده مش نظرية — ده اللي AgentGuard لقاه بالفعل في الكود.

---

## ٢. أبرز الـ Findings مع أنماط الكود الحقيقية

### Finding #1: Agent Collusion عبر الـ Shared Memory

**Rule ID:** `ASI-AGENT-COLLUSION`
**Severity:** CRITICAL
**عدد:** الأعلى في CrewAI

CrewAI بيتيح لكل الـ agents في نفس الـ crew إنهم يشاركوا **shared context**. المشكلة إن ده shared mutable state من غير أي trust verification.

النمط اللي AgentGuard كشفه:

```python
# crew.py — نمط مبسط
class Crew:
    def __init__(self, agents, tasks, memory=True):
        self.agents = agents
        self.shared_memory = memory  # كل الـ agents يشاركون نفس الذاكرة

    def kickoff(self, inputs):
        for agent in self.agents:
            result = agent.execute(task, context=self.shared_memory)
            # agent output → shared memory → next agent input
            # بدون أي validation أو verification
```

**ليه ده خطير؟**

Agent واحد compromised يقدر يحقن بيانات في الـ shared memory، والـ agents الباقية بتاخد البيانات دي على إنها حقيقة. ده **memory poisoning by proxy** — مش تحتاج تصل لقاعدة البيانات مباشرة، تدخل عن طريق agent تاني.

الـ attack chain:
```
Agent A (compromised) → injects malicious context into shared memory
    → Agent B reads it as trusted input
    → Agent C makes decisions based on poisoned data
    → Agent D executes actions on corrupted state
```

**اللي ميّز CrewAI هنا:** في AutoGen مثلاً، الـ GroupChat فيه shared state كمان، لكن AutoGen عنده بعض guardrails. CrewAI الـ memory system أعمق — فيه **long-term memory** عبر `crewAIPlus` اللي بيخزن embeddings في vector database، يعني الـ poison ممكن يكون persistent.

---

### Finding #2: Tool Abuse — الـ Tools بتاعة الـ Agent بدون حدود

**Rule ID:** `ASI02-TOOL-ABUSE`
**Severity:** HIGH → CRITICAL

كل agent في CrewAI يقدر يتعرف عليه tools. والمشكلة إن الـ tool execution بيحصل داخل الـ agent process من غير sandbox isolation حقيقي.

```python
# agent.py — النمط المبسط
class Agent:
    def __init__(self, role, tools=None, allow_delegation=True):
        self.tools = tools or []
        self.allow_delegation = allow_delegation  # ⚠️ الـ agent يقدر يفوض مهام

    def execute_tool(self, tool_name, tool_args):
        for tool in self.tools:
            if tool.name == tool_name:
                return tool.execute(**tool_args)  # تنفيذ مباشر بدون sandbox
```

**ثلاث مشاكل في النمط ده:**

1. **No rate limiting:** لو الـ tool system شغال بكامل، agent يقدر ينفذ عملية مرات كتير في الثانية
2. **No sandbox isolation:** الـ tool بتتنفذ في نفس الـ process، يعني `os.system()` أو `subprocess` calls تقدر توصل للـ host filesystem
3. **Delegation cascade:** `allow_delegation=True` بيخلق chain — agent ينفذ tool، والـ tool outcome بيعمل trigger لتشغيل agent تاني، واللي بيعمل trigger لـ tool تالت

الـ chain amplification pattern:

```
Agent A → Tool X → produces shell command → Agent B executes it
Agent B → Tool Y → returns file contents → Agent C reads secrets
Agent C → Tool Z → sends data to external endpoint → data exfiltrated
```

ده بالضبط اللي rule `ASI-CHAIN-AMPLIFY` بتكشفه — single input بي triggering unlimited destructive operations.

---

### Finding #3: Credential Leak في الـ Environment Configuration

**Rule ID:** `ASI07-CREDENTIAL-LEAK`
**Severity:** HIGH

CrewAI بيتعامل مع APIs كتير — OpenAI, Anthropic, Google، وغيرها. الـ configuration pattern فيه:

```python
# crew.py
import os

class Crew:
    def __init__(self):
        self.llm = os.getenv("OPENAI_API_KEY")  # يأخذ من env vars
        # لكن في بعض الأماكن:
        self.llm = "sk-proj-abc123..."  # ⚠️ hardcoded في examples
```

المشكلة الأكبر مش في الكود نفسه — في الـ **logging**. لما crew بيشتغل، الـ task execution logs بتحتوي على:
- API keys في request parameters
- Database connection strings في tool outputs
- File paths فيها credentials

وده بيحصل لأن CrewAI بيlog الـ full task context على مستوى DEBUG:

```python
# داخل task execution
logger.debug(f"Executing task with context: {task.context}")
# لو task.context فيه API keys أو connection strings — بتتسرب للـ logs
```

---

### Finding #4: Trust Boundary Violations — الـ Manager Agent كـ Single Point of Compromise

**Rule ID:** `ASI10-TRUST-BOUNDARY`
**Severity:** CRITICAL

في الـ Hierarchical Process، الـ Manager Agent هو اللي بيقرر مين يعمل إيه. ده **single point of failure**:

```python
# process.py — hierarchical pattern
class HierarchicalProcess:
    def __init__(self, manager_agent, agents):
        self.manager = manager_agent  # واحد يقرر للكل

    def step(self):
        # الـ Manager بيقرر: مين الـ agent التالي؟ إيه الـ task؟
        next_action = self.manager.decide(self.state)
        agent = self.agents[next_action.agent_id]
        agent.execute(next_action.task)
```

لو الـ Manager agent اتحرف (prompt injection مثلاً)، كل الـ decisions بتاعة تبعته تتحرف — وده بيأثر على كل الـ agents تحتيه. ده مثل إنك تاخذ root access في نظام واحد — لما تسيطر على الـ manager، السيطرة بتكون على الكل.

**النقطة المهمة:** Trust boundary violations في CrewAI مختلفة عن الأطر التانية. في LangChain، trust violation معناها الـ chain execution يسمح بـ unsanitized input. في CrewAI، trust violation معناها **agent يقرر لـ agents تانية** — وده قرار بتأثير مضاعف.

---

### Finding #5: Excessive Agency — الـ Agents عندهم صلاحيات أكبر من اللازم

**Rule ID:** `ASI04-EXCESSIVE-AGENCY`
**Severity:** HIGH

CrewAI بيبيع نفسه على إنه "agents that work together autonomously." المشكلة إن الـ autonomy بيأتي على حساب الـ confinement:

```python
class Agent:
    def __init__(self, role, goal, backstory,
                 allow_delegation=True,    # ⚠️ يقدر يفوض
                 verbose=True,             # بيlog كل حاجة
                 max_iter=None,            # ⚠️ بدون limit
                 max_rpm=None):            # ⚠️ بدون rate limit
```

- `max_iter=None`: الـ agent يقدر يعمل infinite loops — وده بالضبط اللي `ASI09-AGENT-LOOP` بتكشفه
- `max_rpm=None`: بدون rate limiting على الـ API calls
- `allow_delegation=True` by default: كل agent يقدر ينشئ sub-tasks ويوزعها

---

## ٣. الحسبة الرياضية — ليه الـ Risk في Multi-Agent Systems مضاعف

### Multiplicative Risk Model

في single-agent system، عندك:

```
Risk = P(agent_compromised) × Impact(agent_actions)
```

في multi-agent system مثل CrewAI:

```
Risk = Σ [P(agent_i_compromised) × Σ Impact(agent_j_actions | agent_i_compromised)]
        لكل j ≠ i
```

بكلمات بسيطة: كل agent compromised يزيد الـ risk بنسبة تعتمد على عدد الـ agents التانية اللي بتتأثر. Crew فيه 5 agents؟ كل agent compromised بيأثر على 4 agents تانية.

### مثال عملي

```python
crew = Crew(
    agents=[
        researcher_agent,   # يقدر يقرأ من الإنترنت
        writer_agent,       # يقدر يكتب ملفات
        reviewer_agent,     # يقدر يرسل إيميلات (لو tools include SMTP)
        analyst_agent,       # يقدر يقرأ من database
        manager_agent        # يقرر مين يعمل إيه
    ],
    tasks=[...],
    process=Process.hierarchical  # Manager يقرر للكل
)
```

لو `researcher_agent` اتحرف:
- يحقن بيانات في shared memory → يتأثر `writer_agent`
- `writer_agent` بيgenerate محتوى مبنى على بيانات سامة → يتأثر `reviewer_agent`
- `reviewer_agent` بيapprove المحتوى → `manager` بيقرر يرسله
- النتيجة: researcher واحد حرفي بيcontrol الـ output النهائي

---

## ٤. إيه اللي مستخدمي CrewAI لازم يعملوا

### ٤.١ حدد الـ Trust Boundaries

```python
# ❌ قبل
crew = Crew(agents=all_agents, process=Process.hierarchical)

# ✅ بعد — حدد لكل agent إيه اللي يقدر يشاركو
researcher = Agent(
    role="Researcher",
    tools=[search_tool],
    allow_delegation=False,  # ❌ لا تخلطه يفوض
    shared_memory=False      # ❌ لا تخلطه يكتب في shared memory
)
```

### ٤.٢ فعّل الـ Rate Limiting

```python
# ❌ قبل
agent = Agent(role="Worker", max_iter=None)

# ✅ بعد
agent = Agent(
    role="Worker",
    max_iter=10,        # أقصى 10 iterations
    max_rpm=30          # أقصى 30 requests في الدقيقة
)
```

### ٤.٣.Validate الـ Inter-Agent Communication

```python
# ✅ طبّق output validation gates
class ValidatedAgent(Agent):
    def execute(self, task, context=None):
        result = super().execute(task, context)
        # Validate output before it enters shared memory
        if not self._validate_output(result):
            raise ValueError("Agent output failed validation")
        return result
```

### ٤.٤ عزل الـ Tools

```python
# ❌ قبل — agent يقدر يوصل لأي tool
agent = Agent(role="Worker", tools=all_tools)

# ✅ بعد — كل agent يأخذ اللي يحتاجه بس
researcher = Agent(tools=[search_tool, read_tool])  # read-only
writer = Agent(tools=[write_tool])                   # write-only
# لا أحد ياخد shell access
```

### ٤.٥ شغّل AgentGuard على الكود

```bash
pip install dfx-agentguard
agentguard ./my-crew-project/
```

AgentGuard بيكشف:
- Agent collusion patterns
- Tool abuse و chain amplification
- Credential leaks في logs و environment configs
- Trust boundary violations
- Agent loops بدون circuit breakers
- Memory poisoning vectors

---

## ٥. الدرس الأوسع: Multi-Agent = Multiplicative Risk

الدرس من CrewAI مش إنه "إطار مشكل" — كل الأطر فيها ثغرات. الدرس إن **الـ multi-agent architecture بحد ذاته يضاعف الـ risk**.

### الـ Numbers من AgentGuard Audit

| الإطار | ملفات | Findings | Findings/ملف | Multi-Agent? |
|--------|-------|----------|-------------|-------------|
| LlamaIndex | 2,951 | 1,003 | 0.34 | ❌ |
| LangChain | 1,784 | 452 | 0.25 | ❌ |
| CAMEL | 899 | 746 | 0.83 | ✅ |
| AutoGen | 549 | 229 | 0.42 | ✅ |
| Qwen-Agent | 239 | 441 | 1.85 | ✅ |
| **CrewAI** | **84** | **391** | **4.66** | **✅** |

الـ pattern واضح: الأطر multi-agent فيها كثافة findings أعلى بـ 2-18x من الأطر single-agent. CrewAI هو الأعلى لأنه الأكثر تركيزاً على الـ orchestration — أقل ملفات، أعلى كثافة.

### القاعدة

> **كل ما تزيد الـ agents في النظام، كل ما تزيد الـ trust boundaries اللي محتاجة validation. Multi-agent بدون trust verification = distributed attack surface.**

---

## المنهجية

- **الأداة:** AgentGuard v0.8.1 — static analysis scanner متخصص في OWASP ASI Top 10 + 6 novel attack vectors
- **القواعد:** 22 detection rule، 88% precision
- **المقارنة:** AgentGuard vs Semgrep vs CodeQL — AgentGuard detect 100% (50/50)، الباقي 0%
- **الـ Benchmark:** عام ومتاح: <https://dockfixlabs.github.io/agentguard-benchmark/>

---

## الخلاصة

CrewAI إطار قوي للـ multi-agent orchestration. لكن قوته هي ضعفه — الـ delegation chains والـ shared memory والـ hierarchical processes كلها attack surfaces فريدة ما فيش في الأطر single-agent. الـ 391 finding في 84 ملف بتقول واضح: كثافة الثغرات مرتفعة والـ trust boundaries غير موجودة.

المستخدمين لازم يطبقوا:
1. **Strict trust boundaries** بين الـ agents
2. **Output validation gates** على كل inter-agent communication
3. **Rate limiting** على tool execution و API calls
4. **Principle of least privilege** — كل agent ياخد الـ tools اللي يحتاجها بس
5. **AgentGuard** على كل crew project قبل production

---

*الـ report ده جزء من سلسلة AgentGuard Technical Deep-Dives. الـ findings كلها من AgentGuard static analysis scanner. الـ code patterns مبسطة للشرح — شوف الكود المصدري لـ CrewAI والـ rules في [AgentGuard GitHub](https://github.com/dockfixlabs/agentguard) للتفاصيل الكاملة.*
