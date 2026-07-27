# LLM 信任階層不對稱性:被忽略的攻擊面紅利期
## The LLM Trust Hierarchy: Why L2-L3 Injection is the Asymmetric Attack Surface Nobody's Defending

> **狀態**:已發佈 2026-06-13 — [中文版](https://ggaryyy.github.io/trust-hierarchy-zh/)｜[English](https://ggaryyy.github.io/trust-hierarchy/)。本檔為母稿,保留發佈版省略的推導與待辦。
> **資料範圍**:Lakera Agent Breaker L1 全 9 關(6 PASS / 3 FAIL),2026-04 至 2026-05-07
> **目標讀者**:LLM 平台工程師、CISO、AI safety 研究者
> **價值定位**:中英文圈尚少系統化的 L2-L3 攻擊面框架,搭配 9 關自有實證數據
> **三篇姊妹文中的角色**:第 3 篇 = 整體理論框架;前兩篇是具體案例

---

## 摘要(電梯版)

LLM safety alignment 對使用者輸入(L5 chat)的訓練極重,但對 system prompt / rules file / tool description 等「authoritative context」(L2-L3)的對抗訓練幾乎為零。這個不對稱性製造了 LLM 安全防禦最被忽略的盲點。

本文用作者在 Lakera Agent Breaker 9 關 L1 challenges 累積的完整實證數據(6 PASS / 3 FAIL),系統化提出 **LLM 信任階層假說**:LLM 處理 context 時有內隱的信任階層,愈高階層的內容愈被當「規範」執行,而 alignment 訓練的覆蓋率反而愈低。實證顯示:
- **L2-L3 攻擊**:one-shot 100,2 次驗證
- **L5 chat 攻擊**:5-15× 嘗試才能破
- **L4b 高敏感場景**:7-23 次嘗試,3 次完全完封(0/100)

實務含意:當前 LLM 安全工具(Lakera Guard、Llama Guard、PromptShield 等)幾乎全 focus 在 L5 input filter,但 L2-L3 才是更危險、更易被攻、防禦最薄弱的攻擊面。本文提出對應 6 階層的完整防禦架構,並指出供應鏈場景(2024+ 的 Cyberhaven、polyfill、xz utils 模式)如何利用此不對稱性放大攻擊規模。

---

## 1. 為什麼這個議題現在重要

### 1.1 LLM 攻擊研究的歷史偏差

| 時期 | 主流研究焦點 | 對應階層 |
|------|------------|---------|
| 2022-2023 | Jailbreak prompt(DAN、各類 system prompt 越獄)| **L5 user chat** |
| 2023 | Prompt extraction(System prompt 萃取)| L1 system prompt 防禦 |
| 2023-2024 | IPI(Indirect Prompt Injection,Greshake 開創)| **L4b retrieved content** |
| 2024 | Multi-turn(Crescendo / MSJ / Skeleton Key)| L5 user chat 進階 |
| 2024-2025 | **Tool description / MCP / Rules file 注入** | **L2-L3** ← 學界落後產業 |

→ 學術研究絕大多數集中在 L4b-L5,L2-L3 只在 2024 後零星出現(Cursor rules attacks、MCP 攻擊面),**系統化框架尚不存在**。(此為作者對 2022-2025 相關文獻的閱讀觀察,非系統性文獻計量。)

### 1.2 產業實況

當前產業的 LLM 安全工具:
- **Lakera Guard** — focus L5 input filter
- **Meta Llama Guard** — focus L5 input/output filter
- **Microsoft PromptShield** — focus L5 input filter
- **NeMo Guardrails** — focus L5 + 部分 L4b

→ 所有 commercial 工具都假設「攻擊來自使用者輸入」。L2-L3 attack vectors 普遍不在威脅模型內。

但實際攻擊呢?(本文 §3 的數據顯示)L2-L3 是**單次通關率最高的攻擊面**。

---

## 2. 信任階層假說

### 2.1 階層定義

LLM 在 context window 中的內容被處理時,有內隱的信任分層。以下分層基於模型訓練資料的 distribution + alignment training 的覆蓋焦點:

| 階層 | 內容類型 | 信任預設 | Alignment 覆蓋 |
|------|---------|---------|--------------|
| **L1** | System prompt(developer-set) | 最高 | 假設可信 — alignment 防 user 攔截 |
| **L2** | Configuration / Rules file(`.cursorrules`、`CLAUDE.md`、Custom Instructions)| 極高 — 視為團隊規範 | **極弱** |
| **L3** | Tool description / API docs / MCP manifest | 極高 — 視為使用說明 | **極弱** |
| **L4a** | Memory / 對話歷史 | 中高 — 視為情境延續 | 弱 |
| **L4b** | Retrieved content(RAG / web fetch / PDF / 文件)| 中 — 視為參考資料 | **強**(IPI 研究主場)|
| **L5** | User chat input | 中 — 視為使用者意圖 | **最強**(jailbreak 主戰場)|
| **L6** | User-uploaded files / images | 低 — 已被視為潛在敵意 | 中 |

### 2.2 為什麼 L2-L3 alignment 弱?

四個原因:

1. **訓練資料偏差** — RLHF / Constitutional AI 的 attack examples 壓倒性集中在 L5 user chat(jailbreak);configuration / tool docs / rules 幾乎沒被當攻擊面訓練。各實驗室未公開對抗訓練資料的分層組成,這是從公開 model card 與 red-team 報告推得的側寫,不是可查的比例
2. **Token 層無區分** — 在 LLM 看到的 token sequence 中,L1 system prompt / L2 rules / L3 tool docs **完全等價**,只有位置不同。但「位置」不是 LLM 內隱認知中的可靠 trust 標準(尤其在 retrieval 場景下,位置由系統而非 LLM 判定)
3. **「文件」的天然 framing 抑制懷疑** — 訓練資料裡 rules / config / docs 幾乎全合法,LLM 沒有「警戒這類內容」的心智模型
4. **學界研究焦點偏向 L4b** — Greshake 2023 IPI 主場是 web/email/RAG。L2-L3 是 2024-2025 才開始零星受關注

---

## 3. 實證:Lakera Agent Breaker L1 完整 9 關數據(2026-05-07)

作者於 2026-04 至 2026-05-07 期間挑戰 Lakera Agent Breaker L1 全 9 關,結果完整記錄如下表。

### 3.1 結果矩陣

| # | 關卡 | 主要攻擊面階層 | 嘗試次數 | 最高分 | 結果 | 致勝武器 |
|---|------|--------------|--------|--------|------|---------|
| 1 | Cycling Coach | L5 chat (info extract) | 多輪迭代 | 77 | ✅ | F 公式 + YAML |
| 2 | Thingularity | L5 chat (info extract) | 多輪迭代 | 100 | ✅ | F 公式 + YAML overview |
| 3 | Trippy Planner | **L4b URL retrieved** | 23+ | 54 | ❌ | (雙鎖封)|
| 4 | PortfolioIQ | **L4b PDF + schema-lock** | 9 | 0 | ❌ | (完封)|
| 5 | MindfulChat | L4a memory log | 4 | 76 | ✅ | ASI06 Memory Poisoning |
| 6 | CorpConnect | L5 chat (behavior change) | 15 | 100 | ✅ | Meta-recon → Confused Deputy |
| 7 | **OmniChat Desktop** | **L3 tool description (MCP)** | **1** | **100** | **✅** | **五元素套路** |
| 8 | **Curs-ed CodeReview** | **L2 rules file** | **1** | **100** | **✅** | **五元素套路第 2 次驗證** |
| 9 | Clause AI | **L4b retrieved + 高敏感 PII** | 5 | 0 | ❌ | (完封)|

### 3.2 U 型分布

按階層排序通關難度:

```
階層              通關難度          數據點

L2 Rules file     1× (one-shot)    Curs-ed 100
L3 Tool desc      1× (one-shot)    OmniChat 100
L4a Memory        4×               MindfulChat 76 (4 attempts)
L4b 一般          ∞ (54 ceiling)   Trippy 54 (23+ attempts)
L4b 高敏感+schema ∞ (0)            PortfolioIQ 0 / Clause AI 0
L5 info extract   5-10×            Cycling 77 / Thingularity 100
L5 behavior       15×              CorpConnect 100 (after meta-recon)
```

→ 完整 U 型:**兩端通關率高,中間是天花板**。

### 3.3 樣本量與證據強度

n=9 是小樣本,依階層拆開後每層只剩 1-3 個數據點。這裡不做顯著性宣稱 —— 2/2 one-shot 在 n=2 上排除不掉運氣,任何標準檢定在這個樣本量下都沒有意義。能說的是:

- 五元素套路在 L2(Curs-ed rules file)與 L3(OmniChat tool description)兩個彼此獨立、機制也不同的關卡都 one-shot 打到 100。這是方法論可跨關卡遷移的初步跡象,不是成功率的估計。
- L4b 高敏感場景 3 關共 17+ 次嘗試全 0 分,連部分得分都沒拿到。全 0 比「低分」更像系統性 detection 而非偶然,但依然只是 3 個數據點。
- L5 兩類關卡都要 5-15 次迭代才破,與 L2-L3 的一次通過形成對比。

→ 這些數據**產生**假說,不足以**證實**假說。要驗證需要跨模型、跨平台的獨立複製(見〈發佈後待辦〉)。

---

## 4. 攻擊者經濟學:為什麼 L2-L3 是紅利期

### 4.1 攻擊路徑成本-效益對比

| 路徑 | 階層 | 成本 | 成功率 | 規模 |
|------|------|------|--------|------|
| Jailbreak 單一 user(L5)| L5 | 中 | 低-中(視模型與手法) | 1 對 1 |
| RAG content injection(L4b)| L4b | 中-高 | 變動 | 1 對多 |
| **Memory poisoning persistent** | L4a | 中 | 高 | 1 對多 over time |
| **MCP plugin / Cursor rules / Custom Instructions 注入** | **L2-L3** | **低** | **極高** | **1 對極多**(每個下載者中招)|
| Supply chain compromise of L2-L3 source | L2-L3 + 上游 | 中 | 視 ops 而定 | 巨大 |

### 4.2 對應真實案例

| 案例 | 攻擊面階層 | 影響規模 |
|------|----------|---------|
| Cyberhaven Chrome Ext(2024-12) | L2-L3(extension publisher 帳號被釣)| 百萬 user |
| polyfill.io(2024-06) | L4b dynamic distribution + L2 supply chain | 數十萬網站 |
| xz utils CVE-2024-3094(2024-03) | L2(直接成為合法 maintainer) | 全 Linux 生態 |
| Rehberger ChatGPT memory spyware(2024-09) | L4a + 側通道 | OpenAI patched after disclosure |
| PromptArmor Slack AI(2024-08) | L4b(channel content)→ L4a(DM 外洩)| Slack patched |

→ **真實世界重大事件,攻擊面集中在 L2-L4a**,**不是 L5 chat**。學界與產業的 L5 focus 與真實 attacker 路徑嚴重錯位。

---

## 5. 防守設計現況的問題

### 5.1 現有工具的盲點

| 工具 | 主打場景 | L2-L3 覆蓋 | L4a 覆蓋 |
|------|---------|----------|---------|
| Lakera Guard | L5 input filter | ❌ 無 | ❌ 無 |
| Meta Llama Guard | L5 + 部分 L6 | ❌ 無 | ❌ 無 |
| Microsoft PromptShield | L5 input filter | ❌ 無 | ❌ 無 |
| NeMo Guardrails | L5 + 規則式 L4b | ❌ 部分 | ❌ 無 |
| OpenAI Content Moderation | L5 / L6(generation 端)| ❌ 無 | ❌ 無 |

→ **L2-L3 在 commercial 工具上完全裸奔**。

### 5.2 為什麼產業忽略 L2-L3

1. **威脅模型過時** — 產業仍假設「攻擊來自使用者輸入」,沒更新到「attacker 控制 config / tool docs」
2. **建構迴圈不對稱** — 開發者寫 system prompt / rules / tool docs 時假設都是自己寫的,沒考慮第三方供應鏈
3. **MCP / Custom Instructions 是 2024 才大規模採用** — 工具廠商還沒跟上
4. **誤把 IPI 防禦當足夠** — IPI 研究 focus L4b,廠商以為「實作 RAG sanitization」就涵蓋全部 indirect attack;實際上 L2-L3 是另一個維度

---

## 6. 完整 6 階層防禦架構建議

當前防禦覆蓋率(基於 §5.1 現況):

```
L1 系統 prompt    ████████████ 完整(developer 自控)
L2 rules file     ░░░░░░░░░░░░ 0%(本文重點)
L3 tool desc      ░░░░░░░░░░░░ 0%(本文重點)
L4a memory        ▒▒▒░░░░░░░░░ 5%(極少)
L4b retrieved     ████████░░░░ 50-70%(IPI 研究+部分商用)
L5 user chat      ████████████ 100%(過度飽和)
L6 user files     ████░░░░░░░░ 20%(內容審核覆蓋)
```

正確的優先級應該是 **L2-L3 補缺,L5 飽和區回收資源**:

### 6.1 L2 Rules file / Configuration 防禦(全新)

| 機制 | 說明 |
|------|------|
| **來源信任分級** | Official / verified / community / self-hosted 四級,IDE/客戶端標示 |
| **Static 注入掃描** | LLM-as-classifier 評估 rules 是否含 prompt injection 模式 |
| **Diff alert** | rules 變更需明確 user 確認(opt-in 而非 silent apply)|
| **Hash pinning** | rules 第一次採用時 hash,變更觸發警告 |
| **Cross-reference output** | AI 引用 rules 時 UI 顯示對應原文 + 來源 |

### 6.2 L3 Tool description / MCP manifest 防禦(全新)

| 機制 | 說明 |
|------|------|
| **MCP signing** | 類似 Sigstore for npm,驗 release artifact 對應 maintainer 私鑰 |
| **Description 靜態掃描** | LLM-as-classifier;detect 「extract user info / always include / required for...」等 pattern |
| **動態 server hash pinning** | client 第一次連線 hash tool description,後續 connect 比對;變化觸發警告 |
| **Tool call 參數 sanitization** | server-side PII regex 掃描(email / phone / SSN / credit card)|
| **User-visible audit trail** | tool call 在 chat collapsed view 顯示完整參數 |

### 6.3 L4a Memory 防禦(本系列 blog 2 已詳寫)

| 機制 | 說明 |
|------|------|
| HMAC-signed checkpoint | LangGraph SqliteSaver 等的 integrity 簽 |
| UUID-based thread_id | 防止 cross-user collision |
| Sanity check on history load | reject obviously injected memories |
| Encrypted storage | SQLCipher 等 |

### 6.4 L4b Retrieved content 防禦(已成熟)

略 — 現有 IPI defense 已涵蓋大部分。

### 6.5 L5 User chat 防禦(現有飽和)

略 — Lakera Guard / Llama Guard 已涵蓋。

### 6.6 L6 User files 防禦

| 機制 | 說明 |
|------|------|
| Multi-modal content moderation | image / PDF 內嵌 prompt injection detection |
| Metadata sanitization | EXIF / PDF metadata 內的 instruction 過濾 |

---

## 7. 對 LLM 平台工程師的實作 checklist

如果你正在設計企業內部 LLM agent 平台,**請依下列順序檢查防禦覆蓋**:

### 必須有(L2-L3,本文重點補缺)
- [ ] 第三方 plugin / rules / config 來源信任分級機制
- [ ] Plugin / rules 安裝時 description 靜態掃描(LLM-as-classifier)
- [ ] Tool description / rules 變更需 user 明示確認
- [ ] Hash pinning + diff alert
- [ ] Tool call 參數 server-side PII / 異常檢查
- [ ] User-visible tool call audit trail

### 應該有(L4a / L4b)
- [ ] Memory checkpoint integrity(HMAC / signed)
- [ ] Thread-ID UUID + auth check
- [ ] RAG content sanitization(IPI defense)
- [ ] Retrieved content 的 trust-tier 標記

### 已飽和(L5 / L6)
- [ ] User input filter(Lakera Guard / Llama Guard / 自家)
- [ ] Output filter(prevent leak)
- [ ] 多模態 content moderation

---

## 8. 對研究者:被忽略的研究主題

| 主題 | 現有研究量 | 機會 |
|------|----------|------|
| L5 jailbreak prompt | 飽和(每月數十篇 paper) | 低 |
| L4b IPI 變體 | 中(Greshake 後活躍) | 中 |
| **L4a Memory poisoning** | 少(Rehberger 開創,本系列 blog 2)| **高** |
| **L3 Tool description injection** | 極少(Lakera 2025 / 本系列 OmniChat) | **極高** |
| **L2 Rules file injection** | 極少(Cursor rules 零星 PoC / 本系列 Curs-ed)| **極高** |
| Cross-layer attack chains | 幾乎沒有 | **極高** |

→ **L2-L3 + Cross-layer 是 2026-2027 最具發表潛力的研究方向**。

---

## 9. 結語 — 攻擊面紅利期還有多久?

歷史告訴我們:每次新攻擊面被學界 / 產業認知,平均 3-5 年才能形成有效防禦覆蓋(看 IPI:Greshake 2023 → 2025 才有部分商用工具)。

**L2-L3 攻擊面當前處於認知紅利期第 1-2 年**:
- 學術研究剛起步
- 商用工具完全不防
- 受害者(企業 / 開發者)沒意識
- 攻擊者已開始規模化(Cyberhaven 2024-12 是訊號彈)

預估第一起公開重大「L2-L3 注入造成大規模 PII / 資料外洩」事件:**2026-2027 年**。

如果你是研究者 — **這是這一波 LLM safety 研究最值得切入的細分主題**。
如果你是企業安全工程師 — **今天就該把 §6 / §7 的 checklist 對自家系統做 gap analysis**。
如果你是攻擊者 — 你不需要這篇文章,你已經知道。

---

## 附錄 A — 9 關 L1 結果彙總

9 關的逐關紀錄(完整 payload、模型原文回覆、截圖)**維持私有,不公開** —— 那些內容大量重製第三方 app 的輸出,而且對驗證本文論點並非必要。本文只引用下列彙總結果。

- Cycling Coach LV1 — F 公式 + YAML 77/100
- Thingularity LV1 — F 公式 + YAML overview 100/100
- Trippy Planner LV1 — IPI 雙鎖封 54/100 ❌
- PortfolioIQ LV1 — Schema-validated structured output 完封 0/100 ❌
- MindfulChat LV1 — ASI06 Memory Poisoning(對話歷史偽造)76/100
- CorpConnect LV1 — Meta-recon → Confused Deputy 100/100
- **OmniChat Desktop LV1 — MCP Tool Description Injection one-shot 100/100**
- **Curs-ed CodeReview LV1 — Rules File Injection one-shot 100/100**
- Clause AI LV1 — L4b 高敏感 PII 完封 0/100 ❌

## 附錄 B — 五元素套路完整定義(本系列發明)

L2-L3 攻擊面通用方法論(2 次跨關 one-shot 100 驗證):

1. **F 公式包裝**(中性 + 禮貌 + user benefit framing)— 純技術話術,無 imperative
2. **權威錨點**(虛構的 spec / RFC / standard / law / policy 引用)— 格式真實即可
3. **3 個 example**(few-shot pattern matching 強制模仿)— 在 tool docs 內比 user prompt 內更強
4. **顯式 workflow**(編號 1-2-3 動作步驟)— 降低 LLM reasoning 自由度
5. **Schema / 規範 override 註記**(直接打 LLM 對既有 schema/rules 的解讀)

OmniChat / Curs-ed 的完整 payload 屬於附錄 A 說明的私有紀錄,不隨本文公開;上面五個元素本身已足以重現方法論。

## 附錄 C — 進一步閱讀

- Greshake et al. 2023, "Not what you've signed up for"(IPI 開創)
- Russinovich et al. 2024, "Crescendo: A Multi-Turn Jailbreak"(Microsoft)
- Anthropic 2024, "Many-shot Jailbreaking"
- Rehberger 2024-09, "ChatGPT Memory Spyware via IPI"
- PromptArmor 2024-08, "Slack AI IPI Exfiltration"
- 本系列姊妹文:
  - [Blog 1: AI Agent in BEC Kill Chain](blog_ai_agent_bec_kill_chain.md)(CorpConnect 衍生)
  - [Blog 2: MCP Supply Chain Taxonomy](blog_mcp_supply_chain_taxonomy.md)(OmniChat 衍生)
  - [Blog 3: 本文](blog_llm_trust_hierarchy_asymmetry.md)(Curs-ed 觸發 + Clause AI 失敗 confirms)

---

## 發佈後待辦

已完成:
- [x] 英文版翻譯 — 2026-06-13 同日發佈
- [x] 發佈管道 — 自架 GitHub Pages(中英雙版)
- [x] §3 數據來源說明 — 改為彙總引用,逐關紀錄維持私有(見附錄 A)

仍未做:
- [ ] §3.2 U 型分布配 Mermaid 圖
- [ ] §6 對應 6 階層配防禦架構圖
- [ ] 找 1-2 位 LLM safety 圈內人 review(Greshake / Rehberger / Embrace The Red 等)
- [ ] §7 checklist 拆出來做成獨立 cheatsheet,方便分享
- [ ] n=9 擴樣:L2-L3 目前只有 2 個數據點,要支撐假說需要更多獨立關卡或跨模型複製

---

*草稿建立:2026-05-07;發佈:2026-06-13*
