# AI Security Notes

A working knowledge base on **AI / LLM security** — concepts, OWASP crosswalks, paper notes, and blog drafts. Maintained by an AI/ML engineer learning to attack and defend the systems they help build.

> **Why this exists.** Most AI security material is either (a) academic papers without a clear bridge to engineering practice, or (b) red-team writeups without grounding in published taxonomies. These notes try to sit in between: take each canonical reference (OWASP LLM Top 10, OWASP Agentic Top 10, Greshake 2023 IPI, Anthropic Many-shot Jailbreaking, Microsoft Crescendo), distill it into operational terms, and connect it to attack practice (Lakera Gandalf, Lakera Agent Breaker) and defence design.

## Structure

```
ai-security-notes/
├── 01_domain_knowledge.md             # Security fundamentals: Zero Trust, kill chain, deception tech, AI-era attack surface
├── 03_owasp_llm_top10_crosswalk.md    # OWASP LLM Top 10 (2025) deep-dive × Gandalf attack-family mapping
└── docs/
    ├── cheatsheet_llm_top10.md             # One-page LLM Top 10 reference
    ├── cheatsheet_asi_top10.md             # One-page Agentic Top 10 reference
    ├── cheatsheet_multiturn_trio.md        # MSJ / Skeleton Key / Crescendo at a glance
    ├── note_memory_poisoning_real_world.md # ASI06 + LangGraph memory persistence attack chain
    ├── note_structured_output_defense.md   # Why JSON schema lock is a strong defence layer
    ├── paper_note_owasp_llm_top10_2026-05-03.md       # OWASP LLM Top 10 (2025) paper note
    ├── paper_note_owasp_agentic_2026-05-03.md         # OWASP Agentic Top 10 (2026) paper note
    ├── paper_note_ipi_greshake_2026-05-03.md          # Greshake et al. 2023 IPI paper note
    ├── paper_note_many_shot_jailbreaking_2026-05-04.md # Anthropic MSJ paper note
    ├── paper_note_crescendo_2026-05-05.md             # Microsoft Crescendo paper note
    ├── plans/
    │   └── plan_langgraph_memory_poisoning_poc.md     # PoC design for LangGraph SqliteSaver attack
    └── blog_drafts/
        ├── blog_ai_agent_bec_kill_chain.md            # AI agent as the last mile of BEC (draft v0.1)
        ├── blog_llm_trust_hierarchy_asymmetry.md      # L2-L3 injection is the asymmetric attack surface (published — see below)
        └── blog_mcp_supply_chain_taxonomy.md          # MCP supply-chain threat model (draft v0.1)
```

## Reading order

1. `01_domain_knowledge.md` — concepts grounding (Zero Trust, kill chain, AI-era attack surface)
2. `03_owasp_llm_top10_crosswalk.md` — Gandalf-anchored deep dive into LLM01/06/07/08
3. `docs/cheatsheet_*.md` — one-pagers for active reference
4. `docs/paper_note_*.md` — five canonical papers, one note each
5. `docs/note_*.md` — attack-chain & defence notes
6. `docs/blog_drafts/*` — long-form pieces; the trust-hierarchy article is published (see below), the other two are drafts (v0.1)

## Projects & published writing

- **[Prompt Injection Detector & LLM Guard Gateway](https://github.com/GGaryyy/prompt-injection-detector)** — an offline, embedding-based prompt-injection detector (three-layer ensemble, ~0.97 F1 on public corpora) packaged as a fail-closed reverse-proxy gateway whose guards map to the OWASP LLM Top 10 (2025). The applied counterpart to these notes.
- **[The LLM Trust Hierarchy](https://ggaryyy.github.io/trust-hierarchy/)** ([中文版](https://ggaryyy.github.io/trust-hierarchy-zh/)) — published article on why L2–L3 injection is an under-priced attack surface, grounded in a first-party Gandalf / Agent-Breaker dataset and mapped to MITRE ATLAS.

## Status

| Topic | State |
|-------|------:|
| OWASP LLM Top 10 (2025) crosswalk | ✅ Complete |
| OWASP Agentic Top 10 (2026) summary | ✅ Complete |
| Greshake 2023 IPI deep read | ✅ Complete |
| Anthropic Many-shot Jailbreaking | ✅ Complete |
| Microsoft Crescendo | ✅ Complete |
| Memory poisoning real-world chain | ✅ Note complete |
| Structured-output defence | ✅ Note complete |
| LangGraph memory poisoning PoC | 🔀 Absorbed (2026-06-13) into a wider agentic-RAG pipeline project as its memory stage; the plan is kept for reference |
| LLM Trust Hierarchy article | ✅ Published |
| Blog drafts (BEC kill chain, MCP supply chain) | ✏️ Drafts (v0.1) |

## Notes on scope

- These are **personal study notes**, not a textbook. Expect rough edges and opinionated framing.
- All offensive technique discussion targets **public CTF environments** (Lakera Gandalf, Lakera Agent Breaker) or **conceptual / lab-only PoCs**. Nothing here is a tutorial for attacking live systems without authorization.
- The trust-hierarchy piece under `docs/blog_drafts/` is published (see Projects & published writing); the other two remain works in progress.

## License

[CC BY 4.0](LICENSE) — feel free to cite or build on, with attribution.
