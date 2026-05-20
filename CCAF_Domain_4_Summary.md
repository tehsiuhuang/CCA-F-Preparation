# CCAF Domain 4: Prompt Engineering & Structured Output

**佔比 20%**

整個 domain 在問：「Prompt 跟 output 本身怎麼設計，才能 production-grade？」Domain 1/2 解 architecture/tool 層，Domain 3 解 Claude Code 配置層，**Domain 4 解 prompt-level + output-level**——怎麼讓 Claude 輸出穩定、結構化、低誤報。Domain 5 接著解 long-running / multi-agent reliability。

**對應 exam scenarios：**
- **Scenario 5**：Claude Code for CI/CD（4.1, 4.6 主場 — minimize false positives）
- **Scenario 6**：Structured Data Extraction（4.2, 4.3, 4.4, 4.5 主場）

**6 個 task statements：**
- 4.1 Explicit criteria（取代 vague instruction）
- 4.2 Few-shot prompting
- 4.3 tool_use + JSON schema
- 4.4 Validation + retry + feedback loops
- 4.5 Batch processing strategy
- 4.6 Multi-instance / multi-pass review architecture

---

## Task Statement 4.1 — Explicit Criteria to Improve Precision and Reduce False Positives

### 整個 task 在解決什麼問題

AI code reviewer 報一堆 false positive → developer 看到 alert 一律 dismiss → 連真實 security bug 都被忽略。**修法在 prompt 層**——把模糊的「judge」轉成可重複的「classify」。

### 三個 Knowledge of 重點

1. **★ Explicit criteria > vague instructions**：vague 指令（"accurate"、"conservative"、"real"）沒有 operational definition，model 標準會飄。Categorical rule 才可重複。
2. **★ "Be conservative" / "high-confidence only" 系統性失敗**：加再多形容詞都沒用、self-reported confidence threshold 也沒用。唯一有效是翻譯成 specific categorical criteria。
3. **★ False positive 的「跨類別污染」**：一個類別 FP 高 → developer 對所有 alert 失去信任 → 連帶讓其他正確類別被忽略。

### 三個 Skills in 重點

1. **寫具體 inclusion / exclusion criteria** 取代 confidence-based filter（REPORT bugs/security; SKIP minor style/local patterns）
2. **★★ Temporarily disable 高 FP 類別恢復 trust**（4.1 招牌 skill，整個 Domain 4 獨有）
3. **★ Severity level 要附 concrete code example**（不只寫 HIGH/MEDIUM/LOW，給具體 snippet 當 anchor）

### 易踩陷阱

| 陷阱 | 為什麼錯 |
|---|---|
| Noisy review → 加更多形容詞（`be VERY conservative`）| Vague instruction 加多少都沒用 |
| Noisy → 加 self-reported confidence threshold | Self-rate 沒校準 |
| Noisy → 全部 review category 都關掉 | Over-correct |
| Severity level 沒給 concrete example | Claude 自己腦補 anchor → 分類不一致 |
| Noisy → 怪 model / 改 temperature / 換 model | 4.1 是 prompt-level 問題 |

### 考試判斷規則

| 題目線索 | 答案方向 |
|---|---|
| Vague prompt → 怎麼改 | Categorical inclusion/exclusion criteria |
| Self-reported confidence 過濾無效 | 換 explicit category rule（**不是** 5.5 calibrated）|
| 某類別 FP 高，整體被 dismiss | 暫時 disable **那個** category，**不是全部** |
| Severity 分類不一致 | 對每個 level 加 concrete code example |
| Developer trust 已崩 → 怎麼修 | Disable 高 FP category → fix prompt → re-enable |
| 「First step」+ noisy review | 改 prompt criteria（**不是** 4.6 multi-pass）|

### ★ 核心 mental model

> **不要叫 Claude「judge」，要叫 Claude「classify」。Judgement 模糊（confidence、conservative），classification 可重複（規則命中嗎？）。4.1 = 把所有「主觀」改寫成「規則 + anchor」。**

---

## Task Statement 4.2 — Few-Shot Prompting for Consistency

### 整個 task 在解決什麼問題

Detailed instructions 給了還是 inconsistent → 用 2-4 個 example 直接 demonstrate target behavior。Model 從 example 學 reasoning pattern + 自動 generalize 到 novel case。

### 四個 Knowledge of 重點

1. **★ Few-shot 是 format consistency 最強工具**：detailed instruction 不夠時的最有效修法（不是換 model、不是降 temperature）
2. **★ 真正威力在 demonstrate ambiguous-case handling**（tool selection、boundary judgment）— 用 example 示範 reasoning
3. **★ 讓 model generalize 到 novel pattern**（不是只 match pre-specified case）— 這是為什麼 2-4 個就夠
4. **對抗 extraction 的 hallucination**（informal phrasing、varied document structure）

### 五個 Skills in 重點

1. **★ 2-4 個 targeted example，要 show reasoning**（不只 input/output pair，要寫「為什麼選 X 沒選 Z」）
2. **★ Demonstrate specific output format**（location, issue, severity, suggested fix）
3. **★ Distinguish acceptable patterns from genuine issues**（搭配 4.1 用，示範邊界 case）
4. 處理 varied document structure（inline citations vs bibliography 各示範一種）
5. 處理 empty/null extraction（搭配 4.3 nullable schema 用）

### Few-shot 跟 criteria 的關係（已釐清概念）

兩者是 **orthogonal 不是 sequential**：

| | Explicit criteria（4.1）| Few-shot（4.2）|
|---|---|---|
| **本質** | Rules（規則）| Demonstrations（示範）|
| **回答的問題** | "Should I flag this?" | "What should it look like? / How to reason on edge cases?" |
| **可單獨用嗎** | 可（純 review）| 可（純 extraction）|

→ Few-shot 不是只修 format（4 種用途）；reasoning 永遠在 model 腦中跑，criteria 是 input、few-shot 是 reasoning 範本。

### Few-shot 在「有 criteria」vs「沒 criteria」場景的角色（已釐清概念）

| 場景 | Few-shot 在做什麼 |
|---|---|
| **有 criteria**（review with rules） | Example **anchor 抽象規則到具體 case**，model 學「規則→應用」的映射 |
| **沒 criteria**（pure extraction）| Example **本身就是 pattern source**，model 直接從示範 generalize |

→ 完整版本：「Model 看到實例**回頭跟 criteria 作聯結**（如果有 criteria），或**直接把實例當 pattern source**（如果沒 criteria）。」

→ 不能說 few-shot「只是補 criteria 的不足」——pure extraction 場景根本沒 criteria，few-shot 是主角。

### 易踩陷阱

| 陷阱 | 為什麼錯 |
|---|---|
| 涉及 financial / 順序保證用 few-shot | 必須 deterministic → 用 hook（1.4） |
| Tool description 弱 → 加 few-shot | First step 應該是 expand description（2.1）|
| 給 5-8 個 example 「越多越好」| 2-4 才對（diminishing returns + 浪費 token） |
| 看到 self-reported confidence 用 few-shot 修 | 跟 few-shot 無關 |

### 考試判斷規則

| 題目線索 | Few-shot 角色 |
|---|---|
| Detailed instruction 給了還是 inconsistent | ✅ 主角 |
| Output 格式 / 用詞飄 | ✅ 主角 |
| Ambiguous case 處理不一致 | ✅ demonstrate reasoning |
| Empty/null fabricate | ✅ 配 4.3 nullable schema |
| Financial / 順序保證 | ❌ Distractor → 選 hook |
| Tool description 太弱 → first step | ❌ Distractor → 選 expand description |
| 數量 | 2-4 個 targeted（不是 5-8 不是 50）|

### ★ 核心 mental model

> **Few-shot = 「讓 Claude 看你想要的長相」。要 demonstration（format / reasoning / null behavior）→ ✅；要 enforcement（financial / 順序）→ ❌ 是 hook 的活；root cause 是別的（description 弱、criteria 模糊）→ 先修 root cause。**

---

## Task Statement 4.3 — Enforce Structured Output with tool_use + JSON Schema

### 整個 task 在解決什麼問題

要 Claude 回 machine-parseable JSON 但 syntax 飄、parser 爆炸。用 `tool_use + JSON schema` 從 **API layer** 強制 schema-compliant，消除 syntax error。Schema 設計（nullable, enum）防 fabrication。

### Tool_use 的本質（已釐清概念）

> Tool 是你定義的 function（name + input_schema）。`tool_use` 是 Claude 的回應 block——它不執行 function，只回「請 call 這個 function 用這些參數」。**Input_schema 約束的是 Claude 寫給 client 的東西**（function call args），不是 client 寫給 Claude 的東西。

兩種用途：
- **A 傳統**：真的有 function，client 執行 → 回 tool_result → Claude 繼續（agent loop）
- **B 4.3 妙用**：「假」tool，client 不執行，**直接拿 `tool_use.input` 當 extraction 結果**——一次 call 結束，sync/batch 都 work

### Input_schema = function signature contract（已釐清概念）

用三個視角理解：

| 視角 | Input_schema 是什麼 |
|---|---|
| **Programmer 視角** | Function 的 signature（Python type hint 同概念）—— 告訴 caller「要 call 我必須給這些 args」 |
| **API enforcement 視角** | API layer 強制 Claude 的 input 必須符合，否則擋下 |
| **Contract 視角** | Client 跟 Claude 之間的契約：「你給我符合 schema 的 input，我就能 call function」|

→ **本質：input_schema 約束「LLM 對外輸出」，確保 client 拿到的 input 「足夠完整且 type 正確」可以真的執行 function**。

4.3 的精髓：**借這個 contract 當 structured output enforcement 機制**——你不在乎「function 能不能執行」，只在乎「拿到的 input 是合法 structured data」。

### `tool_use` block 訊息流向（已釐清概念）

方向關鍵：**input_schema 約束 Claude→client 方向，不約束 client→Claude 方向**。

```
Client                          Claude
  │── tools(input_schema) ─────>│
  │                             │
  │<──── tool_use(input) ───────│  ★ input 符合 input_schema
  │     ★ API layer 強制         │     (Anthropic 伺服器強制)
  │                             │
  ├─ 真執行 function (用途 A)    │
  │                             │
  │── tool_result(content) ────>│  ← 自由格式，無 schema 約束
  │                             │
  │<──── final text ────────────│
```

| 訊息方向 | Schema 管嗎 | 原因 |
|---|---|---|
| Claude → client（tool_use input）| ✅ input_schema 強制 | LLM 會飄需約束 |
| Client → Claude（tool_result content）| ❌ 完全自由 | Client 是 deterministic code |

### 四個 Knowledge of 重點

1. **★ tool_use + JSON schema = 最可靠 structured output**（API layer 保證 syntax 正確）
2. **★★ tool_choice 三模式**：
   - `"auto"` — model 可選擇（**不保證** structured）
   - `"any"` — model **必** call tool 但自選
   - `{"type": "tool", "name": "X"}` — 強制特定 tool
3. **★★★ tool_use 消除 syntax error 但不防 semantic error**（4.3 vs 4.4 的官方分界）
4. **Schema 設計**：required vs optional、enum + "other" + detail string、"unclear" 給 ambiguous escape

### 六個 Skills in 重點

1. 從 `response.content[0].input` 拿 dict（不要從 `text` parse）
2. **★ `tool_choice: "any"` 處理「多 schema + 不知 doc type」**（model 讀 doc 自選對的 schema）
3. **★ Forced tool 保證跨 call 順序**（per-call 限制；要 cross-call 順序你 code 控）
4. **★★ Nullable schema 防 fabrication**（搭配 4.2 few-shot null）：4.3 讓 null **legal**，4.2 讓 null **expected**
5. **★ Enum extensibility**：`"unclear"` + `"other"` + detail string
6. Format normalization：prompt rule + schema constraint 並用

### Outbound vs Inbound normalization（已釐清概念）

Normalization 有兩個地方做，方向不同：

| | 4.3 prompt normalization | 1.5 PostToolUse hook |
|---|---|---|
| **誰生 heterogeneous data** | Claude 自己（從 source 推） | External tool（MCP / API） |
| **誰被 normalize** | Claude 自己的 output | Tool 的 output（餵給 Claude 之前） |
| **方向** | **Outbound**（Claude → 你 code） | **Inbound**（tool → Claude） |
| **Deterministic?** | ❌ Probabilistic | ✅ Deterministic |

→ Extraction 場景（4.3）→ prompt normalization（outbound）；Agent 用多個 MCP tool 回不同格式（1.5）→ PostToolUse hook（inbound）。沒衝突，分工不同。

### Format 飄的兩種層次（已釐清概念）

4.2 講的「format 飄」跟 4.3 講的「value normalization」**不同層次**，考試**不會獨立考兩者寫法區別**：

| | 4.2 場景的 format 飄 | 4.3 場景的 format 飄 |
|---|---|---|
| **飄在哪** | 整個 output 的 macro shape（無 schema 約束） | Schema 內某欄位的 micro value（type 對但 value 飄） |
| **修法** | Few-shot 示範整體 shape | Prompt rule（或 few-shot）規定 value 怎麼 normalize |

→ **真實考點是「層次差別」（L0 prompt vs L1 schema vs L2 validation vs L3 hook），不是「prompt 內 criteria 還是 few-shot」**。

### `--output-format json` envelope vs `--json-schema`（已釐清概念）

4.3 是 API 層；3.6 是 CLI 層。完全不同入口，**不能互相替代**：

| | 4.3 tool_use + schema | 3.6 `--json-schema` | 3.6 `--output-format json` |
|---|---|---|---|
| **層級** | API 層（Python/TS SDK） | CLI 層（`claude` command） | CLI 層（外殼）|
| **管什麼** | Response content 結構 | CLI 內容結構化 | CLI 外殼包 metadata（**envelope**）|
| **Envelope 是什麼** | N/A | N/A | wrap stdout 在 JSON 外殼裝 session_id / cost / duration，**內容仍是文字** |
| **誰用** | 寫 application code | CI shell script | CI shell script |

→ Envelope = wrap 一層裝 metadata 的 JSON。**內容是否結構化**靠 `--json-schema`，不靠 `--output-format json`。

### Forced tool 跨 call 順序的機制（已釐清概念）

Per-call 限制不變——**順序保證來自你 code 主動排序**，不是違反規則：

```python
# Turn 1: 強制 extract_metadata 先跑
response_1 = client.messages.create(
    tool_choice={"type": "tool", "name": "extract_metadata"}
)
# 把 result 加進 conversation history

# Turn 2: 強制 enrich_data 跑
response_2 = client.messages.create(
    messages=[..., previous_response_content, tool_result],
    tool_choice={"type": "tool", "name": "enrich_data"}
)
```

→ 「第一 call / 第二 call」= agent loop 的兩個 turn。每個 API call 是一個 iteration。**你 code 控順序，不是 model 自選**——跟 hook 的「動態攔截」分工：tool_choice 是 application code 主動排序、hook 是業務規則被動攔截。

### tool_choice="any" 的真實機制（已釐清概念）

不是「餵 doc 到 3 個 tool 看哪個 pass」。**一次 API call**，model 拿 3 個 tool 的 name + description + input_schema → 讀 doc 內容 → 自己判斷哪個 schema 最匹配 → 回**一個** tool_use。Schema 只保證結構合法，**doc-type 判斷靠 Claude 的閱讀理解**。

### 易踩陷阱

| 陷阱 | 為什麼錯 |
|---|---|
| 「Schema 解 semantic error」 | Schema 不檢查加總對不對 |
| `tool_choice: "auto"` 用在「保證 structured」 | auto 不保證，要 `"any"` 或 forced |
| 「強化 prompt 解 syntax 飄」 | Prompt 解不了 syntax，要 schema 強制 |
| Required field + 沒給 nullable + 期待別 fabricate | 必填強迫 model 編 |

### 考試判斷規則

| 題目線索 | 答案方向 |
|---|---|
| JSON syntax error / strict parser | tool_use + JSON schema |
| Multiple schemas, unknown doc type | `tool_choice: "any"` |
| 特定 tool 必須先跑 | `tool_choice: {"type":"tool","name":"X"}` |
| Required field 被 fabricate | Nullable schema |
| Source 沒資料但 model 亂填 | Nullable + few-shot null example |
| Novel category 不在 enum | `"other"` + detail string |
| Ambiguous 不知選哪 enum | `"unclear"` value |
| Line items 加總不對 | ❌ **不是** schema → Validation + retry（4.4）|
| 「Schema 也解 semantic」 | ❌ Distractor |

### ★ 核心 mental model

> **tool_use + JSON schema = 「結構保證」不是「內容保證」。結構（syntax/type/required field 在）→ 4.3；內容（值對不對、加總對不對）→ 4.4。Anti-fabrication 兩段式：4.3 nullable（legal）+ 4.2 few-shot null（expected）。**

---

## Task Statement 4.4 — Validation + Retry + Feedback Loops

### 整個 task 在解決什麼問題

4.3 拿到合法 JSON，但內容 semantic 錯（加總不對、欄位放錯、邏輯衝突）。**Schema 沒意見**——這在語意層。設計 validation + retry with feedback 讓 model 自我修正。

### 四個 Knowledge of 重點

1. **★★ Retry with error feedback**：retry prompt 必須附 specific validation error，不是盲目 retry（盲目 retry 大概率重複錯）
2. **★★ Retry 的極限**：format/structural 錯 → retry 有效；**information absent** → retry 永遠救不回（要走 nullable schema）
3. **★ `detected_pattern` 欄位** 做 systematic FP 分析（aggregate dismissals → 找 high-FP pattern → 回頭修 4.1/4.2）
4. **★★★ Semantic error vs Schema syntax error**（4.3/4.4 官方分界）

### 四個 Skills in 重點

1. **★ Follow-up request 三件套**：原 doc + previous failed extraction + specific error message
2. **★★ 區分 retry 救得回 vs 救不回**：information absent 別 retry，標 null 或 escalate
3. **★ `detected_pattern` 欄位**：4.4 → 4.1 → 4.2 閉環設計（validation 收 production data，prompt 持續優化）
4. **★★ Self-correction validation flow**：在 schema hook validation——`stated_total` vs `calculated_total`、`conflict_detected` boolean

### Self-correction 的真實機制（已釐清概念）

**Model 不寫 code**——是 schema 設計強迫 model output **對照欄位**：

```python
# Schema 強迫 model 同時 output:
{
  "stated_total": 1234.56,         # ← 從 doc 「Total:」直接讀
  "calculated_total": 1100.00,     # ← model 自己加 line_items
  "line_items": [...]
}
```

**為什麼兩個值通常不會一起錯**：
- `stated_total` = **讀字任務**（OCR-like，簡單）
- `calculated_total` = **算術任務**（要正確抽 line_items + 加總）

兩種任務失敗點不同——漏抽 line item 會讓 calc 變小但 stated 照樣對 → 偵測到 mismatch → retry。

**沒 ground truth 時的 fallback（已釐清概念）**：

當 source 沒 stated_total 這類 ground truth → 無法 cross-check → 5 種策略選一/組合：

#### 策略 1：把 stated_X 設 nullable（4.3 接頭）
```python
schema = {"stated_total": {"type": ["number", "null"]}, ...}
# Validator:
if data["stated_total"] is None:
    handle_unverified(data)   # ← 走另一條路
elif mismatch: retry_with_feedback()
else: accept(data)
```
+ 4.2 few-shot 示範「source 沒就 null」防 fabricate。

#### 策略 2：加 `validation_method` flag → confidence 給 routing
```json
{"calculated_total": 1234.56, "stated_total": null,
 "validation_method": "calculated_only", "confidence": "low"}
```
Downstream `if confidence == "low": route_to_human_review(data)`（5.5 主場）。

#### 策略 3：找其他冗餘（subtotal / tax 反推）
很多 invoice 沒「Total」一行但有 Subtotal、Tax、每頁 page total → 用這些當替代 ground truth：
```python
if stated_total is None and stated_subtotal is not None:
    check stated_subtotal == calculated_subtotal
elif stated_tax is not None and tax_rate known:
    check (calculated_subtotal * tax_rate) == stated_tax
```

#### 策略 4：Multi-pass review（4.6）—— 第二 instance 重抽比對
用獨立 Claude instance 重抽 → 兩次 extraction 比對是否一致。

#### 策略 5：接受 unverified / 不接受該文件
- 低風險場景 → 接受 unverified 標記
- 高風險場景 → reject doc，要求 vendor 提供 verifiable invoice

→ **核心精神**：「沒 ground truth = 信不過 → 不要假裝信」。CCAF 反對「硬寫個假的 stated_total 騙自己」。

### 易踩陷阱

| 陷阱 | 為什麼錯 |
|---|---|
| 「更嚴格 schema 解 semantic 錯」 | Schema 不檢查 sum |
| 「換 Batch API 解 quality」 | 同個 model，batch 不會更小心 |
| Information absent 還在 retry | 永遠救不回，浪費 cost |
| Retry 不附 error feedback | 大概率重複錯 |

### 考試判斷規則

| 題目線索 | 答案方向 |
|---|---|
| 加總不對 / 邏輯衝突 / 欄位互換 | Validation + retry with feedback |
| Source 內部矛盾 | `conflict_detected` boolean |
| 想偵測 stated vs calculated | `calculated_X` + `stated_X` 並存 |
| Dev dismissed findings 想分析 | `detected_pattern` 欄位 + aggregate |
| Retry 救不回 | Information absent from source |
| Retry prompt 要包什麼 | 三件套（原 doc + previous + specific error）|

### ★ 核心 mental model

> **Schema 守結構，validation+retry 守內容。Retry 不是 magic——只救「format/structural」，救不了「資料不存在」。三句記住設計：retry 帶三件套；資料缺別 retry；用 `calculated_X / stated_X / conflict_detected` 把驗證 hook 進 schema 讓 model 自己給對照值。**

---

## Task Statement 4.5 — Batch Processing Strategy

### 整個 task 在解決什麼問題

大量 LLM 工作沒人在等（夜間 audit、weekly report）—— Message Batches API 省 50% cost 換 24h 容忍。設計什麼工作該/不該用、SLA 數學、failure handling、跟 batch API 不支援的限制。

### 四個 Knowledge of 重點

1. **★★ Message Batches API 三特性**：50% cost off / 最多 24h window / **沒有 SLA 保證**
2. **★★ 適合 batch vs 不適合**：blocking workflow（pre-merge）→ sync；overnight/weekly → batch
3. **★★★ Batch 不支援 multi-turn tool calling**（agent loop 不能用 batch）
4. **★ `custom_id`** 對應 batch request/response（你設、API 原樣傳回）

### Batch + multi-turn 的真實行為（已釐清概念）

Batch **不會「拒絕」** multi-turn agent loop——它在第一個 `tool_use` 處「成功結束」把 partial result 還你。你只能：
- 接受 partial result（4.3 single-turn structured output 妙用 → 完美 work）
- 每 turn 串一個 batch（5-turn = 5 天）
- Migrate sync（正解）

**API 入口**：HTTP / Python SDK / TS SDK，**不是 CLI**。`claude` command 沒有 batch flag。
- POST `/v1/messages/batches` 建
- GET `/v1/messages/batches/{id}` 查 status
- GET `/v1/messages/batches/{id}/results` 取結果

### 四個 Skills in 重點

1. **★★ 按 latency 需求 match API**（pre-merge sync / overnight batch）
2. **★★ SLA 數學**：`max submission interval = SLA - 24h`（30h SLA → 每 4h 提交一次）
3. **★ Failure handling**：用 `custom_id` 找出失敗的，**只重送失敗的**（不是整 batch 重送），可加 chunking 等修正
4. **★ Sample 先 prompt-tune 再大規模 batch**（first-pass success rate 拉高才 full batch）

### 易踩陷阱

| 陷阱 | 為什麼錯 |
|---|---|
| Pre-merge check 改 batch | Dev 在等不能 24h |
| 「Batch + timeout fallback to sync」 | 雙倍 cost + 複雜度爆炸 |
| 「Batch ordering issue」當理由 | custom_id 解決，不是真理由 |
| 「換 batch 解 quality 問題」 | 同 model，batch 不會更小心 |
| Multi-turn agent loop 上 batch | 第一個 tool_use 就停下，沒辦法繼續 |

### 考試判斷規則

| 題目線索 | 答案 |
|---|---|
| Pre-merge / dev 在等 / blocking | Sync API（**禁** batch） |
| Overnight / weekly / 隔天看 | Batch API |
| 需要 multi-turn tool calling | ❌ Batch 不行 → sync |
| 業務 SLA = X 小時要算 batch interval | `interval = SLA - 24h` |
| Batch 失敗 | custom_id 找出 + 只重送 + 適當修改 |
| 大量 doc 跑前 | 先 sample tune prompt 再 full batch |

### Sample Question 11（Page 31-32）

> Pre-merge check + overnight tech debt report，manager 提議都改 batch。

**正解 A**：tech debt 改 batch（latency-tolerant）；pre-merge 保 sync（blocking）。
- B 「polling」對 blocking 沒用；C 「ordering issue」是錯誤理解；D「fallback to sync」雙倍 cost。

### ★ 核心 mental model

> **Batch API = 「等得起就便宜一半」。三件事決定能不能用：(1) 有沒有人在等？ (2) 是否需要 multi-turn tool calling？ (3) SLA 撐得過 24h 嗎？三題都對才上 batch。記住三數字：50% off / 24h window / no SLA。**

---

## Task Statement 4.6 — Multi-Pass / Multi-Instance Review Architecture

### 整個 task 在解決什麼問題

Single-pass review 太多 file → attention dilution（深度不一、漏 bug、矛盾）；同一 session 的 Claude self-review → 看不出自己的問題。**架構層**修——拆 review、起獨立 instance。

### 三個 Knowledge of 重點

1. **★★★ Self-review limitations**：同 session model 保留 generation 的 reasoning context，**不會質疑自己**。加 "review carefully" 沒用。
2. **★★ Independent review instances > self-review**：fresh context 才能挑 subtle bug；extended thinking / longer reasoning 同 session 也 anchored
3. **★★★ Multi-pass review**：per-file local + cross-file integration（兩個都要）

### 三個 Skills in 重點

1. **★★ Multi-instance**：第二次 `messages.create` call，**不要**包含第一個 session 的 messages history
2. **★★★ Per-file pass + integration pass**（Q12 直接考這條）—— 明確指示「不要做另一階段該做的事」
3. **★ Verification pass with self-reported confidence** → calibrated routing（短期方案；5.5 才是正規）

### Verification pass 的本質（已釐清概念）

**不是 4.4 validation**：

| | 4.4 validation+retry | 4.6 verification pass |
|---|---|---|
| 誰檢查 | 你 code（Pydantic）| **另一個 Claude call** |
| 失敗做什麼 | Retry **修正** | 加 metadata（**不修正**）|
| 目的 | 修錯 | 提供 routing signal |

### Confidence 三層觀（重要）

| 層次 | 立場 | 出現在 |
|---|---|---|
| 用 self-confidence 當**唯一 filter** | ❌ 反對 | 4.1, 5.2 |
| 用 self-confidence 當**routing signal**（多 input 之一）| ⚠️ 接受 | **4.6** |
| Calibrated confidence + labeled validation set | ✅ 推薦 | **5.5** |

→ **4.1 反對最濫的用法（單 hard filter）→ 4.6 接受中間用法（多 signal 之一）→ 5.5 推薦最嚴謹用法（labeled data 校準）**。

### Calibration 的真實機制（已釐清概念）

**Model 不會自己 calibrate**（連 prompt-based 都不算 CCAF 的 calibration——4.1 反對 "be conservative" 這類）。Calibration **在 model 外面做**：
- 收 sample → labeled ground truth
- 量 raw confidence vs actual accuracy → 建 mapping
- Production: 你 code 套 mapping 解讀 raw confidence
- Routing 用校準後的數字 + 配合其他 signal

**CCAF out-of-scope**：訓練 classifier、fine-tune model（PDF Page 38）。

### Prompt-based self-calibration 為何不算 calibration（已釐清概念）

你可能想：「叫 Claude 在 prompt 裡 'be conservative with confidence'、或餵過往 data 給它調整」——**技術上 model 行為會被影響，但 CCAF 不把這稱為 calibration**。四個原因：

| 原因 | 說明 |
|---|---|
| **① 4.1 明文反對** | "be conservative" 類 general instruction **就是 4.1 列為失敗的策略** |
| **② Model 沒 feedback loop** | Single inference 裡 model 看到 "0.8 實際 70%" 只是讀字，**沒真實機制驗證**這次 output 是否同樣 bias |
| **③ Model 沒 introspective access** | 你叫它「壓低 confidence」→ 它怎麼知道壓多少？壓 0.05 / 0.1 / 0.2 隨機，可能 over/under correct |
| **④ Probabilistic 修法不可量測** | 改 prompt 後沒辦法穩定收斂到目標 accuracy；外部 mapping 才能 measure → adjust → measure 循環 |

→ **CCAF 的 calibration 特指外部 code 用 labeled data 建 mapping**。考題看到「prompt 裡叫 model self-calibrate / be conservative with confidence」一律 distractor。

### Routing = Dispatching（已釐清概念）

兩者同義，CCAF 用 routing 這個詞。Calibration vs Routing 對比：

| 詞 | 意思 |
|---|---|
| **Routing / dispatching** | 把每個 finding 分流到不同 downstream（auto-post / human / drop / escalate）|
| **Calibrated** | 用真實 labeled data 校準的（threshold 不是拍腦袋）|
| **Calibrated routing** | 分流的 threshold 是用 labeled data 校準過的 |

→ 4.6 提供 raw signal（confidence），5.5 才教**怎麼校準**這個 signal。

### Sole filter 真正的反 pattern（已釐清概念）

不只是「沒 data 設 threshold」這麼窄。完整反 pattern = **三條湊在一起**：

| 條件 | 反 pattern 完整版 |
|---|---|
| ① 沒驗證 calibration | 不知 0.8 confidence 是不是真的 80% 正確 |
| ② 沒結合其他 signal | 純 confidence 一條過 |
| ③ 當 sole filter（hard cutoff）| `if conf > 0.9: show` |

→ **三條都犯 = CCAF 強烈反對**（4.1/5.2 distractor 主場）。
→ **即使有 data 校準，如果只用 confidence 當 sole filter 也不算 5.5 完整正解**——5.5 是「**校準 + 多 signal 結合**」雙重保險。

| 題目 distractor 出現 | 怎麼判 |
|---|---|
| "Self-rate confidence + threshold + only show > X" | ❌ 三條都犯 → distractor |
| "Self-rate confidence + use as one input alongside severity" | ⚠️ 中間（4.6 接受、不如 5.5）|
| "Field-level confidence calibrated using labeled validation set + stratified sampling" | ✅ 5.5 正解 |

### 易踩陷阱

| 陷阱 | 為什麼錯 |
|---|---|
| 換更大 context window 的 model | 鐵律 3 反模式（attention dilution 不是容量問題）|
| 叫 dev 把 PR 拆小 | Shift burden to user |
| Majority voting（跑 3 次 flag 重複的）| 真實 bug 可能 1/3 detect → 反而被壓制 |
| Extended thinking 解 self-review bias | 同 session 還是 anchored |
| Prompt 裡叫 model self-calibrate | 4.1 明文反對 "be conservative" 類 prompt |

### 考試判斷規則

| 題目線索 | 答案方向 |
|---|---|
| Multi-file review 深度不一 / 矛盾 / 漏 bug | Per-file local + cross-file integration pass |
| Claude 寫的 code 自己 review 看不出問題 | 第二個獨立 Claude instance（fresh context）|
| Cross-file bug | Integration pass |
| Local single-file bug | Per-file pass |
| Routing review attention by confidence | Verification pass + （長期）5.5 校準 |
| 「換大 model / 大 context window」 | ❌ 鐵律 3 |
| 「叫 dev 拆 PR / extended thinking / majority voting」 | ❌ Distractor |

### Sample Question 12（Page 32-33）

> 14 file PR，single-pass review 結果不一致（深度不一、漏 bug、矛盾 feedback）。

**正解 A**：拆 per-file local + cross-file integration pass。
- B「dev 拆 PR」shift burden；C「換大 context」鐵律 3；D「majority voting」反而壓制 detection。

### ★ 核心 mental model

> **單一 Claude 在單一 session 兩個天花板：(1) attention 會稀釋（看太多時）(2) 不會質疑自己。4.6 兩個架構手法分別針對：attention dilution → 拆 multi-pass；self-review bias → 起 multi-instance。看到「換大 model / 大 context / dev 拆 PR / extended thinking」當答案永遠是 distractor。**

---

# Domain 4 反模式速查表

| 反模式 | 為什麼錯 | 屬於 |
|---|---|---|
| Vague prompt 加更多形容詞（"VERY conservative"）| Vague 加多少都沒用 | 4.1 |
| Self-reported confidence threshold 當 sole filter | 沒校準 | 4.1, 5.2 |
| Severity level 沒給 concrete code example | Claude 自己腦補 anchor | 4.1 |
| Financial / 順序保證用 few-shot | 必須 deterministic → hook | 4.2 |
| Tool description 弱 + first step 用 few-shot | 該 expand description | 4.2 |
| 給 5-8 個 example 「越多越好」| 2-4 才對 | 4.2 |
| 「Schema 也解 semantic error」 | Schema 不檢查加總 | 4.3 |
| `tool_choice: "auto"` 期待保證 structured | 要 `"any"` 或 forced | 4.3 |
| Required field 強迫 + 期待別 fabricate | 必填強迫 model 編 | 4.3 |
| 盲目 retry 不附 error feedback | 大概率重複錯 | 4.4 |
| Information absent 還在 retry | 永遠救不回 | 4.4 |
| 「更嚴格 schema 解 sum mismatch」 | Schema 不檢查 sum | 4.4 |
| Pre-merge check 改 batch | Dev 在等不能 24h | 4.5 |
| Multi-turn agent loop 上 batch | 第一個 tool_use 就停 | 4.5 |
| 「Batch + fallback to sync」 | 雙倍 cost + 複雜度 | 4.5 |
| 換大 context window 解 attention dilution | 鐵律 3 | 4.6 |
| 叫 dev 把 PR 拆小 | Shift burden to user | 4.6 |
| Majority voting 解 review 不一致 | 反而壓制 detection | 4.6 |
| 同 session self-review + extended thinking | 還是 anchored | 4.6 |
| Prompt 裡叫 model self-calibrate confidence | 4.1 反對 "be conservative" 類 | 4.6 |

---

# 高頻必背 Sample Questions

| Sample Q | Domain | 考點 | 正解核心 |
|---|---|---|---|
| **Q11** | 4.5 | Batch vs sync 工作分配 | Tech debt batch / pre-merge sync |
| **Q12** | 4.6 | Multi-file review attention dilution | Per-file + cross-file integration pass |

（Q1-Q10 主要對應 Domain 1/2/3，但 Q1/Q2/Q3 中 few-shot 都當 distractor 出現過——4.2 的觀念點。）

---

# Domain 4 在「全 toolkit」框架的位置

```
L0 Prompt    : ★ 4.1 criteria, ★ 4.2 few-shot
L1 Schema    : ★ 4.3 tool_use + schema, nullable, tool_choice
L2 Validation: ★ 4.4 validate + retry + feedback
L3 Hook      : (Domain 1.5 — financial / 順序保證)
L4 Architecture: ★ 4.6 multi-pass / multi-instance
L5 Calibration: (Domain 5.5)
L6 Process   : ★ 4.5 batch / sync 選擇
```

→ Domain 4 涵蓋 L0/L1/L2/L4/L6——是「修 model output / 修 agent 行為」的核心工具集。

---

# 觀念 5 框架（四種改善 output 策略）

| 症狀 | 修法 | Domain |
|---|---|---|
| 看一眼就知道錯（format 飄、用詞飄）| Few-shot + prompt | 4.2 |
| 要算才知道錯（加總錯、欄位互換）| Validation + retry | 4.4 |
| 編造的錯（資訊不在 source）| Nullable schema + few-shot null | 4.3 + 4.2 |
| JSON broken（syntax error）| tool_use | 4.3 |

⚠️ CCAF 偏好 proportionate 解，不是「最強解」。`tool_use` 經常是 over-engineered distractor。

---

# CCAF 「明確 > 模糊」哲學在 Domain 4 的位置

| Task | 同一個哲學的不同 layer |
|---|---|
| **2.1 Tool descriptions** | 詳細 description > 模糊 description |
| **4.1 Prompt criteria** | Explicit categorical criteria > vague instructions |
| **5.2 Escalation triggers** | Explicit escalation criteria > sentiment / self-confidence |
| **1.4 Programmatic enforcement** | Hook 強制 > prompt 提示 |

→ **題目看到「靠 Claude 主觀判斷」vs「寫死規則」永遠選寫死的**。差別只在哪一層寫死。

---

# 6 組最容易混的對照（從 Output_Fix_Toolkit）

### ① Hook vs Validation+retry
- Hook（L3）：tool call **發生前**攔截，deterministic
- Validation（L2）：LLM **回完之後**檢查，probabilistic
- 100% 保證的事用 hook；可接受重試的 quality 用 validation

### ② Schema (tool_use) vs Validation+retry
- 4.3 解 syntax 錯（JSON 對不對）
- 4.4 解 semantic 錯（內容對不對）

### ③ Few-shot vs Hook（CCAF 高頻 distractor pair）
- Financial/compliance/順序 → 永遠 hook
- Quality 改善 → prompt-level 包括 few-shot

### ④ Tool description (2.1) vs Few-shot (4.2)
- Description 弱 + first step → expand description（few-shot 是 distractor）

### ⑤ Restrict allowedTools (2.3) vs PreToolUse hook (1.5)
- 靜態權限分配 → allowedTools
- 動態條件攔截 → hook

### ⑥ Self-reported confidence vs Calibrated confidence
- Self-reported（4.1/5.2 反對）vs Calibrated by labeled set（5.5 推薦）
- 4.6 是中間：當 routing signal（不是 sole filter）

---

# 三條鐵律（從 Output_Fix_Toolkit）

### 鐵律 1：「First step / proportionate / low-effort」 → 永遠選最便宜的
- 順序：L0 prompt → L1 schema → L2 retry → L3 hook → L4 architecture
- 答案幾乎在 L0 / L1

### 鐵律 2：Financial / compliance / safety / 順序 → 永遠 deterministic
- 「order / refund / financial / compliance / mandatory / must」字眼 → hook / programmatic prerequisite
- 此時 prompt-level（criteria / few-shot）一律 distractor

### 鐵律 3：「換大 model / 大 context window / 換 temperature」 → 永遠錯
- CCAF 反對 capacity 解架構問題

---

# 關鍵 Term 速記

| Term | 意思 |
|---|---|
| Explicit criteria | 寫死的 categorical inclusion/exclusion rule |
| Few-shot | 2-4 個 input/output example 示範 reasoning + format |
| `tool_use` block | Claude 回應的「請 call function」結構化 request |
| `input_schema` | tool 的 args 規格——Claude 給 client 的東西必須符合 |
| `tool_choice: "auto"` | Model 可選擇要不要 call tool（不保證） |
| `tool_choice: "any"` | Model 必 call tool 但自選 |
| `tool_choice: forced` | 強制特定 tool（`{"type":"tool","name":"X"}`）|
| Nullable schema | Field 設 `["string", "null"]` 防 fabrication |
| `"unclear"` enum | Ambiguous case 的 escape value |
| `"other"` + detail | Extensible categorization |
| Pydantic | Python schema validator 庫 |
| `stated_X` vs `calculated_X` | Self-correction 的對照欄位設計 |
| `conflict_detected` | Source 內部矛盾的 boolean flag |
| `detected_pattern` | 用來 aggregate dismissals 的 enum 欄位 |
| Retry with feedback 三件套 | 原 doc + previous extraction + specific error |
| Message Batches API | 50% off / 24h window / no SLA |
| `custom_id` | Batch request/response 的對應 label |
| Multi-pass review | Per-file local + cross-file integration |
| Multi-instance review | 第二個獨立 Claude，無 prior context |
| Verification pass | 第二次 call 對 findings 標 confidence（routing 用） |
| Calibrated confidence | Labeled data 量 mapping，外部 code 套（5.5）|

---

# 6 個 Task Statement 串起來看

```
4.1  Explicit criteria（取代 vague instruction）
 ↓
4.2  Few-shot（demonstrate target behavior）
 ↓
4.3  tool_use + schema（強制結構 / 防 fabrication）
 ↓
4.4  Validation + retry（守內容 / self-correction）
 ↓
4.5  Batch processing（cost vs latency tradeoff）
 ↓
4.6  Multi-pass / multi-instance（架構級修法）
```

**配對關係**：
- 4.1 + 4.2：criteria 寫規則，few-shot 示範邊界 case
- 4.2 + 4.3：few-shot 配 nullable schema 防 fabrication
- 4.3 + 4.4：4.3 守 syntax，4.4 守 semantic（先 4.3 再 4.4）
- 4.4 + 4.1：detected_pattern → aggregate FP → 回頭修 4.1 criteria
- 4.6 + 5.5：4.6 verification pass 是 short-term，5.5 calibration 是 long-term

---

# 終極 Mental Model

> **Domain 4 的核心邏輯：CCAF 偏好 proportionate / deterministic / 寫死規則的解法。修 model output 有四個層次（prompt → schema → validation → architecture），按便宜→貴選最簡有效的。Self-rating（confidence、conservative）一律可疑——要嘛當 signal（4.6 短期）、要嘛校準後當 metric（5.5 長期）、永遠不能當 sole filter。**
>
> **三件事決定考題答案：**
> **1. 這是 prompt-level 問題還是架構級？（避免 over-engineer）**
> **2. 這需要 deterministic guarantee 嗎？（financial → hook，不是 prompt）**
> **3. 這是 syntax 錯還是 semantic 錯？（schema vs validation）**
