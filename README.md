# DockFix Labs Research — AI Agent Framework Security Audits

All audits performed with [AgentGuard](https://github.com/dockfixlabs/agentguard) v0.8.3 (LGPL v3).

## Master Comparison — 6 AI Agent Frameworks

| Framework | Stars | Files | Findings | CRITICAL | Density | Key Risk |
|-----------|-------|-------|----------|----------|---------|----------|
| **CrewAI** | 25K+ | 84 | 391 | — | **4.66** | Multi-agent collusion, memory poisoning |
| **Dify** | 60K+ | 637 (core) | 306 | 65 | 0.48 | Plugin importlib, agent collusion |
| **LlamaIndex** | 40K+ | 2,951 | 1,003 | 209 | 0.34 | exec(), raw SQL injection |
| **Qwen-Agent** | 10K+ | 239 | 441 | — | 1.85 | Multi-agent loops, tool abuse |
| **AutoGen** | 38K+ | 549 | 229 | — | 0.42 | Docker host mount, self-modification |
| **LangChain** | 100K+ | 1,784 | 452 | — | 0.25 | CVSS 10.0 ShellToolMiddleware |
| **Haystack** | 19K+ | 1,600 | ~75 | 25 | **0.05** | Best security posture overall |

## Individual Reports

- [LangChain Audit (July 2026)](./2026-07-langchain-audit.md) — CVSS 10.0 in ShellToolMiddleware
- [CrewAI Audit (July 2026)](./tech_report_crewai.md) — 19x density anomaly
- [LlamaIndex Audit (July 2026)](./tech_report_llamaindex.md) — exec() in evaporate extractor
- [Haystack Audit (July 2026)](./tech_report_haystack.md) — Best security, mutable prompts
- [Dify Audit (July 2026)](./tech_report_dify.md) — Plugin system, agent collusion
- [Sovereign Audit 2026](./sovereign-audit-2026.md) — 6 frameworks, 6 novel attack vectors

## Active GHSAs & Issues

| ID | Framework | Severity | Status |
|----|-----------|----------|--------|
| [GHSA-44f8](https://github.com/langchain-ai/langchain/security/advisories/GHSA-44f8-xvpq-8jcg) | LangChain | CVSS 10.0 | Triage (23 days) |
| [autogen#7917](https://github.com/microsoft/autogen/issues/7917) | AutoGen | Docker CRITICAL | Open (5 comments) |
| [autogen#7918](https://github.com/microsoft/autogen/issues/7918) | AutoGen | Agent HIGH | Open (4 comments) |

## Methodology

AgentGuard v0.8.3 — 22 rules, CWE+CVSS mapped, OWASP ASI compliant.  
88% precision measured on 50 independent samples.

## License

Reports: CC-BY-4.0 | Tool: LGPL v3
