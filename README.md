# JacoChen | AI Architect

Building enterprise AI platforms from scratch — LangGraph orchestration, MongoDB semantic layers, anti-hallucination pipelines, and the architecture trade-offs nobody writes about.

One engineer. ~300 users. Four business domains. Eight months in production.

---

## 🔧 Technical Architecture

| Post | Topic |
|------|-------|
| [Six Trade-offs That Shaped Our Enterprise AI Workbench](https://jacochan-eattheblock.github.io/2026/09/04/building-enterprise-ai-workbench/) | LangGraph vs chains, MongoDB for everything, monolith with migration ports |
| [How I Built an Enterprise AI Platform from Scratch in 2 Months](https://jacochan-eattheblock.github.io/2026/09/04/build-ai-platform-from-scratch/) | Week-by-week solo build, shortcuts vs debt, three things I'd change |
| [From Text-to-MQL to AI-Native Dashboards](https://jacochan-eattheblock.github.io/2026/09/05/from-text-to-mql-to-ai-native-dashboards/) | Semantic views ($set additive collapse), intent routing, 5 architectural shifts |
| [7 Techniques That Cut RAG Hallucination by 80%](https://jacochan-eattheblock.github.io/2026/09/05/reduce-rag-hallucination/) | Structured output, bigram scoring, dual-use routing, 6-gate validation |
| [Prompt Engineering for Enterprise AI](https://jacochan-eattheblock.github.io/2026/09/06/prompt-engineering-for-enterprise-ai/) | Prompts as semantic mappers, sandwich architecture, three surgical cuts |
| [Context Engineering Lessons from Production](https://jacochan-eattheblock.github.io/2026/09/06/context-engineering-lessons-from-production/) | Three-layer memory, rolling compression, context budget |
| [Memory Systems and Token Economics](https://jacochan-eattheblock.github.io/2026/09/06/memory-systems-and-token-economics-for-production-agents/) | Semantic vs episodic memory, cost attribution, token pricing |
| [Building AI Observability Without LangSmith](https://jacochan-eattheblock.github.io/2026/09/06/building-ai-observability-without-langsmith/) | 8-table lineage system, ~2ms overhead, on-premise constraints |
| [Designing Tools for Production AI Agents](https://jacochan-eattheblock.github.io/2026/09/06/designing-tools-for-production-ai-agents/) | SkillHub, component registry, semantic query plans, eval-driven improvement |
| [Evals and Multi-Agent Coordination](https://jacochan-eattheblock.github.io/2026/09/06/evals-and-multi-agent-coordination-in-production/) | Three-layer evaluation, compound intent DAG, code-orchestrated execution |
| [K8s Sandbox Security for AI Skill Execution](https://jacochan-eattheblock.github.io/2026/09/06/k8s-sandbox-security-for-ai-skill-execution/) | YARA scanning, ephemeral Jobs, NetworkPolicy, prepuller DaemonSet |

## 📚 Reading Insights (中文)

| Post | Books |
|------|-------|
| [纳瓦尔宝典：7 个认知升级](https://jacochan-eattheblock.github.io/2026/09/06/naval-almanack-reading-notes/) | The Almanack of Naval Ravikant |
| [巴菲特信中偷到的 6 个系统设计思维](https://jacochan-eattheblock.github.io/2026/09/06/buffett-letters-reading-notes/) | Warren Buffett's Letters to Shareholders |
| [老师攒 600 万，程序员为什么不行？](https://jacochan-eattheblock.github.io/2026/09/06/millionaire-teacher-reading-notes/) | Millionaire Teacher |
| [穷查理：反过来想，总是反过来想](https://jacochan-eattheblock.github.io/2026/09/07/poor-charlie-almanack-reading-notes/) | Poor Charlie's Almanack |
| [原则：把人生当系统来运维](https://jacochan-eattheblock.github.io/2026/09/07/principles-ray-dalio-reading-notes/) | Principles by Ray Dalio |
| [程序员必备 10 个思维模型](https://jacochan-eattheblock.github.io/2026/09/07/great-mental-models-reading-notes/) | The Great Mental Models Vol.1 & 2 |
| [马斯克的「算法」五步法](https://jacochan-eattheblock.github.io/2026/09/07/elon-musk-reading-notes/) | Elon Musk by Isaacson |
| [慢慢富的底层逻辑](https://jacochan-eattheblock.github.io/2026/09/07/wealth-slow-and-steady-reading-notes/) | 4 本理财书合集 |
| [从找到想做的事到不上班也能活](https://jacochan-eattheblock.github.io/2026/09/07/find-your-path-reading-notes/) | 4 本职业独立书合集 |
| [AI 杠杆时代的不对称赌注](https://jacochan-eattheblock.github.io/2026/09/07/ai-leverage-asymmetric-bets-and-finding-your-mission/) | 下行可控、上行巨大 |
| [Jeff Dean 的 1% 法则](https://jacochan-eattheblock.github.io/2026/09/07/yc-jeff-dean-1-percent-rule-ai-insights/) | YC + Google Research |

## Tech Stack

- **Orchestration**: LangGraph (stateful graphs, interrupt/resume, sub-graph composition)
- **Database**: MongoDB (checkpoint + store + business + vector search — one DB for everything)
- **LLM**: Structured output (Pydantic), semantic routing, few-shot anchoring, evaluation loops
- **Security**: YARA scanning, K8s ephemeral sandbox, NetworkPolicy isolation, PIPL compliance
- **Observability**: Custom 8-table lineage system (no LangSmith — on-premise constraints)

## Contact

- 🌐 Blog: [jacochan-eattheblock.github.io](https://jacochan-eattheblock.github.io)
- 📧 Email: jacochen1108@gmail.com
