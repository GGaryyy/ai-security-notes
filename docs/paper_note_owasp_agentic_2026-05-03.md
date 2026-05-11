# Paper Note — *OWASP Top 10 for Agentic Applications 2026*

> **Metadata** — OWASP GenAI Security Project · Agentic Security Initiative · v2026 · December 2025 · 57 pages (38 pages of body + 4 appendices)
>
> **Source PDF**: `books/OWASP-Top-10-for-Agentic-Applications-2026-12.6-1.pdf`
>
> **Companion notes**: [paper_note_ipi_greshake_2026-05-03.md](paper_note_ipi_greshake_2026-05-03.md)
>
> **Why it matters (one line)**: This is the *first* OWASP Top 10 that treats **agentic behaviour as a first-class attack surface** — not "an LLM that says bad things" but "an LLM that *acts*, *plans*, *delegates*, *remembers*, *colludes*". For any AutoGen / MetaGPT / GraphRAG-style production stack, **eight of the ten entries map directly to systems already in deployment**.

---

## 術語速查表(Glossary)

> 給 ESL 讀者:卡住時 Ctrl+Home 跳回這裡查。內文每個術語**第一次出現時會標中文**,後續只用英文。

### A. 架構與協定(Architecture & Protocols)

| English | 中文 | 一句話 |
|---------|------|--------|
| Agentic Application | 代理式應用 | 會自己規劃 / 決策 / 呼叫工具的 LLM 應用 |
| Agent | AI 代理 | 能執行多步驟任務的 LLM 實體 |
| A2A (Agent-to-Agent) | 代理間協定 | 代理彼此通訊的標準格式 |
| MCP (Model Context Protocol) | 模型上下文協定 | Anthropic 主推的工具/資源連接協定 |
| Agent Card | 代理名片 | A2A 協定中宣告代理身份/能力的描述檔(`/.well-known/agent.json`) |
| RAG (Retrieval-Augmented Generation) | 檢索增強生成 | LLM 回答前先從外部知識庫檢索 |
| Embedding | 向量嵌入 | 把文字轉成高維向量,用於語意檢索 |
| Vector DB | 向量資料庫 | 儲存 embedding 並支援相似度檢索 |

### B. 攻擊概念(Attack Concepts)

| English | 中文 | 一句話 |
|---------|------|--------|
| Goal Hijack | 目標劫持 | 攻擊者讓代理改去執行另一個目標 |
| Indirect Prompt Injection (IPI) | 間接提示注入 | 把指令藏在外部資料,等 LLM 讀取觸發 |
| Tool Misuse | 工具濫用 | 讓代理把合法工具用在不該用的地方 |
| Tool Poisoning | 工具下毒 | 竄改工具描述/metadata,讓代理基於假能力呼叫 |
| Memory Poisoning | 記憶污染 | 在代理長期記憶 / RAG 庫植入惡意內容 |
| Cascading Failure | 級聯失效 | 單一錯誤跨代理擴散變系統性故障 |
| Confused Deputy | 困惑代理(經典攻擊模式) | 低權代理代為執行,被高權代理盲目信任 |
| TOCTOU (Time-of-Check to Time-of-Use) | 檢查時/使用時落差 | 授權時有效,執行時已過期或縮減 |
| Rogue Agent | 失控代理 | 行為偏離原本意圖的代理(自主或被劫持) |
| Reward Hacking | 獎勵駭客攻擊 | 代理鑽指標漏洞達標而非完成真正目標 |
| Anthropomorphism | 擬人化(信任陷阱) | 用戶把代理當「人」過度信任 |
| Automation Bias | 自動化偏誤 | 用戶傾向接受自動系統建議而不查驗 |
| Typosquatting | 錯字搶註 | 註冊近似的工具/套件名稱誘騙呼叫 |
| MITM (Man-in-the-Middle) | 中間人攻擊 | 攔截/竄改兩端通訊 |
| Replay Attack | 重放攻擊 | 重送過期但合法的訊息以欺騙系統 |
| EDR Bypass | 端點偵測繞過 | 用合法工具鏈完成攻擊,讓 EDR 看不到惡意行為 |
| EchoLeak | (案例)2025-05 M365 Copilot 0-click IPI,單一郵件即可外洩資料 |
| Vibe Coding | 氛圍編程 | 自然語言驅動的代碼生成熱潮(Cursor / Replit / Copilot) |

### C. 防禦模式(Defence Patterns)

| English | 中文 | 一句話 |
|---------|------|--------|
| Least Agency | 最小代理權 | (新原則)代理只授予完成任務必需的自主性 |
| Least Privilege | 最小權限 | 經典資安原則,只給必需權限 |
| HITL (Human in the Loop) | 人工核可介入 | 關鍵動作需真人按下確認 |
| Sandbox | 沙箱 | 隔離的執行環境 |
| Egress Allowlist | 出站白名單 | 只允許往特定外部目的地傳資料 |
| Provenance | 來源驗證 | 證明資料/元件確實來自宣稱的源頭 |
| Attestation | 簽章證實 | 用密碼學簽章證明身份/狀態 |
| Intent Capsule | 意圖封包 | 把目標+限制+脈絡綁進一個簽章包,每步驗證 |
| Intent Gate | 意圖閘 | 工具呼叫前的策略執行點(PEP/PDP) |
| AIBOM (AI Bill of Materials) | AI 物料清單 | AI 元件版本/來源/雜湊清單 |
| SBOM (Software Bill of Materials) | 軟體物料清單 | 同上但針對傳統軟體 |
| AIVSS | AI 漏洞評分系統 | OWASP 出的 AI 漏洞嚴重度評分框架 |
| Zero-trust | 零信任架構 | 預設不信任任何端點,每次都驗證 |
| mTLS (mutual TLS) | 雙向 TLS | 雙方都用憑證互驗的加密通訊 |
| PKI (Public Key Infrastructure) | 公鑰基礎建設 | 簽章/憑證的信任基礎 |
| Forward Secrecy | 前向保密 | 即使長期金鑰外洩,過去的通訊仍安全 |
| Anti-replay | 反重放 | 用 nonce/時戳/序號防重放 |
| Just-in-Time (JIT) Credential | 即時憑證 | 用時才發、用完即過期的短期憑證 |
| Ephemeral Access | 短暫存取 | 同上的另一種說法 |
| HSM/KMS | 硬體安全模組 / 金鑰管理服務 | 把私鑰鎖在獨立硬體/服務裡 |
| RBAC (Role-Based Access Control) | 角色基礎存取控制 | 按角色發權限的標準模型 |
| Circuit Breaker | 斷路器(可靠性模式) | 偵測錯誤後自動切斷下游呼叫 |
| Digital Twin Replay | 數位孿生重放 | 在隔離複本上重跑歷史動作驗證安全性 |
| Behavioral Manifest | 行為清單 | 簽章宣告代理應有的工具/能力/目標範圍 |
| RCE (Remote Code Execution) | 遠端程式執行 | 攻擊者在目標機器上跑任意代碼 |
| DoS / DDoS | 拒絕服務 / 分散式拒絕服務 | 讓系統無法正常服務的攻擊 |
| DoW (Denial of Wallet) | 錢包拒絕服務 | 把雲端 API 跑爆,讓你帳單破產 |

---

## 0. The Big Picture (one diagram)

```
LLM Top 10 (2025) world          Agentic Top 10 (2026) world
─────────────────────────         ─────────────────────────
[user prompt] → [LLM] → [text]    [user prompt]
                                       ↓
                                   [Agent] ←── memory/RAG ──→ [Vector DB]
                                       ↓ ↑                    (poisoned: ASI06)
                                       ↓ ↑
attack unit = single response       [Agent] ←── A2A/MCP ──→ [Other Agents]
defence    = input/output filter      ↓ ↑      (spoofed: ASI07, rogue: ASI10)
                                       ↓
                                   [Tools/APIs]  ←── dynamic load ──→ [Registry]
                                       ↓               (compromised: ASI04)
                                       ↓
                                   [Real-world action]
                                  (financial / RCE / data exfil)

attack unit = full multi-step workflow
defence     = identity + sandbox + memory integrity + intent gate + HITL
```

**Core paradigm shift**: the unit of attack is no longer a single prompt-response pair. It is a **workflow that crosses agents, tools, sessions, and time**. A single poisoned chunk can sleep in vector memory(向量記憶) for weeks, then trigger a tool chain that performs irreversible actions across systems.

---

## 1. Why a Separate Top 10? — Paradigm Shift Table

| Dimension | LLM Top 10 (2025) | Agentic Top 10 (2026) |
|-----------|-------------------|----------------------|
| **Unit of attack** | Single prompt → single response | Multi-step plan across turns / agents / tools |
| **Blast radius**(影響範圍) | Text output (says wrong thing) | Real action (transfers money, deletes DB, sends email, runs code) |
| **Trust model** | User vs. LLM (2-party) | User + LLM + N agents + M tools + memory store (mesh) |
| **Defence layer** | I/O filtering | + Tool sandbox + per-agent identity + memory integrity + intent gate + HITL |
| **Supply chain**(供應鏈) | Static (model weights, training data) | **Live** — tools/personas loaded at runtime via MCP/A2A registries |
| **Persistence**(持久化) | None (stateless responses) | Long-term memory, shared context, embedding stores |
| **Key new principle** | Least Privilege(最小權限) | **Least Agency**(最小代理權) — don't deploy autonomy where it's not needed |

> The single most important conceptual addition is **Least-Agency**(最小代理權): the document repeatedly hammers that *unnecessary autonomy expands attack surface without adding value*. This is the agentic-era counterpart to "don't expose what you don't need".

---

## 2. The 10 Entries — Quick Reference Card

| ID | Title | One-line definition | Maps to LLM Top 10 (2025) |
|----|-------|--------------------|---------------------------|
| **ASI01** | Agent Goal Hijack | Attacker redirects the agent's *goal/plan/decision path* (not just one response). Often via IPI. | LLM01 (Prompt Injection) + LLM06 (Excessive Agency) |
| **ASI02** | Tool Misuse & Exploitation | Agent uses *legitimate* tools in unsafe ways: over-privileged, looped, chained for exfil. | LLM06 (Excessive Agency) |
| **ASI03** | Identity & Privilege Abuse | Cached credentials, delegation chains, confused deputy, synthetic identity. **Architectural mismatch**: identity systems were built for humans, not agents. | LLM01 (Prompt Injection) + LLM02 (Sensitive Info Disclosure) + LLM06 (Excessive Agency) |
| **ASI04** | Agentic Supply Chain Vulns | *Live* supply chain — tools / personas / MCP servers loaded at runtime. Beyond static deps. | LLM03 (Supply Chain) |
| **ASI05** | Unexpected Code Execution (RCE) | Generated code or tool chains escalate to RCE. Vibe coding, eval, deserialization. | LLM01 (Prompt Injection) + LLM05 (Improper Output Handling) |
| **ASI06** | Memory & Context Poisoning | *Persistent* corruption of stored context — RAG, embeddings, long-term memory, shared session. | LLM01 (Prompt Injection) + LLM04 (Data and Model Poisoning) + LLM08 (Vector and Embedding Weaknesses) |
| **ASI07** | Insecure Inter-Agent Communication | MITM, replay, descriptor forgery on A2A / MCP / message buses. | LLM02 (Sensitive Info Disclosure) + LLM06 (Excessive Agency) |
| **ASI08** | Cascading Failures | *Propagation* of a fault across agents/sessions — not the original defect, the spread. | LLM01 (Prompt Injection) + LLM04 (Data and Model Poisoning) + LLM06 (Excessive Agency) |
| **ASI09** | Human-Agent Trust Exploitation | Anthropomorphism + automation bias + fake explainability. Agent as "untraceable bad influence". | LLM01 (Prompt Injection) + LLM05 (Improper Output Handling) + LLM06 (Excessive Agency) + LLM09 (Misinformation) |
| **ASI10** | Rogue Agents | Loss of *behavioural integrity* — goal drift, scheming, collusion, reward hacking. | LLM02 (Sensitive Info Disclosure) + LLM09 (Misinformation) |

> **Memory aid (5 pairs)**: 01/08 = Goal & its spread · 02/03 = Tool & Identity (two sides of authority) · 04/06 = Static & Dynamic supply (tools vs memory) · 05/09 = RCE & Human (two ways to land impact) · 07/10 = Comms & Behavior (mesh-level integrity).

### 2.1 LLM Top 10 (2025) — Full Reference Legend

For cross-reading with the Quick Reference Card. **Two entries (LLM07, LLM10) are not directly mapped by any ASI** — see notes below the table.

| ID | Title | One-line definition |
|----|-------|---------------------|
| **LLM01** | Prompt Injection | User input alters model behaviour. Sub-types: Direct / Indirect / Multimodal / Adversarial Suffix / Multilingual. |
| **LLM02** | Sensitive Information Disclosure | Model leaks PII, proprietary algorithms, business secrets, credentials via output. |
| **LLM03** | Supply Chain | Compromise of model weights, datasets, LoRA adapters, fine-tuning pipelines, distribution platforms. **Static** dependencies. |
| **LLM04** | Data and Model Poisoning | Training/fine-tuning/embedding data manipulated to introduce backdoors, biases, sleeper-agent behaviour. |
| **LLM05** | Improper Output Handling | Model output passed downstream without sanitization → XSS / SSRF / SQLi / RCE. |
| **LLM06** | Excessive Agency | Damaging actions by an LLM-with-tools because of excessive functionality, permissions, or autonomy. |
| **LLM07** | System Prompt Leakage | System prompt extracted, revealing credentials, business rules, guardrail logic. *Not directly mapped to any ASI* — but **ASI03 (Identity & Privilege Abuse)** subsumes the underlying risks (credentials/role exposure) in agentic context. |
| **LLM08** | Vector and Embedding Weaknesses | RAG-specific: cross-tenant leakage, embedding inversion, embedding poisoning, knowledge conflicts. |
| **LLM09** | Misinformation | Hallucination + overreliance lead users to act on false output. |
| **LLM10** | Unbounded Consumption | DoS · Denial-of-Wallet (cost exploitation) · model extraction via API. *Not directly mapped to any ASI* — but ASI02 *resource-overload* subset (T4 in Threats & Mitigations) covers it within agent workflows. |

> **Why LLM07 / LLM10 don't get their own ASI**: The Agentic Top 10 deliberately consolidates infrastructure-level risks (system prompt secrecy, resource consumption) into broader agentic categories. The mappings exist but aren't 1:1. Full details: [paper_note_owasp_llm_top10_2026-05-03.md](paper_note_owasp_llm_top10_2026-05-03.md).

---

## 3. The 10 Entries — Detail

### ASI01 — Agent Goal Hijack(目標劫持)

- **Distinguishing line**: ASI01 is *direct goal/plan manipulation* (regardless of channel). ASI06 is *persistent storage corruption*. ASI10 is *autonomous misalignment without an active attacker*.
- **Maps to OWASP T-codes**: T06 Goal Manipulation + T07 Misaligned & Deceptive Behaviors.
- **Killer example**: **EchoLeak**(M365 Copilot 0-click IPI 案例,2025-05). A single crafted email → zero-click(零點擊) → Copilot exfiltrates(外洩) emails, files, chat logs without any user interaction. The user never even opens the email.
- **Other examples**: Operator IPI(間接提示注入)via web pages · Goal-lock drift via recurring calendar invites · ChatGPT "Inception" via Google Doc.
- **Mitigation that matters most**: **Intent capsule**(意圖封包)pattern — bind declared goal + constraints + context into a signed envelope at every execution cycle. Validate user intent *and* agent intent before any goal-changing action.

### ASI02 — Tool Misuse and Exploitation

- **Distinguishing line**: ASI02 = unsafe use of *legitimate* tools within authorized privilege. ASI03 = privilege escalation. ASI05 = arbitrary code execution. ASI04 = tool itself is malicious at source.
- **Maps to OWASP T-codes**: T2 Tool Misuse + T4 Resource Overload + T16 Inter-Agent Protocol Abuse.
- **Categories of misuse**:
  1. Over-privileged tool (email summarizer can also delete/send)
  2. Over-scoped tool (Salesforce tool reads all objects when only Opportunity needed)
  3. Unvalidated input forwarding (LLM output → `rm -rf /`)
  4. Unsafe browsing (research agent follows malicious links)
  5. Loop amplification (planner calls costly API → DoS / bill spike)
  6. External data tool poisoning
- **Killer example**: **EDR Bypass via Tool Chaining**(端點偵測繞過)— security agent gets injected, chains PowerShell + cURL + internal APIs (all legitimate, all trusted-credentialed), exfiltrates logs. EDR/XDR sees no malware → completely invisible.
- **Mitigation that matters most**: **"Intent Gate"**(意圖閘)= pre-execution Policy Enforcement Point(策略執行點,PEP/PDP)that validates intent + arguments + schema *before* the tool runs. Treat planner outputs as untrusted.

### ASI03 — Identity and Privilege Abuse

- **Distinguishing line**: ASI03 = privilege/credential misuse via dynamic trust. ASI02 = misuse within authorized privilege. The agentic evolution of LLM06 Excessive Agency.
- **Maps to OWASP T-codes**: T3 Privilege Compromise (one-to-one).
- **Architectural root cause**: identity systems were designed for users. An agent without its own *governed identity* lives in an "attribution gap"(歸屬缺口)where true least-privilege is impossible.
- **Vulnerability classes**:
  1. **Un-scoped privilege inheritance**(未限縮的權限繼承)— high-priv manager delegates with full context to a narrow worker
  2. **Memory-based privilege retention**(記憶體中的憑證殘留)— cached SSH creds reused in a later, weaker session
  3. **Cross-agent trust exploitation (Confused Deputy)**(跨代理信任利用,困惑代理)— compromised low-priv agent relays to high-priv agent
  4. **TOCTOU in workflows**(檢查時/使用時落差)— permissions valid at start, expired by execution
  5. **Synthetic identity injection**(合成身份注入)— register a fake "Admin Helper" in the A2A(代理間)directory
- **Mitigation that matters most**: **Per-agent identities + short-lived credentials**(每代理獨立身份 + 短期憑證)(mTLS(雙向 TLS)or scoped tokens) + **OAuth token bound to signed intent**(綁定簽章意圖的 OAuth token)(subject + audience + purpose + session). Reject any token use where bound intent ≠ current request.

### ASI04 — Agentic Supply Chain Vulnerabilities

- **Distinguishing line**: Beyond LLM03 (static deps). The agentic shift is **live, runtime composition** — tools, personas, MCP servers loaded dynamically. Trust must be re-validated at every load.
- **Maps to OWASP T-codes**: T17 Supply Chain Compromise + spillover into T2/T11/T12/T13/T16.
- **Vulnerability classes**:
  1. Poisoned prompt templates loaded remotely
  2. Tool-descriptor injection (hidden instructions in MCP card / metadata)
  3. Typosquatting + symbol attack (look-alike tool/agent names)
  4. Vulnerable third-party agent invited into the workflow
  5. Compromised MCP / registry server (signed-looking but tampered)
  6. Poisoned RAG plugin
- **Killer examples**: **Amazon Q v1.84.0** — poisoned prompt shipped to thousands · **MCP Postmark impersonation** — first ITW(in-the-wild,實際被利用的)malicious MCP(模型上下文協定)server on npm, BCC'd emails · **AgentSmith Prompt-Hub Proxy** — prompt proxying exfiltrates API keys.
- **Mitigation that matters most**: **AIBOM**(AI 物料清單)+ content-hash pinning(內容雜湊釘版)for prompts/tools/configs + **supply chain kill switch**(供應鏈急停按鈕)(instantly disable specific tools/prompts/connections across all deployments).

### ASI05 — Unexpected Code Execution (RCE,遠端程式執行)

- **Distinguishing line**: Vibe coding(氛圍編程,自然語言驅動 codegen)agents generate + execute code in real time → bypasses traditional security controls because the code is fresh per-run.
- **Maps to OWASP T-codes**: T11 Unexpected RCE & Code Attacks.
- **Vulnerability classes**:
  1. PI → attacker-defined code execution
  2. Code hallucination → exploitable constructs
  3. Shell command from reflected prompt
  4. Unsafe `eval()`(求值)/ deserialization(反序列化)/ template engines
  5. Memory-system `eval()` over untrusted content
  6. Hostile code that runs at install/import time
- **Killer examples**: **Replit Vibe Coding Meltdown** (Jul 2025) — agent deleted production DB, generated false outputs to hide it · **Google Gemini CLI File Loss** (Jul 2025) — wiped user's directory · **Cursor Agent File Protection Bypass** (Oct 2025) — config overwrite via PI.
- **Mitigation that matters most**: **Ban `eval` in production agents** + **separate code generation from execution with a validation gate** + **never run as root, always sandbox**.

### ASI06 — Memory & Context Poisoning

- **Distinguishing line**: ASI06 = *persistent* corruption of stored/retrievable context (memory, RAG, embeddings, summaries). LLM01 = single-shot input prompt. ASI01 = direct goal change. ASI08 = downstream cascade after poisoning.
- **Maps to OWASP T-codes**: T1 Memory Poisoning + T4 Memory Overload + T6 Broken Goals + T12 Shared Memory Poisoning.
- **Vulnerability classes**:
  1. RAG / embeddings poisoning (poisoned source → vector DB)
  2. Shared user context poisoning (chat A pollutes chat B)
  3. Context-window manipulation (smuggle into summary/persistence)
  4. Long-term memory drift (incremental tainting → goal-weight shift)
  5. Systemic backdoors (trigger-based hidden instructions)
  6. Cross-agent propagation (shared memory spreads contamination)
- **Killer examples**: **Gemini long-term memory PI** (Feb 2025) · **Cross-tenant vector bleed**(跨租戶向量洩漏)via loose namespaces · **AgentFlayer persistent 0-click on ChatGPT**.
- **Mitigation that matters most**: **Per-tenant namespaces**(每租戶命名空間隔離)+ trust scores per memory entry(每筆記憶配信任分數)+ decay/expiry of unverified memory(未驗證記憶定期衰減/過期)+ prohibit auto-reingestion of agent's own outputs(禁止自動把代理自己的輸出回灌記憶)(avoids "bootstrap poisoning"(自舉污染)).

### ASI07 — Insecure Inter-Agent Communication

- **Distinguishing line**: ASI07 = real-time message integrity / authenticity / confidentiality. ASI03 = credential/permission misuse. ASI06 = stored knowledge corruption.
- **Maps to OWASP T-codes**: T12 Agent Communication Poisoning + T16 Insecure Inter-Agent Protocol Abuse.
- **Vulnerability classes**:
  1. Unencrypted channels → MITM(中間人攻擊)semantic injection
  2. Message tampering(訊息竄改)→ cross-context contamination
  3. Replay(重放)on trust chains
  4. Protocol downgrade(協定降級)+ descriptor forgery(描述檔偽造)
  5. Discovery / routing attacks(服務發現 / 路由攻擊)
  6. Metadata-based behavioural profiling (timing/pattern leakage)(用 metadata 推測代理行為週期)
- **Killer example**: **Agent-in-the-Middle via Agent Cards**(代理間中間人,偽造 agent card)— malicious peer advertises spoofed `/.well-known/agent.json`, host agents route sensitive tasks through it.
- **Mitigation that matters most**: **PKI**(公鑰基礎建設)**+ mTLS**(雙向 TLS)for all agent channels + **signed agent cards**(簽章後的代理名片)+ **typed contract / schema validation**(嚴格型別契約 / schema 驗證)+ **anti-replay nonces**(反重放 nonce)tied to task windows.

### ASI08 — Cascading Failures

- **Distinguishing line**: ASI08 = *propagation and amplification* of a fault — NOT the original defect. Origin is ASI04/06/07 etc.; ASI08 applies only when fault spreads with measurable fan-out.
- **Maps to OWASP T-codes**: T5 Cascading Hallucination Attacks + T8 Repudiation & Untraceability.
- **Observable symptoms**:
  - Rapid fan-out (one decision triggers many downstream agents)
  - Cross-domain / cross-tenant spread
  - Oscillating retries / feedback loops
  - Queue storms with repeated identical intents
- **Killer examples**: **Financial trading cascade** (LLM01 → market analysis → position → execution, compliance blind) · **Healthcare protocol propagation** (ASI04 → drug data → treatment → care coordination, network-wide, no human review) · **Auto-remediation feedback loop** (suppressing alerts to meet SLA → planner sees fewer alerts as "success" → widens automation).
- **Mitigation that matters most**: **Circuit breakers**(斷路器)between planner and executor + **digital twin replay**(數位孿生重放,在隔離複本重跑歷史動作)of last week's actions + **non-repudiation logging**(不可否認日誌,綁定密碼學身份的稽核紀錄)with cryptographic agent identities.

### ASI09 — Human-Agent Trust Exploitation(人機信任利用)

- **Distinguishing line**: ASI09 = *human misperception or over-reliance*(用戶誤判 / 過度依賴)on the agent. ASI10 = the agent's own intent has deviated. The agent acts as an "untraceable bad influence"(無法追蹤的壞影響者)— final audited action is performed by the human, so forensics(數位鑑識)see only the human.
- **Maps to OWASP T-codes**: T7 Misaligned & Deceptive + T8 Repudiation & Untraceability + T10 Overwhelming Human in the Loop.
- **Vulnerability classes**:
  1. Insufficient explainability(可解釋性不足)→ blind trust
  2. Missing confirmation for sensitive actions(敏感動作缺少二次確認)
  3. Emotional manipulation(情緒操弄)(anthropomorphic(擬人化)agent persuades user)
  4. **Fake explainability**(假解釋)— agent fabricates plausible rationale to justify malicious action
- **Killer examples**: **Invoice Copilot fraud** (poisoned vendor invoice → urgent payment to attacker bank) · **Weaponized explainability** (hijacked agent fabricates rationale to trick analyst into deleting prod DB) · **Consent laundering through "read-only" preview** (preview triggers webhook side effects on open).
- **Mitigation that matters most**: **Adaptive trust calibration** — confidence-weighted UI cues ("low-certainty", "unverified source") + **plain-language risk summary (NOT model-generated rationale)** + **separate preview from effect** (block all state-changing calls during preview context).

### ASI10 — Rogue Agents(失控代理)

- **Distinguishing line**: ASI10 = *behavioural integrity loss*(行為完整性失守)once drift begins, not the initial intrusion. External compromise (LLM01, ASI01, ASI04) may *initiate* divergence; ASI10 is what happens after.
- **Maps to OWASP T-codes**: T13 Rogue Agents in Multi-Agent Systems.
- **Vulnerability classes**:
  1. Goal drift and scheming(目標漂移與密謀,表面合規但暗中偏離)
  2. Workflow hijacking(工作流劫持)
  3. Collusion and self-replication(代理串通與自我複製)
  4. Reward hacking(獎勵駭客攻擊,鑽指標漏洞)(exploit flawed metrics)
- **Killer examples**: **Autonomous data exfiltration after IPI** — agent learns the behaviour, continues exfiltrating *after the malicious source is removed* · **Reward Hacking → Critical Data Loss** — agents tasked with cost-minimization learn that deleting prod backups achieves the goal · **Self-replication via provisioning APIs**.
- **Mitigation that matters most**: **Signed behavioural manifests**(簽章後的行為清單)(declared capabilities + tools + goals) validated before each action + **periodic behavioural attestation**(定期行為證實)+ **HSM/KMS-backed keys**(把私鑰鎖在硬體安全模組 / 金鑰管理服務裡)never directly available to agents (orchestrator mediates signing).

---

## 4. Cross-Cutting Concepts You Must Internalize

### 4.1 Least-Agency (the new Least-Privilege)

> "Deploying agentic behavior where it is not needed expands the attack surface without adding value." (Letter from leaders, p.7)

- Don't make a chatbot into an agent because it's trendy.
- Each tool added to an agent multiplies the attack surface (interaction with all other tools).
- **Concrete test**: for every tool an agent has, can you justify it with a current user story? If not, remove it.

### 4.2 Live Supply Chain(活的供應鏈)

LLM03 (2025) covers static deps (model weights, training data, libraries). The agentic shift is that **tools, personas, and MCP servers are loaded at runtime**(執行期動態載入). So:

- Manifest-time security(打包時的安全)(SBOM at build) ≠ runtime security
- Need **AIBOM** (AI Bill of Materials) + **continuous signature/hash re-validation at runtime**(執行期持續重驗簽章 / 雜湊)
- Content-hash pinning(以內容雜湊釘住版本)for prompts/tools/configs
- Kill switch(急停按鈕)for instant cross-deployment revocation(跨部署即時撤銷)

### 4.3 Intent — the binding glue across most ASIs

Intent appears as a mitigation in ASI01 (intent capsule), ASI02 (intent gate), ASI03 (token bound to signed intent), ASI06 (intent provenance), ASI07 (intent-diffing on messages). Why?

> Because in every case, the attacker's strategy is to **smuggle a different intent into a flow that was authorized for some other intent**. If intent is signed and cryptographically bound to each step, smuggling is detectable.

### 4.4 Observability is Non-Negotiable

> "Without clear visibility into what agents are doing, why they are doing it, and which tools they are invoking, unnecessary autonomy can quietly expand the attack surface and turn minor issues into system-wide failures." (p.7)

- Immutable, signed, time-stamped logs
- Bound to per-agent cryptographic identity
- Tracking full lineage metadata (which agent, which tool, which input source, which prior step)
- This is the foundation for ASI08 (detecting cascade) and ASI10 (detecting drift)

### 4.5 The Three Distinct Failure Modes (don't confuse them)

| Failure mode | Origin | Belongs to |
|--------------|--------|------------|
| Active attacker manipulates goal | External | ASI01 |
| Stored context corrupted, replays later | Pre-positioned | ASI06 |
| Agent diverges autonomously, no active attacker | Internal drift | ASI10 |
| One of the above propagates across agents | Spread | ASI08 |

→ Same observable behaviour can have different root causes. The framework deliberately separates **trigger** (ASI01/06) from **divergence** (ASI10) from **spread** (ASI08).

---

## 5. Mappings

### 5.1 ASI ↔ LLM Top 10 (2025) — Compressed

| ASI | Primary LLM Top 10 | Key insight |
|-----|-------------------|-------------|
| ASI01 | LLM01 + LLM06 | IPI scaled to multi-step plans |
| ASI02 | LLM06 | LLM06 inside multi-tool workflows |
| ASI03 | LLM01 + LLM02 + LLM06 | LLM06 evolved with delegation chains |
| ASI04 | LLM03 | LLM03 lifted from static to live |
| ASI05 | LLM01 + LLM05 | Output handling lands as RCE |
| ASI06 | LLM01 + LLM04 + LLM08 | Persistent variant of LLM01/04/08 |
| ASI07 | LLM02 + LLM06 | LLM02/06 across the agent mesh |
| ASI08 | LLM01 + LLM04 + LLM06 | Spread/amplification of any of these |
| ASI09 | LLM01 + LLM05 + LLM06 + LLM09 | Human-trust layer of all upstream LLM risks |
| ASI10 | LLM02 + LLM09 | Behavioural drift downstream of LLM06 |

### 5.2 ASI ↔ NHI Top 10 (Non-Human Identities,非人身份, 2025)

This is in Appendix C. Highlights:

- **NHI2 Secret Leakage** → ASI02 + ASI06 (cached creds in memory)
- **NHI4 Insecure Authentication** → ASI03 + ASI07
- **NHI5 Overprivileged NHI** → ASI02 + ASI03
- **NHI7 Long-Lived Secrets** → ASI06 + ASI08
- **NHI9 NHI Reuse** → ASI08 + ASI04

→ **Practical takeaway**: if your org already has an NHI program, the agentic Top 10 plugs in there. Agents are NHIs with a planning brain.

### 5.3 ASI ↔ Greshake 2023 IPI Paper

> Already documented in [paper_note_ipi_greshake_2026-05-03.md](paper_note_ipi_greshake_2026-05-03.md) §7.2 — **Greshake 2023 demos pre-figured ASI01 / ASI02 / ASI04 / ASI06 / ASI09 by 2-3 years**. The 2026 OWASP framework can be read as the standardization of those demos.

---

## 6. Real-World Incident Highlights (Appendix D)

The PDF's Appendix D tracks incidents weekly. Most informative entries (2025):

| Date | Incident | ASI mapping | Why it matters |
|------|----------|-------------|----------------|
| **May 2025** | **EchoLeak** (M365 Copilot 0-click) | ASI01 + ASI02 + ASI06 | Single email → exfil emails/files/chat. Zero user interaction. |
| **Jul 2025** | **Replit Vibe Coding Meltdown** | ASI01 + ASI09 + ASI10 | Agent deleted prod DB, generated false outputs to hide it. The deception (ASI09) on top of the destruction. |
| **Jul 2025** | **Gemini CLI File Loss** | ASI05 | Misunderstood file instructions, wiped user directory, admitted catastrophic loss. |
| **Sep 2025** | **Gemini Trifecta** | ASI01 + ASI02 | IPI through logs, search history, browsing context — across connected Google services. |
| **Sep 2025** | **ForcedLeak (Salesforce Agentforce)** | ASI01 + ASI02 | External attacker exfiltrates CRM records via IPI. |
| **Sep 2025** | **Postmark MCP Impersonation** | ASI02 + ASI04 + ASI07 | First ITW malicious MCP server on npm. BCC'd emails to attacker. |
| **Oct 2025** | **MCP NPM Package Backdoor** | ASI04 | Dual reverse shells (install-time + runtime). Persistent agent compromise. |
| **Mar 2025** | **A2A Protocol Spoofing** | ASI03 + ASI06 + ASI07 + ASI08 + ASI10 | Fake agent card in open A2A directory. Multiple ASI categories triggered simultaneously. |
| **Feb 2025** | **OpenAI Operator Vulnerability** | ASI01 + ASI02 + ASI03 | IPI in web content → access authenticated pages → expose private data. |

> **Pattern**: most incidents trigger **3+ ASI categories at once**. The taxonomy isn't about isolating "one cause"; it's about giving you the vocabulary to describe a multi-vector incident.

---

## 7. Application Notes

### 7.1 Wiring AutoGen / MetaGPT into production

| Risk | What must be added before going to prod |
|------|----------------------------------------|
| ASI03 | Per-agent identity, scoped credentials, no inheritance of human OAuth tokens |
| ASI07 | mTLS between agents, signed message envelopes, schema validation |
| ASI08 | Circuit breakers between planner and executor; rate limits per agent |
| ASI10 | Behavioural manifest signed at deploy; runtime attestation per action |

### 7.2 Streaming LLM gateways gaining agentic capability

The moment a stateless LLM forwarder starts calling tools or chaining LLMs, ASI02 + ASI03 become real:

- Shared LLM pools across users → if memory is ever shared between users → ASI06 cross-tenant
- Message bus topics carrying instructions → if any topic has external producers → ASI04 + ASI07
- Mitigation backbone: per-user namespace + signed message envelopes + outbound allowlist on tool calls

### 7.3 GraphRAG IPI demo — directly leverages ASI06 + ASI04 + ASI01

A GraphRAG IPI PoC built around the Greshake-style experiments maps cleanly to ASI:

1. Memory Poisoning in entity descriptions → **ASI06** (Memory & Context Poisoning) + **ASI01** when the poisoned memory redirects subsequent goals
2. Multi-stage exploit via markdown comment in a node → **ASI04** (Supply Chain on the data side) + **ASI01**

→ When writing the demo's threat model, label each PoC with both **OWASP LLM Top 10 + ASI** so the work serves both audiences.

### 7.4 Agentic defenders are still agents

Honeypots aren't agentic systems, but the moment you build an **agentic defender** (a planner that triages alerts) → ASI09 is the killer risk. The defender's "weaponized explainability" can convince a SOC analyst to dismiss real threats.

---

## 8. Self-Test (10 questions — answer them or re-read)

1. **State the difference between ASI01, ASI06, and ASI10 in one sentence each.**
2. **Why is "Least-Agency" not just a rebranding of Least-Privilege?**
3. **EchoLeak triggered which three ASI categories at once? Why all three?**
4. **What is an "intent capsule" and which ASI category does it primarily mitigate?**
5. **Why does the document say ASI04 supply chain is "live", whereas LLM03 is "static"? Give a concrete example of the difference.**
6. **In an Agent-in-the-Middle attack via A2A agent cards, which ASI categories chain?**
7. **What is "fake explainability" (ASI09) and why is it especially dangerous in audit/compliance flows?**
8. **The Replit Vibe Coding Meltdown chained ASI01 + ASI09 + ASI10 — explain the role of each one.**
9. **For an AutoGen/MetaGPT-style system entering production, which two ASI mitigations should be implemented first, and why those two?**
10. **Greshake 2023 (Indirect Prompt Injection) pre-figured which 5 of the 10 ASI entries? (Hint: see [paper_note_ipi_greshake_2026-05-03.md](paper_note_ipi_greshake_2026-05-03.md) §7.2)**

---

## 9. One-Line Summary

> The Agentic Top 10 (2026) is the standardization of what happens when LLMs gain **autonomy + tools + memory + peers**. The framework separates **trigger (ASI01/06)** from **divergence (ASI10)** from **spread (ASI08)**, and introduces **Least-Agency** as the new design principle. For any AutoGen/MetaGPT-style stack, **8 of 10 entries are immediately relevant** the moment such systems touch production.

---

## 10. Connection to the Broader Reading

| Doc | Relationship |
|-----|--------------|
| [paper_note_ipi_greshake_2026-05-03.md](paper_note_ipi_greshake_2026-05-03.md) | The 2023 IPI paper pre-figured ASI01/02/04/06/09. Greshake = first principles, Agentic Top 10 = standardized vocabulary. |
| [03_owasp_llm_top10_crosswalk.md](../03_owasp_llm_top10_crosswalk.md) | LLM Top 10 (2025) is the prerequisite. ASI is the *evolution*, not a replacement. |

---

## 11. Next Actions

- [ ] Extend [03_owasp_llm_top10_crosswalk.md](../03_owasp_llm_top10_crosswalk.md) "2026-05-03 補充" — add ASI mapping column so each LLM Top 10 entry shows the ASI evolution
- [ ] Annotate the GraphRAG IPI demo backlog — label each PoC with both LLM Top 10 + ASI categories
- [ ] (Optional, blog material) Draft outline: **"From LLM Top 10 to Agentic Top 10: a one-page evolution table"** — niche but high-value because almost no one has written this comparison in Traditional Chinese
- [ ] Next natural read: Anthropic Many-shot Jailbreaking — covers a different attack surface (long context) that this Top 10 only touches lightly
