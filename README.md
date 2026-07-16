# DockFix Labs Research

Public security research on AI agent vulnerabilities. All findings are reproducible using [AgentGuard](https://github.com/dockfixlabs/agentguard) (LGPL v3).

## Reports

- [LangChain Audit (July 2026)](./2026-07-langchain-audit.md) — 452 findings in 1,784 files. CVSS 10.0 in ShellToolMiddleware.
- [CrewAI Audit (July 2026)](./2026-07-crewai-audit.md) — 391 findings in 84 files (19x density vs LangChain).
- [Sovereign Audit 2026](./sovereign-audit-2026.md) — 6 frameworks, 6 novel attack vectors.

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

AgentGuard v0.8.2 — 22 rules, CWE+CVSS mapped, OWASP ASI compliant.
88% precision measured on 50 independent samples.

## License

Reports: CC-BY-4.0. Tool: LGPL v3.
