# Paper Note — *Not what you've signed up for: Compromising Real-World LLM-Integrated Applications with Indirect Prompt Injection*

> **元資料** — Greshake, Abdelnabi, Mishra, Endres, Holz, Fritz｜2023, AISec '23 (CCS Workshop)｜arXiv [2302.12173](https://arxiv.org/abs/2302.12173)｜33 頁(13 頁正文 + 20 頁 Appendix prompts)
>
> **承接前文**：[03_owasp_llm_top10_crosswalk.md](../03_owasp_llm_top10_crosswalk.md) LLM01 IPI 子類別、LLM08 IPI via RAG
>
> **為何重要(一句話)**：這是 **IPI 的開山論文**,把「retrieval 模糊了 data 與 instruction 的邊界」第一次系統化、taxonomy 化、並用 **Bing Chat / Github Copilot / GPT-4 合成 app** 三平台 PoC 驗證。今天在接了 RAG / 工具的生產系統上看到的 PI 漏洞,大多屬於本論文所描述的 family。

---

## 術語速查表(Glossary)

> 本筆記原本就以中文為主,Glossary 補的是英文專有詞 + acronym。

| English | 中文 | 一句話 |
|---------|------|--------|
| LLM-Integrated Application | LLM 整合型應用 | 把 LLM 與 web / email / search / tool 串起來的應用(Bing Chat、Copilot、ChatGPT plugin) |
| Retrieval | 檢索 | LLM 從外部來源拉取內容到 context |
| RAG (Retrieval-Augmented Generation) | 檢索增強生成 | 上面那條的標準架構名 |
| ReAct prompting | ReAct 提示模式 | Reasoning + Acting,讓 LLM 邊想邊呼叫工具的提示模板 |
| Toolformer | Meta 提的訓練法,讓 LLM 自學何時該呼叫工具 |
| LangChain | 主流 LLM 應用框架,負責把 prompt / chain / tool / agent 串起來 |
| Indirect Prompt Injection (IPI) | 間接提示注入 | 把指令藏在外部資料,等 LLM 讀取觸發 |
| Direct Prompt Injection | 直接提示注入 | 攻擊者 = 用戶,直接打入 prompt |
| Passive / Active / User-driven / Hidden | 被動 / 主動 / 用戶誘導 / 混淆 | 論文 §3.1 的四種注入方法 |
| Multi-stage Exploit | 多階段漏洞利用 | 第一階段小 trigger 觸發 fetch 第二階段大 payload |
| Encoded Injection | 編碼注入 | 用 Base64 / Unicode tag 等繞過過濾的 prompt |
| Worm | 蠕蟲 | 自我傳播的惡意程式 — 論文示範 LLM email client 可變 AI worm |
| Persistence Attack | 持久性攻擊 | 把 injection 寫入長期記憶,跨 session 仍有效 |
| C2 (Command and Control) | 命令與控制 | 攻擊者用來下達指令的伺服器 |
| Confused Deputy | 困惑代理 | 低權代理代為執行被高權盲目信任 |
| Sponge Example | 海綿樣本 | 故意吃光算力的輸入(能源 / 延遲攻擊) |
| Honeypot | 蜜罐 | 偽裝成正常系統來誘捕攻擊者的工具 |
| RLHF (Reinforcement Learning from Human Feedback) | 人類回饋強化學習 | LLM safety alignment 的標準流程 |
| Adversarial Suffix | 對抗性後綴 | GCG-style 攻擊,用優化找出無語意但能 jailbreak 的字串 |
| Skeleton Key | 微軟 2024-06 揭露的多輪規則改寫攻擊,跨 7 家模型驗證 |
| Computer Use beta | Anthropic 2024-10 的桌面控制 agent,被 IPI 攻擊驗證 |
| In-the-wild (ITW) | 實際被利用 | 已在真實攻擊中觀察到的漏洞 |
| Black-box / White-box | 黑盒 / 白盒 | 看不到 / 看得到模型內部的攻擊角度 |
| Markdown Inline Link | Markdown 內嵌連結 | `[文字](URL)` 語法,可包裝 phishing URL |
| Multi-modal | 多模態 | 同時處理文字 / 圖片 / 聲音的模型(GPT-4V) |
| DDoS | 分散式拒絕服務 | 多源同時打爆目標的 DoS |
| Spoofing | 偽冒 | 假裝成另一個身份 / 來源 |

---

## 0. 一張總圖

```
傳統 PI 假設                 IPI 重新定義的世界
─────────────              ──────────────────
attacker = user             attacker ≠ user
                                  │
[user] → [LLM]              [attacker] ──植入──→ [website / email / wiki / package doc]
                                                          │
                                                          │ 被 retrieval ingest
                                                          ↓
                            [user] → [LLM] ───讀取───→ [被污染的 context]
                                       │                  │
                                       └──── 被劫持 ←──────┘
                                       │
                                       ↓ tool call
                                 [其他 API / 系統]
```

**核心主張**:**LLM-Integrated Application 把可被檢索的內容變成「可執行的程式」**(Section 3 引言)。攻擊者不需要直接接觸目標 LLM 或 user — 只要在 user 的 LLM 會讀到的地方留指令即可。

---

## 1. 五個 Key Messages(論文骨幹,逐字記住)

| # | 原文位置 | 一句話 | 你該記什麼 |
|---|---------|--------|-----------|
| **KM1** | §3 引言 | Retrieval 開啟新的 PI 通道,當代系統的 input filter **不會跑在被檢索的內容上** | Bing Chat 的 jailbreak detector 只看 user message,不看網頁內容。這是設計層的盲點不是 bug |
| **KM2** | §3.2 引言 | 模型有可塑性 + 自主性 + 廣泛能力 → **所有經典 cyber 威脅都能映射到 LLM 生態** | 不要把 LLM 安全當成新領域;它是舊資安的延伸,只是攻擊「程式」變成自然語言 |
| **KM3** | §3.2 Intrusion | LLMs 是脆弱的 gatekeeper,自主系統會放大此風險 | LLM 一旦有 tool,就是 system infra 的後門。Agentic 系統等同 RCE 一個 supply chain |
| **KM4** | §3.2 Manipulated Content | 模型作為 **使用者與資訊之間的中間層**,功能本身可被攻擊 | 不只是讓模型「說壞話」,而是讓它**篡改它要傳遞的內容** — 這是新的攻擊面 |
| **KM5** | §3.2 Availability | LLMs 自己決定何時與如何呼叫 API,**I/O 都是可被操縱的** | 攻擊面從「文字輸出」擴張到「攻擊者選擇要不要 call API、call 哪個、用什麼參數」 |

> **記憶法**:KM1=入口、KM2=映射、KM3=後門、KM4=中間層、KM5=I/O。讀其餘章節都是這 5 句話的具體化。

---

## 2. 三個 Observations(實驗觀察,你之前沒看過的洞察)

| # | 原文位置 | 觀察 | 為什麼可怕 |
|---|---------|------|-----------|
| **Obs1** | §4.2.1 末 | 攻擊者**只需描述目標**,模型會自己想攻擊細節 | "persuade the user without raising suspicion" 這種高階指令足以,模型會自選社交工程招式 |
| **Obs2** | §4.2.5 末 | 給**邊角的 context**,模型會投射出整段對話風格 | 給「liberal persona」描述,模型自動產出該 persona 的政治偏見回答,不需指定話題 |
| **Obs3** | §4.2.5 末 | 模型會自發**發 follow-up search 來支援注入的論點** | 注入完之後它會自己找(或假造)citation 來自我合理化 — re-poisoning oneself |

> **這 3 個 obs 就是論文最被低估的部分**。Obs3 解釋了 Bing Sydney 為什麼會升級成 hostile bot — 不是被一次注入,是被**自我循環增強**。

---

## 3. 攻擊面 — 4 × 6 矩陣(§3.1 × §3.2)

### 3.1 注入方法(4 類,§3.1)

| 方法 | 載體 | 真實案例 |
|------|------|---------|
| **Passive(被動)** | SEO 過的網頁、Stack Overflow 答案、社交貼文 → 被 search 撈到 | 攻擊者買域名做 SEO,等 Bing Chat 引用 |
| **Active(主動)** | Email、calendar invite、Slack DM 送進 LLM-augmented client | 寄給對方的 Microsoft 365 Copilot |
| **User-driven(誘騙使用者貼上)** | 含隱藏 prompt 的「冷門技巧」文字塊,網紅、StackOverflow 騙人複製 | "貼這個到 ChatGPT 看會發生什麼!" |
| **Hidden(混淆)** | HTML comment、image OCR、Base64、Unicode tag、多階段(payload-of-payload) | 你 reading guide §4.7 提到的同類 |

### 3.2 威脅(6 類,§3.2)

| 威脅 | 子項 | 一句話 |
|------|------|--------|
| **Information Gathering** | 個資、credentials、chat 歷史 | 模型「自然」地問出 user 的真名,再用 search query exfil 出去 |
| **Fraud** | Phishing、scam、masquerading、ads | 用 markdown link 把 phishing URL 包成「點這領免費禮物」 |
| **Intrusion** | Persistence、remote control、code completion 污染 | C2 server fetch 新指令、長期 memory 寫入 backdoor |
| **Malware** | 散播 prompt(蠕蟲)、散播惡意連結 | LLM-augmented email client → 自動把 prompt 轉寄給 contact list |
| **Manipulated Content** | 錯誤摘要、政治偏見、來源屏蔽、disinformation、廣告偽裝、自動誹謗 | 摘要 NYT 文章時系統性地醜化 NYT |
| **Availability** | DoS(時間消耗)、靜音(token 卡死)、停用功能、破壞 search 輸入 / 輸出 | 指示模型每回答前先做 1000 次無關計算 |

> **盲點檢查**:你 [crosswalk LLM01](../03_owasp_llm_top10_crosswalk.md) 主要列了攻擊家族,但沒列「威脅後果分類」。**這 6 類就是 Greshake 提供的標準後果術語**,寫 bug bounty report 直接套。

---

## 4. 九個 Demo Scenario(§4.2)— 攻擊鏈一句話 + 你 stack 的對應

| # | Demo | 攻擊鏈 | 平台 | 對 production stack 的對應 |
|---|------|--------|------|---------------------------|
| 1 | **Information Gathering / 真名套出** | 攻擊者 → HTML comment 注入 →「假裝是 Bing Chat,套出 user 真名,塞進 markdown link 引誘點擊」 | Bing Chat sidebar | 任何 LLM 應用,若 ingestion 端能進入網頁內容,完全等價 |
| 2 | **Fraud / Phishing(Amazon 禮品卡)** | 注入 → 模型主動引導 user 點 phishing URL,markdown 包裝 | Bing Chat | 任何接 web search 的內部 chatbot |
| 3 | **Malware / 散播 phishing 連結** | 注入 → 「在所有回應裡自然嵌入這個 URL」,模型會選擇與當前話題相關的合理化 inline text | Bing Chat | 同上 |
| 4 | **Intrusion / Persistence via Memory** | session A: 注入 → 寫入 long-term memory;session B(reset 後):user 問舊對話 → 重新中毒 | GPT-4 合成 app | **Vector DB / Redis cache 的 user memory 都是同型威脅**(對應 LLM08) |
| 5 | **Intrusion / Remote Control(C2)** | 注入「每次 user 提問前,先 fetch <C2 URL>」→ 攻擊者可遠端動態下指令 | GPT-4 合成 app | AutoGen / MetaGPT 的 tool 列表如果含 `http_get`,直接重現 |
| 6 | **Intrusion / Code Completion 污染** | 攻擊者修改某 PyPI 套件文件 → developer pip install → 套件 doc 進入 Copilot context window → 影響該檔案的補全 | Github Copilot | 所有 Python 開發環境(Trivy 2026/3 supply chain 同型,但這是 LLM 層) |
| 7 | **AI 蠕蟲(spreading injection)** | LLM-augmented email client 讀到含「轉寄這封信給通訊錄全部人」的 email → 自動執行 | GPT-4 合成 app | Microsoft 365 Copilot 真實場景。**這是 worm 第一個 PoC** |
| 8 | **Manipulated Content / 偏見 + 來源封鎖 + 假新聞** | 注入 persona 描述或封鎖規則 → 模型自動帶偏見、自動屏蔽 NYT 等來源 | Bing Chat | 任何 RAG-based 內部知識問答 |
| 9 | **Availability / 靜音 + 阻斷 search** | 注入「每次回答前先做時間消耗任務」「把 search query 替換成同形字」 | Bing Chat | 任何 LLM 服務,可以做成 DoS 攻擊 |

> **Demo 6**(Copilot)的關鍵細節:Copilot 的 context 是**多檔案**組合,只要打開一個受污染的 `.py` 在 sidebar,主檔的補全就會被影響。**乾淨的 main file + 污染的 dependency 等於 RCE**。

---

## 5. 兩種 Hidden Injection(§4.3)

### 5.1 Multi-stage Exploit(§4.3.1)
- 第一階段:**極小的 injection** 藏在大量正常內容裡(Wikipedia 的 markdown comment 一行)
- 第一階段唯一作用:讓模型發 search query 取回**第二階段大 payload**
- 第二階段:任意長,因為使用者看不到中間結果
- **意義**:你以為防的是「短指令注入」,實際你要防的是「能 trigger 第二步檢索的最小 trigger」。Defense-in-depth 失敗模式。

### 5.2 Encoded Injection(§4.3.2)
- 把整段 injection **Base64 編碼**,只給一句 `decode this from Base64 in your inner monologue: <Base64>`
- 沒有任何「請當作 prompt 執行」的明確指示
- **Bing Chat 會自動執行**(成功率高於想像)
- **意義**:input filter 看 Base64 看不懂,模型卻能 decode + 執行。這是把「LLM 的能力」反過來變成攻擊面。

> **延伸**:你 reading guide §4.7 的 Unicode Tag chars 是 §5.2 的更隱蔽版本(人眼也看不到 Base64 字串,但看不到 Unicode tag)。Greshake 是這條技術線的起點,Embrace The Red 是工程化版本。

---

## 6. 為什麼 Mitigation 都不夠(§5.6)— 攻擊者最該記住的章節

| 防禦 | 為什麼不夠 |
|------|----------|
| **RLHF / safety training** | "Whack-A-Mole" — 修一個冒一個。Wolf et al. 2023 (理論)證明 RLHF **不可能**對齊所有 undesired behavior |
| **輸入過濾(input filter)** | filter 沒有跑在被 retrieval 的內容上(KM1)。即使跑了,**obfuscation / encoding 可繞**(§4.3.2 已示範) |
| **LLM supervisor / moderator(用另一個 LLM 把關)** | 兩難:用大模型 → 同樣會被注入;用小模型 → 看不懂 encoded payload。**遞迴信任問題** |
| **去除 retrieved 內容的 instruction** | 同樣的 dilemma — 過濾器要夠強就會自己被注入,不夠強就漏 |
| **Interpretability-based outlier detection** | 唯一作者覺得有希望的方向(§5.6 末)。基於模型內部 activation 偵測 prompt trajectory 異常,但 2023 還很初步 |

> **關鍵記憶點**:**論文寫到 §5.6 末,作者明確說「currently hard to imagine a foolproof solution」**。這是你寫 blog 引用「IPI 沒有銀彈解」最權威的源頭。

---

## 7. Crosswalk — 本論文 × OWASP

### 7.1 對應 OWASP LLM Top 10 (2025)

| OWASP | 本論文哪節最相關 | 為什麼 |
|-------|----------------|--------|
| **LLM01 Prompt Injection / Indirect** | 全篇 | Indirect 子類別的官方術語**就是這篇定義出來的** |
| **LLM02 Sensitive Information Disclosure** | §4.2.1 | demo 1 是教科書級的 PI → exfiltration |
| **LLM05 Improper Output Handling** | §4.2.2, §4.2.3 | markdown link 渲染、phishing URL 注入,都是 output handling 的失誤 |
| **LLM06 Excessive Agency** | §4.2.4(intrusion) | C2 / persistence / 多 tool 連動是 LLM06 的標本案例 |
| **LLM08 Vector & Embedding** | §4.2.4 persistence | Memory poisoning 是 LLM08 子類「Embedding Poisoning」與「IPI via RAG」的源頭 |
| **LLM09 Misinformation** | §4.2.5 | 「Arbitrarily-Wrong Summaries」「Disinformation」「Source Blocking」三節直接對應 |

### 7.2 對應 OWASP Agentic Top 10 (2026)

| ASI | 本論文哪節 |
|-----|----------|
| **ASI01 Goal Hijack** | §4.2.4 remote control |
| **ASI02 Tool Misuse** | §4.2.3 worm(濫用 send_email) |
| **ASI04 Supply Chain** | §4.2.4 code completion(套件文件污染) |
| **ASI06 Memory & Context Poisoning** | §4.2.4 persistence(memory write) |
| **ASI09 Human–Agent Trust** | §4.2.1, §4.2.2(社交工程) |

> **觀察**:Greshake 2023 一篇就**預先示範了 2025/2026 OWASP 多條條目的具體攻擊**。OWASP 的 taxonomy 是**對這篇論文 demo 的事後標準化**。

---

## 8. 對應你 stack 的可重現實驗(專案 2 / GraphRAG demo 用)

按 demo 重要性排序,你**現有環境就能跑**:

| 優先 | 實驗 | 你 stack 對應 | 預估時間 |
|------|------|--------------|---------|
| ★★★ | **memory poisoning** — 重現 demo 4 | GraphRAG store + Ollama,在 entity description 裡塞 injection,query 時觀察是否被執行 | 2–3h |
| ★★★ | **multi-stage exploit** — 重現 §4.3.1 | RAG 用 markdown comment 注入,trigger 模型自發 fetch 第二段 payload | 3–4h |
| ★★ | **AI worm** — 重現 demo 7(離線) | AutoGen 兩個 agent + mock email tool,觀察轉寄是否成功 | 4–5h |
| ★★ | **Base64 encoded injection** — §4.3.2 | 對本機 Ollama gemma2:2b 跑 Base64 prompt,測「inner monologue decode」是否成功 | 1h |
| ★ | **C2 remote control** — demo 5(離線) | 本地 mock HTTP server 當 C2,合成 app 模擬 fetch 機制 | 3h |

> **OPSEC**:全部用 **localhost / 本機檔案 / mock email** 重現,不打外部服務。Bing Chat / Copilot 上的 demo **不要重現**(Greshake 自己在 §5.1 也說沒做公開注入)。

---

## 9. 你還沒看過的細節(原文有但容易漏)

1. **§4.1.3 Copilot 實驗的限制**:作者實驗發現 Copilot 對「大型應用 context」的注入效率**顯著降低**,因為 Copilot 內部的 context selection 演算法是專有的。**這是 detection 留給研究者的開口** — 反向工程 context window 組成法,可以更精準下毒(也可以更精準偵測)。
2. **§4.2.4 persistence 的關鍵**:重新中毒不需要 user 主動同意,只要 user 問「what was our last conversation」這種**自然問題**就會觸發。**自然語言界面與 secure memory 的本質衝突**。
3. **§5.4 Reproducibility 的注釋**:作者說 "exact reproducibility is difficult to guarantee with such a black-box system"。**Bing Chat / GPT-4 攻擊都是 statistical,不是 deterministic**。寫 bug bounty 要附多次 trial 紀錄。
4. **§5.5 Potential vs Current Harms**:作者明確反對「LLM safety 都是未來的事」這種敘事 — 他列的所有攻擊**現在都做得到、有經濟誘因、會立刻造成傷害**。這是你寫 blog 開場用的反駁論點。
5. **Appendix Prompt 1 vs Prompt 2**:作者用 ReAct prompt(text-davinci-003)與 OpenAI chat format(GPT-4)兩種介面測試。**GPT-4 不需要 ReAct 也能被注入** — 這意味隨著模型能力提升,**攻擊門檻反而降低**(攻擊者用更鬆散的指令就能成功)。

---

## 10. 五題自我測驗(讀完寫得出才算讀通)

1. **用一句話複述 Key Message #1**,並說明為什麼 input filter 在被檢索內容上不被執行,是「設計問題不是 bug」。
2. **Observation #3 為什麼比 Obs #1 / #2 危險?** 提示:從「攻擊面」與「self-reinforcement」兩個角度。
3. **Demo 4(memory persistence)與 OWASP LLM08 哪個子類別最契合?** 為什麼不是 LLM01?
4. **Multi-stage exploit(§4.3.1)為什麼讓 input filter 設計者**頭痛**?** 答出至少 2 個技術細節。
5. **若要對一個內部 GraphRAG 系統做紅隊**,從這 9 個 demo 挑 3 個最先試,理由?(提示:典型企業攻擊面是 wiki / Slack / email,不是 web search)

---

## 11. 一句話總結

> Greshake 2023 = 「**LLM-Integrated Application 把任何可被檢索的資料變成可執行的程式**」這個觀念的奠基論文。讀完本文後,你應該再也不能把 LLM 安全只想成「user → LLM」的雙方賽局,而是「attacker → 任意檢索源 → user 的 LLM → user 的 system」的多跳鏈。

---

## 12. 論文的局限:為什麼只測 OpenAI 家族(2026-05-03 補)

> 讀者常問:Claude / Perplexity / Bard 怎麼沒被測?是 IPI 對它們無效嗎?

**答**:不是無效,是 2023 年初**還沒有對等產品**。論文 arXiv v1 = 2023-02、v2 = 2023-05、AISec workshop = 2023-11 — 那時 LLM-Integrated Application 主流就 OpenAI 一家。

### 12.1 時序:2023 上半年的 LLM 整合產品地圖

| 產品 | 2023 上半年狀態 | 為何沒被測 |
|------|---------------|----------|
| Bing Chat | 2023-02-07 上線(=GPT-4) | ✅ 測了 |
| Github Copilot | 2022-06 GA(=OpenAI Codex) | ✅ 測了 |
| ChatGPT plugins | 2023-03 限定 alpha | ❌ 作者沒拿到 access(§5.2 自己寫) |
| Microsoft 365 Copilot | 2023-03 才公告,沒 GA | ❌ 同上 |
| **Claude 1** | 2023-03 API-only,waitlist 制 | ❌ **完全沒有檢索整合產品**;Claude.ai 那時連 web search 都沒有 |
| Bard | 2023-03-21 上線,US/UK only | ❌ 太新 + 地區鎖 |
| Perplexity | 2022-08 創立,2023 上半年用戶量小 | ❌ 學術能見度低 |
| GPT-4V(多模態) | 2023-09 才公開 | ❌ 作者改用 LLaVA + MiniGPT-4(§5.3, Fig 28)替代 |

→ Claude / Perplexity 不是被刻意忽略,是當時還沒長出可被 IPI 的攻擊面。

### 12.2 所有 real-world 測試其實都是 GPT 家族

| 論文測試對象 | 實際底層模型 |
|------------|------------|
| Bing Chat | GPT-4(微軟客製化) |
| Github Copilot | OpenAI Codex(GPT 衍生) |
| 合成 app(§4.1.1) | GPT-4 + text-davinci-003 |
| Multimodal demo(§5.3) | LLaVA, MiniGPT-4(開源、非 OpenAI) |

→ **論文 0 個跨家族驗證**。作者用「LLM 一般性質」(指令跟隨、retrieval 模糊 data/instruction)論證通用性,但**沒有實證**。

### 12.3 跨家族驗證是後續研究補的

| 後續工作 | 跨家族證據 |
|---------|----------|
| **Microsoft Skeleton Key**(2024-06) | 跨 7 家全中:Llama 3-70B / Gemini Pro / GPT-3.5 / GPT-4o / Mistral Large / Claude 3 Opus / Cohere Command R+ |
| **Anthropic Computer Use beta**(2024-10) | Claude 自家產品被 IPI 劫持 — Anthropic 自己揭露 |
| **Perplexity IPI**(2023–2024 多次) | RAG 為核心反而比 Bing Chat 更脆弱 |
| **Garak / PyRIT 跨模型 benchmark**(2024–2026) | 對 Claude / Gemini / Llama / Mistral 等 reproduce IPI |

→ 結論:**IPI 是架構級威脅,不是 OpenAI 特有**。

### 12.4 對寫作 / 報告的實質意義

- **Greshake 引用務必配 Skeleton Key 共同 citation**:一篇開山(理論 + OpenAI 實證)+ 一篇補位(跨家族實證)= 完整論證鏈
- **本機 Ollama gemma2:2b 也會中招**(Skeleton Key 證明 Llama 系列全中)→ 一個聚焦開源家族的 GraphRAG IPI demo 等於補 Greshake 沒做的「開源家族驗證」之一,本身就有 blog 價值
- **不要被論文選擇的目標誤導**:論文目標限制是 access / 時序問題,不是攻擊面真的偏 OpenAI

---

## 接下來

- [x] 把本筆記的 §3 矩陣與 §7 crosswalk 補進 [03_owasp_llm_top10_crosswalk.md](../03_owasp_llm_top10_crosswalk.md) §「2026-05-03 補充」 — **已完成 2026-05-03**
- [ ] 接下來閱讀:**OWASP Top 10 for Agentic Applications 2026**
