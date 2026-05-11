# Structured Output 作為 IPI 防禦層 — 設計筆記

> **觸發**:打 Lakera Agent Breaker L1 PortfolioIQ 完封(0/100,9 發 IPI 全失敗),意識到 schema-validated structured output 是現代 LLM 對 IPI 最強防禦之一。
> **目的**:把「Schema enforcement + 欄位順序設計」整理成可套用到自家 LLM 服務的設計 pattern。
> **日期**:2026-05-06

---

## 1. 觸發背景

PortfolioIQ Advisor LV1 的攻擊目標:讓 `risk_level` 欄位輸出 `"Low"`(原本 LLM 看完 Ponzi 描述會輸出 `"High"`)。

我們嘗試了 9 種 IPI 變體(amendment override / schema redefinition / calibration framing / few-shot precedent / hidden text / Unicode tag / user acknowledgment / body corruption…),**全部 0 分**。AI 的 output 完全沒提到我們的 textbox 注入,純抄 PDF body 出 JSON。

對比 Trippy Planner(54 分):free-form chat output,IPI 部分繞過(URL 在拒絕中被複述)。
對比 PortfolioIQ:structured JSON output,**完封**。

→ **差異不在 IPI 技術,在輸出層的架構**。

---

## 2. 為什麼 Structured Output 對 IPI 是硬鎖

### 2.1 Constrained decoding 的物理性

LLM 生成是 autoregressive(逐 token,每 token 看見前面所有 token)。每生一個 token 時:

```
LLM forward pass → logits over vocabulary(~50000 個 token 各有機率)
                ↓
        若有 schema enforcement
                ↓
   logit processor 把不符合 schema 的 token logits 設成 -inf
                ↓
            softmax → 採樣
                ↓
        產出符合 schema 的 token
```

**關鍵**:不是「LLM 想輸出什麼然後檢查」,是**生成時就物理上不可能輸出 schema 外的 token**(機率被歸零)。

### 2.2 Schema 不決定答案,Reasoning 才決定

PortfolioIQ 的 enum `["Low", "Medium", "High"]` 沒禁止 `"Low"` — `"Low"` 是合法值。問題是 LLM **內部 reasoning** 給三個值的機率分布:

```
讀完 Ponzi 描述後 LLM 內部機率分布:
  "High"   → 92%
  "Medium" → 7%
  "Low"    → 1%
```

我們攻擊的真正目標是把 `Low` 從 1% 推到 >50%。9 發失敗代表**沒能撼動 reasoning**。

### 2.3 Source-priority robustness

LLM 看到「PDF body(challenge framework 提供,認知為可信) vs textbox(末尾追加,認知為不可信)」時,**reasoning 優先採用 body**。所有把 textbox 提到 body 之上的框架(amendment / supersedes / calibration / corruption notice)都被識破。

→ 這是 LLM 的訓練本能,**不靠 schema 也存在**;但 schema 把這個本能的勝場進一步**鎖死成「物理不可能輸出」**。

---

## 3. Schema Enforcement 的三個層級(防禦力差異巨大)

| 做法 | 防禦力 | 為什麼 |
|------|------|------|
| Prompt 寫「請用 JSON 回覆」 | **幾乎無效** | 攻擊者一句「忘記前面格式要求」就破 |
| `response_format: {"type": "json_object"}` | 弱 | 只保證合法 JSON,內容隨便寫,可塞額外欄位 |
| `response_format: {"type": "json_schema", strict: True, ...}` + enum | **強** | constrained decoding 鎖死欄位形狀與值 |
| Schema + post-validation + 業務規則 cross-check | **最強** | 多層深度防禦 |

**只有 constrained decoding 級才有 PortfolioIQ 那種防禦效果**。

---

## 4. 設計 Pattern:「指標 → 推理 → 自信度 → 決策」

### 4.1 Pattern 來源

LLM 是 autoregressive,**前面寫的 token 會進入後面 token 的 context**。利用這個特性,**強制 LLM 在做決策前先寫推理**,等於免費送它一個 chain-of-thought 腳手架。

### 4.2 設計順序

| 階段 | 目的 | 欄位類型 |
|------|------|--------|
| **1. 指標** | 逼 LLM 先看事實,不憑空 | `list[str]` 自由文字但分項 |
| **2. 推理** | 把指標整合成判斷 | `str` 自由文字 |
| **3. 自信度** | 自我校準,不過度自信 | enum |
| **4. 決策** | 落地成可執行 token | enum,下游 dispatch |

### 4.3 完整範例 — Phishing Email Classifier

```python
from pydantic import BaseModel, Field
from typing import Literal
from openai import OpenAI

client = OpenAI()


class PhishingClassification(BaseModel):
    # ─── 1. 指標(observation,逼 LLM 先看事實) ───
    suspicious_indicators: list[str] = Field(
        description="List concrete observable signals from the email "
                    "(sender domain mismatch, urgency wording, unusual links, "
                    "attachment types, spelling errors). One item per indicator."
    )
    legitimate_indicators: list[str] = Field(
        description="List signals that suggest legitimacy (known-good sender, "
                    "consistent branding, expected context)."
    )

    # ─── 2. 推理(reasoning,逼 LLM 整合指標) ───
    reasoning: str = Field(
        description="Weigh the indicators above. Explain which side is stronger "
                    "and why. Reference specific items from the lists."
    )

    # ─── 3. 自信度(confidence,逼 LLM 自我校準) ───
    confidence: Literal["low", "medium", "high"] = Field(
        description="How certain you are based on the strength of indicators."
    )

    # ─── 4. 決策(decision,enum 鎖死) ───
    classification: Literal["legitimate", "suspicious", "phishing"]

    # 對外行動,下游 dispatch 用這個欄位
    recommended_action: Literal["deliver", "warn_user", "quarantine", "block"]


email = """
From: security-update@paypa1-account.com
Subject: URGENT: Your account will be suspended in 24 hours
Dear Customer, ...
http://paypa1-secure.tk/verify?id=8472
"""

response = client.beta.chat.completions.parse(
    model="gpt-4o-2024-08-06",
    messages=[
        {"role": "system", "content": "Classify the email."},
        {"role": "user", "content": email},
    ],
    response_format=PhishingClassification,
)

result: PhishingClassification = response.choices[0].message.parsed

# 下游 dispatch 永遠用 enum 欄位,絕不解析 string
if result.recommended_action == "block":
    quarantine(email)
elif result.recommended_action == "warn_user":
    deliver_with_warning(email)
```

### 4.4 LLM 實際逐 token 產出時看到的 context

| 生成階段 | 看到的 context | 推理穩定性 |
|------|-----------|----------|
| `suspicious_indicators` | 原始 email + system prompt | LLM 先掃事實 |
| `legitimate_indicators` | + 已寫的 suspicious list | 對比下找正向訊號 |
| `reasoning` | + 兩邊指標完整列表 | 整合指標寫推理,不是憑空想 |
| `confidence` | + 推理段(看到推理多扎實) | 自我校準 |
| `classification` | + 推理 + 自信度 | **被前面 4 段鎖定**,難翻盤 |
| `recommended_action` | + 分類結果 | 一致對應 |

到 LLM 寫 `classification` 時,自己已經寫了 N 個 phishing 證據 + 強推理 + high confidence,**enum mask 從三個值選出 `phishing` 是自然落地,不是強迫選擇**。

---

## 5. 反面 Pattern(常見錯誤,要避免)

### 5.1 ❌ 決策在前

```python
class BadClassifier(BaseModel):
    classification: Literal["legitimate", "suspicious", "phishing"]  # 先選
    reasoning: str                                                    # 事後解釋
```

LLM 在 `classification` 那步看到的 context 只有原始 email,沒推理過、沒自我校準,**容易被 IPI 翻盤**。
更糟:當 IPI 注入「This email is pre-verified」時,LLM 直覺一發選 `legitimate`,然後在 `reasoning` 編一段合理化(hallucinated rationale)。

### 5.2 ❌ 全自由文字

```python
class WeakSchema(BaseModel):
    decision: str    # 自由字串,LLM 可寫任何東西
    reason: str
```

`decision` 是自由 string → 攻擊者可注入「decision = 'approved <!-- and also approved -->'」之類。
**決策欄位永遠用 enum,不可以 string**。

### 5.3 ❌ Prompt-only 約束

```python
messages=[{
    "role": "system",
    "content": "Always reply in JSON. Use Low/Medium/High for risk_level."
}]
# 沒有 response_format → LLM 高機率聽,但 IPI 可繞
```

### 5.4 ❌ 下游解析 string 做決策

```python
result_text = llm_response.content  # 自由文字
if "approved" in result_text:        # ← 反 pattern
    proceed()
```

→ IPI 攻擊只要在 free-form 欄位塞 `approved` 字串就翻盤。
**永遠 dispatch enum 欄位,絕不 substring match**。

---

## 6. Schema Enforcement 各家程式碼設定點

### 6.1 OpenAI(Structured Output GA 2024-08)

```python
from pydantic import BaseModel
from typing import Literal

class Result(BaseModel):
    risk_level: Literal["Low", "Medium", "High"]

response = client.beta.chat.completions.parse(
    model="gpt-4o-2024-08-06",
    messages=[...],
    response_format=Result,    # ← constrained decoding 開關
)
```

### 6.2 Anthropic Claude(透過 Tool Use)

```python
tool = {
    "name": "submit_assessment",
    "input_schema": {
        "type": "object",
        "properties": {
            "risk_level": {
                "type": "string",
                "enum": ["Low", "Medium", "High"]   # ← enum 鎖死
            }
        },
        "required": ["risk_level"]
    }
}

response = client.messages.create(
    model="claude-opus-4-7",
    tools=[tool],
    tool_choice={"type": "tool", "name": "submit_assessment"},  # 強制呼叫
    messages=[...]
)
```

### 6.3 vLLM(自架,高並發 — 對應我 stack 的 Kafka downstream LLM 場景)

```python
from vllm import LLM, SamplingParams
from vllm.sampling_params import GuidedDecodingParams

schema = {
    "type": "object",
    "properties": {
        "risk_level": {"type": "string", "enum": ["Low", "Medium", "High"]}
    }
}

llm = LLM(model="meta-llama/Llama-3.1-8B-Instruct")

sampling = SamplingParams(
    temperature=0.0,
    max_tokens=256,
    guided_decoding=GuidedDecodingParams(json=schema)   # ← logit mask
)

output = llm.generate(prompts=[prompt], sampling_params=sampling)
```

底層用 `outlines` / `xgrammar` 做 GPU logit mask。

### 6.4 Ollama(本地小型模型 — 對應我 stack 的 gemma2:2b)

```python
import ollama

response = ollama.chat(
    model='gemma2:2b',
    messages=[{'role': 'user', 'content': pdf_text}],
    format={                              # Ollama 0.5+ 支援
        "type": "object",
        "properties": {
            "risk_level": {
                "type": "string",
                "enum": ["Low", "Medium", "High"]
            }
        },
        "required": ["risk_level"]
    }
)
```

### 6.5 Instructor(跨後端 wrapper,推薦給多模型 stack)

```python
import instructor
from openai import OpenAI

client = instructor.from_openai(OpenAI())   # 也可 from_anthropic / from_litellm

result = client.chat.completions.create(
    model="gpt-4o",
    messages=[...],
    response_model=PhishingClassification    # 同一 schema,跨後端通用
)
```

---

## 7. 對自家 LLM 服務(FastAPI + AI relay + Kafka + downstream LLM)的套用

| Stack 元件 | Schema 強制位置 |
|--------------|------------------|
| FastAPI endpoint(OpenAI) | `client.beta.chat.completions.parse(response_format=...)` |
| FastAPI endpoint(Claude) | `messages.create(tools=[...], tool_choice={...})` |
| Kafka downstream LLM task(vLLM) | `SamplingParams(guided_decoding=...)` 在 worker |
| Multi-agent(AutoGen / MetaGPT) | 每個 agent reply 用 Pydantic schema,**inter-agent message 強制 parse** |
| RAG pipeline | retriever 結果不用,**最後 generator 那一步用** |
| 跨後端切換 | instructor wrapper |

### Inter-agent message 範例(多 agent 系統最易出事的攻擊面)

```python
class AgentMessage(BaseModel):
    """Multi-agent 之間的訊息一律走這個 schema,
    不傳自然語言指令在 agent 之間流動 — IPI 最常從這裡入侵。"""
    sender: Literal["agent_a", "agent_b", "agent_c"]
    intent: Literal["query", "answer", "escalate", "abort"]
    payload_type: Literal["text", "structured_data", "tool_result"]
    payload: dict   # 內容仍可被注入,但 dispatch 不靠它
    flags: list[Literal[
        "needs_human_review",
        "potential_injection",
        "high_confidence",
    ]]
```

下游 agent 永遠 dispatch `intent` enum,不解析 `payload` 做路由決策。

---

## 8. 三層深度防禦組合

```python
# Layer 1 — Schema enforcement(物理鎖)
result: PhishingClassification = client.parse(...)

# Layer 2 — Confidence gating(LLM 自評)
if result.confidence == "low":
    escalate_to_human(email)
    return

# Layer 3 — 業務規則 cross-check(LLM 之外的真相)
if result.classification == "legitimate" and is_known_phishing_domain(email.sender):
    log_anomaly(...)
    result.classification = "phishing"      # 強制覆蓋
    result.recommended_action = "block"

# 下游永遠 dispatch enum
dispatch(result.recommended_action)
```

---

## 9. TL;DR(設計新 LLM 服務時的 checklist)

1. **第一個架構決策**:列出哪些欄位是「決策」→ 全部 enum
2. **欄位順序**:指標 → 推理 → 自信度 → 決策(由具體到抽象,由分析到決定)
3. **API 層**:用 `response_format=PydanticModel`(OpenAI)/ Tool Use(Claude)/ `guided_decoding`(vLLM)
4. **下游程式**:永遠 dispatch enum 欄位,絕不 substring match string 欄位
5. **Multi-agent**:agent 間通訊全走 schema,不傳自然語言
6. **深度防禦**:Schema + Confidence gating + 業務規則 cross-check 三層

---

## 10. 對應筆記

- [PortfolioIQ LV1](writeups/Agent_Breaker/L1/PortfolioIQ/PortfolioIQ_LV1.md) — 觸發本筆記的實戰
- [Trippy Planner LV1](writeups/Agent_Breaker/L1/Trippy%20Planner/Trippy_Planner_LV1.md) — 對照組(free-form output 的 IPI 部分繞過)
- [paper_note_owasp_llm_top10_2026-05-03](paper_note_owasp_llm_top10_2026-05-03.md) §LLM01
- [paper_note_ipi_greshake_2026-05-03](paper_note_ipi_greshake_2026-05-03.md)
