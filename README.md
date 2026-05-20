# CCAF 學習資料夾 — 使用指南

這個 repo 是準備 **Claude Certified Architect — Foundations (CCAF)** 認證考試的全套資料.

包含三類檔案:
1. **官方原典** — Anthropic 出的 exam guide
2. **System prompt 檔** — 拿來餵新 Claude session, 讓它變成「教 CCAF 的老師」
3. **Summary / Toolkit 檔** — 我自己整理的筆記, 可以**直接讀** 或當 handoff context 餵新 session

---

## 📁 檔案結構

```
ccaf/
├── README.md                                          ← 你正在看的這份
├── Claude_Certified_Architect_–_Foundations_Certification_Exam_Guide.pdf ← 官方 40 頁原典                                                   
│
├── system_prompt_domain_1.md                          ← 教 Domain 1 的 system prompt
├── system_prompt_domain_2.md                          ← 教 Domain 2 的 system prompt
├── system_prompt_domain_3.md                          ← 教 Domain 3 的 system prompt
├── system_prompt_domain_4.md                          ← 教 Domain 4 的 system prompt
├── system_prompt_domain_5.md                          ← 教 Domain 5 的 system prompt
│
├── CCAF_Handoff_Context.md                            ← ★ 最重要的一份: 學習狀態 + 40 個釐清概念 + 練習階段判斷流程
├── CCAF_Output_Fix_Toolkit.md                         ← ★ 全工具光譜 (L0-L6) + 4-Branch + Context Loading 三維 + 練習心法
│
├── CCAF_Domain_1_Summary.md                           ← Domain 1 deep-dive (含 10 個釐清概念)
├── CCAF_Domain_2_Summary.md                           ← Domain 2 deep-dive (含 10 個釐清概念)
├── CCAF_Domain_3_Summary.md                           ← Domain 3 deep-dive (含 10 個釐清概念)
├── CCAF_Domain_4_Summary.md                           ← Domain 4 deep-dive (含 10 個釐清概念)
├── CCAF_Domain_5_Summary.md                           ← Domain 5 deep-dive (含 10 個釐清概念)

```

---

## 🎯 五大 Domain 佔比

| Domain | 佔比 | 主題 |
|---|---|---|
| **1** Agentic Architecture & Orchestration | **27%** | Coordinator / subagent / hooks / `Task` / `fork_session` |
| **2** Tool Design & MCP Integration | 18% | Tool description / `allowedTools` / `tool_choice` / MCP |
| **3** Claude Code Configuration & Workflows | 20% | CLAUDE.md / Skills / Rules / Slash commands / Plan mode |
| **4** Prompt Engineering & Structured Output | 20% | Criteria / few-shot / tool_use / validation+retry / batch |
| **5** Context Management & Reliability | 15% | Case facts / scratchpad / structured errors / calibrated confidence |

---

## 🚀 怎麼用這些檔案

### 用途 A: 從零開始學某個 Domain (用 system_prompt_domain_X.md)

這 5 個 `system_prompt_domain_*.md` 是**設計給新 Claude session 當 system prompt** 用的.

**步驟**:
1. 開一個新 Claude session (Devmate, Claude.ai, API 都行)
2. 把 `system_prompt_domain_X.md` 內容貼到 system prompt 欄位 (或對話開頭)
3. 跟新 session 說「請開始教我 Domain X」
4. 它會用設定好的風格教 (英文題目 + 中文解釋 + bullet point + ★ 標記)

**何時用**: 完全沒學過某 Domain, 想找個老師從頭講.

---

### 用途 B: 接續學習 / 練習階段 (用 CCAF_Handoff_Context.md)

`CCAF_Handoff_Context.md` 是**整個學習進度的 handoff 文件**, 包含:
- 已學完哪些 Domain
- **40 個個人化釐清概念** (從跟之前 session 學習過程中淬鍊出的)
- 個人學習偏好 (語言 / 風格 / 互動規則)
- **練習階段 5 步判斷流程**
- 已做過的 sample question 列表

**步驟**:
1. 開新 Claude session
2. 跟它說「請先讀 ~/ccaf/CCAF_Handoff_Context.md 再開始」 + (建議也加上 toolkit)
3. 進入 practice 模式, 直接貼 multiple-choice 題目

**何時用**: 已學完 5 個 Domain, 在做練習題 / mock test / 釐清觀念.

---

### 用途 C: 速查工具光譜 (用 CCAF_Output_Fix_Toolkit.md)

`CCAF_Output_Fix_Toolkit.md` 是**橫切式的速查表** (cross-cutting reference), 包含:
- 全工具按 L0-L6 layer 分 (prompt → schema → validation → hook → architecture → calibration)
- 19 組「容易混的對照」
- **15 步終極決策樹**
- **4-Branch 判斷法** (incorrect financial / scale / reasoning / root cause)
- **Context Loading 三維判斷** (Always / Path / Task)
- 3 條鐵律
- 50+ term 速記
- 40+ 反 pattern 清單
- Practice 階段精煉心法

**何時用**:
- 練習題遇到不知選哪個 → 看終極決策樹
- 看到熟悉但記不清的詞 → 翻 term 速記
- 看到答案說不上來為何錯 → 翻反 pattern 速記
- 模擬考前最後溫習 → 看三條鐵律 + Confidence 三層觀

---

### 用途 D: Domain 4 / 5 深入複習 (用 CCAF_Domain_4_Summary.md / CCAF_Domain_5_Summary.md)

每個 Summary 檔包含:
- 該 Domain 所有 task statement deep-dive
- 該 Domain 的 10 個 inline 釐清概念
- 官方原文 citation (page + bullet)
- ★ 高頻必背標記

**何時用**: 想針對某個 Domain 集中複習 (Domain 1-3 的 summary 在 `raw/`, 風格較早期).

---

## 💡 推薦的 onboarding 流程 (新 session)

如果你想複製這個學習狀態到新 Claude session, 貼下面這段給它:

```
我在準備 Anthropic CCAF 認證, 已經學完整 5 個 Domain.

請先依序讀這 3 個檔案:
1. ~/ccaf/CCAF_Handoff_Context.md  (學習狀態 + 40 個釐清概念 + 練習判斷流程)
2. ~/ccaf/CCAF_Output_Fix_Toolkit.md  (全工具光譜 + 4-Branch + Context 三維 + Practice 心法)
3. ~/ccaf/Claude_Certified_Architect_–_Foundations_Certification_Exam_Guide.pdf  (官方原典, 必要時 reference)

我在 PRACTICE 階段. 我會貴 multiple-choice 題目過來, 你的回答模式:

1. 直接告訴我答案 (A/B/C/D)
2. 為什麼對 + 為什麼其他選項錯 (簡短, bullet point)
3. 對應哪個 task statement / 哪個心法 (引用 toolkit 章節)
4. 不用重新展開教學 — 假設我已會基礎

語言規則:
- 題目英文 (CCAF 真實格式)
- 解釋中文
- 技術術語保留英文 (tool_use, tool_choice, allowedTools, PreToolUse, etc.)

風格規則:
- Bullet point 化, 不要散文
- 我搞錯時直接指出, 不要附和
- 簡短就好, 不要重新教學
```

---

## 📚 檔案分類速查表

| 想做什麼 | 用哪個檔 |
|---|---|
| 從零學 Domain X | `system_prompt_domain_X.md` (給新 session 當 system prompt) |
| 接續學習 / 練習題 | `CCAF_Handoff_Context.md` (給新 session 讀) |
| 速查工具該選哪個 | `CCAF_Output_Fix_Toolkit.md` (自己讀 / 給 session 讀) |
| Domain 4 / 5 deep-dive | `CCAF_Domain_4_Summary.md` / `CCAF_Domain_5_Summary.md` |
| Domain 1-3 deep-dive | `raw/CCAF_Domain_1_Summary.md` (較早期版本) |
| 對官方原文 cross-check | `Claude_Certified_Architect_–_Foundations_Certification_Exam_Guide.pdf` |
| 補充: Claude Code 架構 / Hook / Command-Agent binding | `raw/` 內的對應 `.md` |

---

## ⚠️ 注意

- `system_prompt_domain_*.md` 是**設計給 Claude 當 system prompt 讀的**, 你自己直接讀也可以但風格是 instruction 給 LLM, 不是給人類教學
- `CCAF_Handoff_Context.md` 跟 `CCAF_Output_Fix_Toolkit.md` 兩個是**人類自己讀也很有用的整理**, 同時也設計給新 session 讀
- Domain 1-3 的 summary 在 `raw/` 是因為當時整理風格較早期, 後來 Domain 4 / 5 的 summary 更精緻 (含 inline 釐清概念). 之後若有時間可以把 Domain 1-3 升級
- 所有 markdown 檔的 citation 格式: `> 官方原文 [Page X, Domain Y.Z, Knowledge of/Skills in, bullet N]: "..."`

---
