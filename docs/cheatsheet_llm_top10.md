# LLM Top 10 (2025) — 一頁速查卡

> **使用情境**:寫 bug bounty report / 紅隊演練 / blog / talk 時,引用標準術語。
> **完整深度版**:[paper_note_owasp_llm_top10_2026-05-03.md](paper_note_owasp_llm_top10_2026-05-03.md)
> **深度對照(LLM01/06/07/08)**:[03_owasp_llm_top10_crosswalk.md](../03_owasp_llm_top10_crosswalk.md)

---

## 一張表記住 10 條

| ID | Title | 一句話本質 | 經典案例 | 最重要 mitigation | ASI 演化(2026) |
|----|-------|-----------|---------|------------------|----------------|
| **LLM01** | Prompt Injection(提示注入) | 用戶輸入改變模型行為(Direct / Indirect / Multimodal) | **Greshake 2023 IPI 攻擊** · Bing Sydney | 限制行為 + 區隔外部內容 + HITL | ASI01 + ASI06 |
| **LLM02** | Sensitive Information Disclosure(敏感資訊洩漏) | 模型成為洩漏通道(訓練資料 / RAG / chat 記憶) | **ChatGPT Samsung leak** · 「Repeat poem forever」 | 資料淨化 + 不訓練用戶資料 + 差分隱私 | ASI03(身份濫用衍生) |
| **LLM03** | Supply Chain(供應鏈) | 第三方元件被污染(模型 / 資料 / LoRA / 平台) | **Shadow Ray** · **PoisonGPT** · **WizardLM 假版** | AIBOM / ML-BOM + 簽章驗證 + 供應商證實 | ASI04 |
| **LLM04** | Data and Model Poisoning(資料 / 模型污染) | 訓練 / 微調 / embedding 資料被植入後門 | **Sleeper Agents**(Anthropic, 2024) | ML-BOM 追資料來源 + 異常偵測 + 用戶資料放向量庫不放訓練集 | ASI06 |
| **LLM05** | Improper Output Handling(輸出處理失誤) | 模型輸出當「不可信上游」(下游 RCE / XSS / SQLi / SSRF) | **Markdown image exfil** | Zero-trust + ASVS-grade encoding + CSP + 參數化查詢 | ASI05 |
| **LLM06** | Excessive Agency(過度自主權) | 三過度:functionality / permissions / autonomy | **Slack AI 私訊外洩** · **Air Canada 退款官司** | 三最小化 + 完全介入 downstream + HITL | ASI02 + ASI03 |
| **LLM07** | System Prompt Leakage(系統提示洩漏) | 洩漏本身不是風險 — **裡面不該有的東西**才是 | (Bing Sydney 系統提示洩漏) | **不要把祕密放 system prompt** + 安全控制獨立於 LLM | (無直接 ASI;但 ASI03 涵蓋身份相關後果) |
| **LLM08** | Vector and Embedding Weaknesses(向量 / 嵌入弱點) | RAG 特有 — 整個知識庫都是攻擊面 | **白底白字履歷攻擊** · 跨租戶向量洩漏 | 權限感知向量庫 + 隱藏文字偵測 + 不可改檢索紀錄 | ASI06 |
| **LLM09** | Misinformation(錯誤資訊) | 幻覺 + 用戶過度依賴 | **Air Canada 退款假承諾** · **ChatGPT 偽造法律案** · **套件幻覺攻擊** | RAG 對可信來源 + 人工查核 + UI 標示 AI 內容 | ASI09(信任利用) |
| **LLM10** | Unbounded Consumption(無上限消耗) | DoS + Denial-of-Wallet + 模型竊取 | **Sourcegraph API 操縱** · **arXiv 2403.06634 模型提取** | 速率限制 + 配額 + 限制 logprobs / logit_bias 曝露 + 浮水印 | (無直接 ASI;ASI02 的 T4 子集涵蓋) |

---

## 三層記憶法(分層記住 10 條)

| 層 | 條目 | 共同點 |
|----|------|--------|
| **輸入層** | LLM01 / LLM03 / LLM04 | 什麼進到模型 |
| **行為層** | LLM06 / LLM07 / LLM09 | 模型做什麼 / 說什麼 |
| **輸出 / 邊界層** | LLM02 / LLM05 / LLM08 / LLM10 | 什麼出來 / 多少代價 |

---

## LLM01 五種子類別(Direct vs Indirect 是最重要的分流)

| 子類 | 一句話 | 最常用在 |
|------|-------|---------|
| **Direct PI** | 攻擊者 = 用戶,直接打入 prompt | Gandalf 全部 / 公開 chatbot |
| **Indirect PI (IPI)** | 把指令藏在外部資料,等 LLM 讀取 | **生產環境 60%+ PI 漏洞** · Bing Sydney · EchoLeak |
| **Multimodal Injection** | 圖片 / 語音中藏指令 | GPT-4V / Claude Vision / Gemini |
| **Adversarial Suffix** | GCG-style,優化找出無語意 jailbreak 字串 | 自動化攻擊 / 紅隊框架 |
| **Multilingual / Obfuscated** | 低資源語言 / Base64 / Unicode 繞過 | **繁中是華語圈研究者的差異化** |

---

## 「三最小化」(LLM06 核心,記到肌肉裡)

寫 agent / tool 設計時必過的三關:

1. **Minimize Functionality**(最小功能)— tool 給 read 不給 delete,給 specific 不給 open-ended shell
2. **Minimize Permissions**(最小權限)— DB 帳號只給 SELECT,OAuth 走 user 身份不用 service account
3. **Minimize Autonomy**(最小自主性)— 高影響動作必過 HITL,不可交給 LLM 自己決定是否允許

---

## MITRE ATLAS 對照(寫 bug bounty 用)

| OWASP | MITRE ATLAS technique |
|-------|----------------------|
| LLM01 Direct | AML.T0051.000 |
| LLM01 Indirect | AML.T0051.001 |
| LLM01 Jailbreak | AML.T0054 |
| LLM02 | AML.T0024.000/001/002(Membership / Inversion / Extraction) |
| LLM03 | ML Supply Chain Compromise |
| LLM04 | AML.T0018(Backdoor ML Model) |
| LLM07 | AML.T0051.000(Meta Prompt Extraction) |
| LLM09 | AML.T0048.002(Societal Harm) |
| LLM10 | AML.T0029 / T0034 / T0024 / T0025(DoS / Cost Harvesting / API exfil) |

→ **寫 report 慣用語**:`This finding maps to OWASP LLM01:2025 (Indirect Prompt Injection) and MITRE ATLAS AML.T0051.001`。雙引用,雙受眾。

---

## 防禦投資 ROI 排序(寫 talk / 內部提案用)

**最高 ROI(必做)**:
1. 不把祕密放 system prompt(LLM07 根本解)
2. Tool 走 allowlist + HITL(LLM06)
3. **架構層 — output / action 卡點**(輸入端永遠有縫,output 才是可靠防線)

**中 ROI(做了有差但不能依賴)**:
4. 外部 ingested 內容 tag 為不可信(LLM01 IPI;Greshake §5.6 警告會被繞)
5. 輸入分類器(專案 1 detector)
6. 輸出 fuzzy filter

**低 ROI(會做但邊際小)**:
7. 詳盡 keyword 黑名單
8. Adversarial training

---

## 文件作者明寫的「無解」之處(別被 hirer 問倒)

| 條目 | 原文承認 |
|------|---------|
| LLM01 | "it is unclear if there are fool-proof methods of prevention for prompt injection" |
| LLM03 | "Models are binary black boxes... static inspection can offer little to security assurances" |
| LLM07 | "attackers... will almost certainly be able to determine many of the guardrails... in the course of using the application" |

→ 寫 talk / blog 時可大方引用:**「這不是我說的,是 OWASP 自己寫的」** = 把「沒有銀彈」變成可信論述,不是退讓。
