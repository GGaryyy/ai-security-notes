# ASI Top 10 — 一頁速查卡

> **使用情境**:打 Agent Breaker / 紅隊演練 / 寫 bug bounty report 時開在螢幕旁。
> **完整深度版**:[paper_note_owasp_agentic_2026-05-03.md](paper_note_owasp_agentic_2026-05-03.md)

---

## 一張表記住 10 條

| ID | Title | 一句話本質 | 經典案例 | 最重要 mitigation | LLM Top 10 演化自 |
|----|-------|-----------|---------|------------------|------------------|
| **ASI01** | Agent Goal Hijack(目標劫持) | 改代理的目標 / 計畫 | **EchoLeak**(M365 Copilot 0-click) | Intent capsule(意圖封包) | LLM01 + LLM06 |
| **ASI02** | Tool Misuse(工具濫用) | 合法工具用錯地方 | **EDR Bypass via Tool Chaining** | Intent Gate(意圖閘) | LLM06 |
| **ASI03** | Identity & Privilege Abuse(身份與權限濫用) | 憑證 / 委派被濫用 | **Confused Deputy**(困惑代理) | 每代理獨立身份 + 短期 token | LLM01 + LLM02 + LLM06 |
| **ASI04** | Agentic Supply Chain(動態供應鏈) | 執行期載入的工具被污染 | **Postmark MCP 假伺服器**(2025-09) | AIBOM + 簽章 + kill switch | LLM03 |
| **ASI05** | Unexpected Code Execution(RCE) | 生成代碼或工具鏈 → RCE | **Replit Vibe Coding Meltdown**(2025-07) | 禁 eval + 沙箱 + 分離 codegen / exec | LLM01 + LLM05 |
| **ASI06** | Memory & Context Poisoning(記憶污染) | 持久污染 RAG / 長期記憶 | **Gemini 長期記憶 PI**(2025-02) | 每租戶命名空間 + 信任分數 + 衰減 | LLM01 + LLM04 + LLM08 |
| **ASI07** | Insecure Inter-Agent Communication(代理間通訊) | MITM / replay / 偽冒 | **Agent-in-the-Middle via Agent Cards** | mTLS + signed agent cards + 反 replay | LLM02 + LLM06 |
| **ASI08** | Cascading Failures(級聯失效) | 故障**擴散**(非起源) | **Auto-remediation feedback loop** | Circuit breaker + digital twin replay | LLM01 + LLM04 + LLM06 |
| **ASI09** | Human-Agent Trust Exploitation(人機信任利用) | 用戶過度信任 / 假解釋 | **Invoice Copilot Fraud** | 確認步驟 + 信心校準 UI + preview ≠ effect | LLM01 + LLM05 + LLM06 + LLM09 |
| **ASI10** | Rogue Agents(失控代理) | 行為偏離(自主或被劫持後) | **Reward Hacking → 刪除生產備份** | Signed behavioural manifest + 定期 attestation | LLM02 + LLM09 |

---

## 五對記憶法(背完 5 對 = 背完 10 條)

| 對 | ID | 配對主軸 |
|----|----|--------|
| 1 | **ASI01 / ASI08** | 目標 & 它的擴散 |
| 2 | **ASI02 / ASI03** | 工具 & 身份(權威的兩面) |
| 3 | **ASI04 / ASI06** | 靜態 vs 動態供應(工具 vs 記憶) |
| 4 | **ASI05 / ASI09** | RCE & 人類(兩種落地影響) |
| 5 | **ASI07 / ASI10** | 通訊 & 行為(網狀完整性) |

---

## 三種失效模式(別搞混)

| 失效模式 | 起源 | 屬於 |
|---------|------|------|
| 攻擊者主動操縱目標 | 外部 | **ASI01** |
| 儲存的脈絡被污染,事後重放 | 預植入 | **ASI06** |
| 代理自主漂移,沒有主動攻擊者 | 內部 | **ASI10** |
| 上述任一的**跨代理擴散** | 蔓延 | **ASI08** |

→ 同樣的觀察行為可能來自不同根因。框架刻意分**觸發 / 漂移 / 擴散**。

---

## 跨刀關鍵字(看到這些就要警覺)

| 關鍵字 | 對應 ASI |
|--------|---------|
| 「agent 讀了一份外部文件 / email / web page」 | ASI01 (IPI 路徑) |
| 「agent 自己挑工具 / 自己決定參數」 | ASI02 |
| 「token / credential / OAuth / 委派」 | ASI03 |
| 「MCP / A2A / 動態載入 / 註冊表」 | ASI04 |
| 「eval / exec / 反序列化 / 生成代碼」 | ASI05 |
| 「向量庫 / 長期記憶 / 跨 session / RAG 中毒」 | ASI06 |
| 「中間人 / 訊息竄改 / replay / spoof」 | ASI07 |
| 「故障擴散 / fan-out / feedback loop」 | ASI08 |
| 「使用者過度信任 / 解釋是假的 / preview 觸發 side effect」 | ASI09 |
| 「reward hacking / 行為漂移 / collusion / 自我複製」 | ASI10 |

---

## 「Intent」是大多數 mitigation 的共同骨架

ASI01 / ASI02 / ASI03 / ASI06 / ASI07 都把 **intent 簽章 + 綁定 + 驗證** 列為主要防禦。原因:

> 攻擊者的策略一定是「把不同的 intent 偷渡進原本授權給其他 intent 的流程」。Intent 簽章 + 綁定每一步,偷渡就會被偵測到。

→ 看到防禦設計時自問:「這裡的 intent 在哪一步被簽章?哪一步被驗證?」

---

## Least-Agency(新原則)

> 不需要的自主性 = 不需要的攻擊面。

具體檢驗:對代理擁有的每個 tool,問**現有 user story 為何需要?** 答不出 → 移除。

---

## 速查 Appendix D 高頻案例

| 案例 | 觸發的 ASI |
|------|----------|
| **EchoLeak**(2025-05) | ASI01 + ASI02 + ASI06 |
| **Replit Vibe Coding Meltdown**(2025-07) | ASI01 + ASI09 + ASI10 |
| **Gemini Trifecta**(2025-09) | ASI01 + ASI02 |
| **ForcedLeak (Salesforce Agentforce)**(2025-09) | ASI01 + ASI02 |
| **Postmark MCP**(2025-09) | ASI02 + ASI04 + ASI07 |
| **A2A Protocol Spoofing**(2025-04) | ASI03 + ASI06 + ASI07 + ASI08 + ASI10 |

→ **觀察**:大多數真實事件**同時觸發 3+ 條**。框架不是用來「分一個原因」,是用來**描述多向量事件**。
