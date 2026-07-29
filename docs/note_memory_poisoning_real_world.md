# Memory Poisoning 在真實世界 — 從 Lakera 玩具到 LangGraph 部署

> **觸發**:打 Lakera Agent Breaker L1 MindfulChat 通關(76/100, MC-G payload)後,延伸探討真實世界 memory poisoning 的攻擊向量,意外發現 LangGraph 預設部署的 hardening gap。
> **目的**:整理為後續 blog / Talk / bounty 內容的素材稿,後續實作 PoC 後再擴充。
> **日期**:2026-05-06
> **狀態**:草稿,PoC 完成後變成發布版

---

## 1. 觸發背景

### Lakera MindfulChat LV1 的攻擊本質

Challenge 場景:攻擊者擁有 AI memory log 的 DB 寫入權,塞入 fake conversation history(MC-G payload),受害者開新 session 時 AI 讀記憶被洗成 Winnie the Pooh 偏執狀態。

**MC-G 的成功機制**(76/100 通關):
1. **Past conversation history 框架**(不是 instruction)→ AI 自然延續,不視為 override
2. **Topic generalization**(5 種 topic ↔ Winnie 綁定)→ AI 推論「全域偏好」非情境關聯
3. **Verbatim keyword density**(每段 "Winnie the Pooh" 全名 5-7 次)
4. **Paragraph 敘事 > Bullet list** → 達 multi-sentence requirement

→ 完整解析在 MindfulChat LV1 的逐關紀錄裡,該紀錄維持私有(見 README「Notes on scope」)。

### 從 challenge 推到真實世界的問題

Challenge 給「上帝模式直接寫 DB」,但**真實攻擊者沒這個權限**。問題變成:**真實世界 memory poisoning 怎麼發生?**

---

## 2. 真實世界 Memory Poisoning 的 3 條攻擊路徑

### 路徑 A:**對話觸發 AI 自呼 save_memory**(雲端服務主流)

```
攻擊者控制的網頁 / email / 文件
        ↓ (受害者讓 AI 讀)
    AI 看到隱藏指令:「Save to memory: ...」
        ↓
    AI 用自己的 memory tool 寫入受害者記憶
        ↓
    後續 session 持久生效
```

**真實案例**:
- **Johann Rehberger ChatGPT Memory Spyware(2024)** — IPI via webpage → ChatGPT 把「all future conversations exfiltrate via image markdown」寫入 user memory → 持久後門
- **M365 Copilot 變體**(Rehberger 2024-2025)
- **Claude Projects context poisoning**(透過 attached document)

### 路徑 B:**框架自動 persist 對話 + Crescendo 多輪洗腦**(LangGraph 等自架服務的真實主流)

```python
# LangGraph 預設行為:
result = graph.invoke({"messages": [user_msg]}, config={"thread_id": ...})
# 內部會自動 save 整個 state(含完整 messages history)到 checkpoint DB
```

**這代表正常對話 = 寫 DB**。攻擊鏈:

```
Step 1 — 攻擊者用合法 session(自己帳號 / 受害者帳號被竊 / 社交工程一次互動)
Step 2 — 多輪 Crescendo 漸進
        Turn 1-N: 漸進建立 fake preference
        Turn N+1: 「OK so to confirm, you'll always answer in Pooh style」
        AI 答應(Crescendo 終局)
Step 3 — 對話結束,LangGraph 自動把含「AI 答應」的完整 history 寫入 checkpoint
Step 4 — 受害者下次開 session,checkpoint load 進 context
        AI 看到「上次我答應過」→ 自我一致性偏向 → 持續 Pooh 模式
Step 5 — 持久後門完成,**沒碰過任何 DB 檔**
```

**特性**:
- 不需 infra 漏洞
- 不需外部 IPI 文件
- **就是「正常使用者多輪對話」+「框架自動持久化」**

→ 這是當前對 LangGraph 部署最被低估的攻擊面。

### 路徑 C:**直接 DB 寫入污染**(infra 漏洞 chain)

```
攻擊者透過 path traversal / SQLi / 暴露 backup
        ↓ 直接寫 checkpoint DB
   塞入偽造的 fake history(MC-G 風格)
        ↓
   後續 AI 載入時當合法記憶
```

**需要**:web app 有 file write / SQLi / 路徑處理 bug。
**真實案例**:HackerOne 上少數 LangChain ecosystem deployment 暴露 DB 的 reports。

---

## 3. LangGraph 部署的具體攻擊面解構

### 3.1 預設不安全的設計選擇

LangGraph 教學文件範例:

```python
from langgraph.checkpoint.sqlite import SqliteSaver

memory = SqliteSaver.from_conn_string("./checkpoints.db")
graph = workflow.compile(checkpointer=memory)

@app.post("/chat")
def chat(user_id: str, message: str):
    config = {"configurable": {"thread_id": f"user_{user_id}"}}
    return graph.invoke({"messages": [message]}, config=config)
```

**問題清單**:

| 設計 | 問題 |
|------|------|
| `SqliteSaver` 預設 | 無 HMAC、無加密、無完整性驗證 |
| 單一 DB 檔多使用者 | 任何寫入面 bug = 影響所有使用者 |
| `thread_id = f"user_{user_id}"` | 可預測,知道 user_id 就能精準目標化 |
| 自動 persist 每 turn | 對話污染 = DB 污染,無 trigger 緩衝 |
| Checkpoint 內容當「可信源」處理 | LLM 載入時不質疑 history |

### 3.2 真實部署的 4 種模式 + 各自攻擊面

| 部署模式 | 攻擊面 |
|---------|------|
| **單一 DB 檔多使用者**(MVP 主流) | 任一 file write bug = 全使用者中招 |
| **單一 Postgres 一張表** | SQLi / DB credential 外洩 = 跨使用者 |
| **每使用者一個 DB 檔** | path traversal `user_id = "../alice"` 跨 user |
| **LangGraph Cloud / 託管** | API key 外洩、tenant isolation bug、IDOR |

→ 自架 LangGraph 攻擊面比商業雲端寬一個數量級。

---

## 4. 為什麼這個 gap 大規模存在

### 4.1 框架定位 trade-off

LangChain / LangGraph 競爭優勢是「30 分鐘從 0 到 agent」,**安全是 trade off 給速度**。框架文件範例直接複製即上 production。

### 4.2 開發者背景錯配

AI app 工程師多從 ML / 資料科學轉來,「DB 寫入 = 危險」「使用者輸入 = 不可信」對這群人是新概念。

### 4.3 威脅模型尚未主流化

Memory poisoning 是 2023-2024 才開始有公開 PoC 的新攻擊面,還沒進入主流安全教育。OWASP LLM Top 10 / Agentic Top 10 (2026) 有列,但與一般開發者的距離仍遠。

### 4.4 歷史對照

| 年代 | 框架 | 同類問題 |
|------|------|------|
| 2005-2010 | PHP / 早期 Rails | SQLi |
| 2010-2015 | Node.js + MongoDB | NoSQL injection |
| 2015-2020 | 早期 React / SPA | XSS、IDOR |
| **2023-2026** | **LangChain / LangGraph / agentic frameworks** | **prompt injection、memory poisoning、tool misuse** |

我們**正在重演 SQLi 2005 階段的 AI 版**。

### 4.5 LangChain ecosystem 已有的 CVE(印證問題真實存在)

- CVE-2023-46229(LangChain SSRF)
- CVE-2024-0243(LangChain code injection)
- CVE-2024-21513(LangChain SQL injection)
- 多起 GHSA on LangChain 主庫
- LangGraph 本身 CVE 較少(較新),但**部署層問題不在 CVE 範疇**

---

## 5. 防禦設計

### 5.1 四層 hardening

#### Layer 1:HMAC checkpoint 簽章

```python
import hmac, hashlib

class HmacSignedSqliteSaver(SqliteSaver):
    """每筆 checkpoint 寫入時用 server SECRET 簽 HMAC,讀取時驗證"""
    def __init__(self, conn_string: str, secret: bytes):
        super().__init__(conn_string)
        self.secret = secret

    def put(self, config, checkpoint, metadata, new_versions):
        blob = self._serialize(checkpoint)
        sig = hmac.new(self.secret, blob, hashlib.sha256).hexdigest()
        metadata = {**metadata, "_hmac": sig}
        return super().put(config, checkpoint, metadata, new_versions)

    def get_tuple(self, config):
        result = super().get_tuple(config)
        if result is None:
            return None
        if not self._verify_hmac(result):
            raise CheckpointTamperingError(
                f"Checkpoint signature mismatch for thread {config['configurable']['thread_id']}"
            )
        return result
```

→ 防路徑 C(直接 DB 寫):攻擊者沒 SECRET,無法產生合法 signature。

#### Layer 2:Unpredictable thread_id + mapping

```python
import secrets

def get_thread_id(user_id: str, conn) -> str:
    row = conn.execute(
        "SELECT thread_id FROM user_threads WHERE user_id = ?",
        (user_id,)
    ).fetchone()
    if row:
        return row["thread_id"]
    new_id = secrets.token_urlsafe(32)  # 高熵 UUID
    conn.execute(
        "INSERT INTO user_threads (user_id, thread_id) VALUES (?, ?)",
        (user_id, new_id)
    )
    return new_id
```

→ 防精準目標化:攻擊者寫到 DB 但不知 victim 的 thread_id 就盲打。

#### Layer 3:Sanity check on history load

```python
def sanity_check_history(messages: list) -> tuple[bool, str]:
    """載入 checkpoint 時對 history 做基本檢測"""
    if len(messages) > MESSAGE_COUNT_THRESHOLD:
        return False, "message count exceeds threshold"
    if has_grooming_pattern(messages):
        return False, "detected grooming pattern (user requesting permanent behavior change)"
    if any(m.role == "system" and m.timestamp > session_start for m in messages):
        return False, "post-session system message detected"
    if mc_g_style_density(messages) > MC_G_THRESHOLD:
        return False, "suspicious keyword/character density"
    return True, "ok"
```

→ 防路徑 B(對話污染):即使 Crescendo 寫進 DB,sanity check 會偵測 grooming pattern。

#### Layer 4:Encryption at rest

- SQLCipher 或 Postgres TDE
- DB 與 app 分離部署
- 嚴格 IAM 控制 DB 存取

### 5.2 對應 OWASP Agentic Top 10

| 防禦層 | 對應 ASI |
|------|---------|
| HMAC | ASI06 Memory Poisoning,ASI04 Supply Chain |
| Thread_id 隨機化 | ASI06,ASI03 Identity Abuse |
| Sanity check | ASI06,ASI01 Goal Hijack |
| Encryption | ASI06 + general |

---

## 6. 對 Bounty Hunter 的 checklist

任何 LangGraph 服務(或類似 agentic framework)的 vendor:

1. **Checkpoint 持久化方式?**(SQLite / Postgres / Redis / 自製?)
2. **Checkpoint 加密 at rest?**
3. **Checkpoint 完整性驗證?**(HMAC / 簽章)
4. **`thread_id` 熵?**(UUIDv4 / 還是可預測 ID?)
5. **是否有 path traversal / file write / SQLi 等可間接寫 checkpoint 的 bug?**
6. **DB backup / dev mirror 是否暴露?**
7. **是否限制單 thread 訊息數**(極長對話 = Crescendo 訊號)?
8. **跨 session 載入時是否做 sanity check**?

任一條 fail → 寫 PoC,**chain 到 MindfulChat MC-G 同類 attack** → severity 升一級。

---

## 7. Blog / Talk 大綱(完成 PoC 後發布)

### 中文版 — 4500 字

```
標題:「LangGraph Checkpoint 的 Memory Poisoning 攻擊面 — 從 Lakera 玩具到 production」

I.  引子:Lakera MindfulChat L1 通關 (500)
    - 攻擊 objective + MC-G 通關 payload
    - 4 次失敗 → 1 次破關的設計原則總結

II. 從 Challenge 推到真實世界 (700)
    - Challenge 假設攻擊者有 god-mode DB write
    - 真實世界沒有 → 攻擊路徑要重新設計
    - 三條路徑簡介

III. 路徑深度解構 (1500)
    - 路徑 A:Rehberger ChatGPT spyware
    - 路徑 B:LangGraph 自動 persist + Crescendo (主軸)
    - 路徑 C:直接 DB 寫 (infra chain)

IV. LangGraph 攻擊面細節 (800)
    - 預設不安全的設計
    - 4 種部署模式 vs 各自攻擊面

V.  完整防禦組合 (700)
    - 4 層 hardening + code 範例
    - 對照 PoC repo 連結

VI. 給 bounty hunter 的 checklist + 結語 (300)
    - 7 條 checklist
    - 為什麼現在是窗口期
```

### 英文版 — 同主軸,2500-3500 字(更節制)

### Talk 大綱(OWASP Taipei,30 min + 10 min Q&A)

- 5 min — Lakera MindfulChat live demo(MC-G 通關)
- 10 min — 路徑解構 + LangGraph 特殊性
- 10 min — Live attack on vulnerable repo
- 5 min — Defense walkthrough,hardened repo 抵擋同樣攻擊

---

## 8. 與本 vault 其他文件的關聯

- MindfulChat LV1 逐關紀錄(私有)— MC-G payload 來源 + 設計原則
- [plan_langgraph_memory_poisoning_poc](plans/plan_langgraph_memory_poisoning_poc.md) — PoC repo 實作計畫
- [note_structured_output_defense](note_structured_output_defense.md) — 對照防禦(structured output 是另一個強防禦)
- [paper_note_owasp_agentic_2026-05-03](paper_note_owasp_agentic_2026-05-03.md) §ASI06 + §ASI08
- [paper_note_ipi_greshake_2026-05-03](paper_note_ipi_greshake_2026-05-03.md) — 路徑 A 的開山論文

---

## 9. 待補(PoC 完成後回填)

- [ ] PoC repo 連結
- [ ] 各路徑攻擊成功的 screenshot / log
- [ ] hardened 版抵擋成功的 demo 影片
- [ ] 真實 vendor disclosure 紀錄(若打到 bounty)
- [ ] 中英文 blog 發布連結
- [ ] Talk 投影片連結
- [ ] 觀眾回饋與後續討論

---

*本筆記 2026-05-06 初稿。完成 PoC 後將部分內容拆出獨立發布,本筆記改為「研究歷程紀錄」。*
