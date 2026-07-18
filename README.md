# DockFix Labs Research — AI Agent Framework Security Audits

All audits performed with [AgentGuard](https://github.com/dockfixlabs/agentguard) v0.8.3 (LGPL v3).

## Master Comparison — 5 AI Agent Frameworks

| Framework | Stars | Files | Total Findings | CRITICAL | HIGH | Density (findings/file) | Key Risk |
|-----------|-------|-------|---------------|----------|------|------------------------|----------|
| **CrewAI** | 25K+ | 84 | 391 | — | — | **4.66** | Multi-agent collusion, memory poisoning |
| **LlamaIndex** | 40K+ | 2,951 | 1,003 | 209 | 580 | 0.34 | Raw SQL execution, exec() in evaporate |
| **LangChain** | 100K+ | 1,784 | 452 | — | — | 0.25 | CVSS 10.0 in ShellToolMiddleware |
| **AutoGen** | 38K+ | 549 | 229 | — | — | 0.42 | Docker host mount, agent self-modification |
| **Qwen-Agent** | 10K+ | 239 | 441 | — | — | 1.85 | Multi-agent loops, tool abuse |
| **Haystack** | 19K+ | 1,600 | 214 (~75 effective) | 25 | 154 | **0.05** | Best security posture overall |

## Individual Reports

- [LangChain Audit (July 2026)](./2026-07-langchain-audit.md)
- [CrewAI Audit (July 2026)](./2026-07-crewai-audit.md)
- [LlamaIndex Audit (July 2026)](./tech_report_llamaindex.md)
- [Haystack Audit (July 2026)](./tech_report_haystack.md)
- [Sovereign Audit 2026](./sovereign-audit-2026.md) — 6 frameworks, 6 novel attack vectors
- [PAPER: 6,173 Findings](./PAPER_6173_FINDINGS.md) — comprehensive vulnerability paper

## Active GHSAs

| GHSA | Framework | Severity | Status |
|------|-----------|----------|--------|
| [GHSA-44f8-xvpq-8jcg](https://github.com/langchain-ai/langchain/security/advisories/GHSA-44f8-xvpq-8jcg) | LangChain | CVSS 10.0 | Triage (filed Jul 5) |

## Active Issues

| Issue | Framework | Severity |
|-------|-----------|----------|
| [autogen#7917](https://github.com/microsoft/autogen/issues/7917) | AutoGen | Docker host mount (CRITICAL) |
| [autogen#7918](https://github.com/microsoft/autogen/issues/7918) | AutoGen | Agent self-modification (HIGH) |

## Methodology

AgentGuard v0.8.3 — 22 rules, CWE+CVSS mapped, OWASP ASI compliant.
88% precision measured on 50 independent samples.

## License

Reports: CC-BY-4.0. Tool: LGPL v3.
