# CCAF Domain 3: Claude Code Configuration & Workflows

**佔比 20%**

整個 domain 聚焦在 **Claude Code 這個 CLI 產品**——怎麼設定 CLAUDE.md、怎麼建 slash command、怎麼用 plan mode、怎麼整合進 CI/CD。跟 Domain 1/2 偏寫 SDK/API code 不同，Domain 3 偏操作 Claude Code（設定檔、md 檔、CLI flag）。

---

## Task Statement 3.1 — CLAUDE.md Hierarchy, Scoping, Modular Organization

### 核心觀念

CLAUDE.md 是 Claude Code 的「全域 system prompt」——告訴 Claude 這個專案的規範、conventions、context。有三層 hierarchy，可以 modular 組織。

### CLAUDE.md 的三層 hierarchy

| 層級 | 位置 | 範圍 | 進 git |
|---|---|---|---|
| **User-level** | `~/.claude/CLAUDE.md` | 你個人，所有專案 | ❌ |
| **Project-level** | `.claude/CLAUDE.md` 或 root `CLAUDE.md` | 整個專案，團隊共用 | ✅ |
| **Directory-level** | 子目錄裡的 `CLAUDE.md` | 該目錄及其子目錄 | ✅ |

### 四個 Knowledge of 重點

1. **三層 hierarchy**（user / project / directory）
2. **User-level 不進 git** — 新團員 clone 後看不到（★ 高頻考點）
3. **`@import` 語法做 modular 引用** — 讓主檔不膨脹，各 package 選擇性 import
4. **`.claude/rules/` 取代 monolithic CLAUDE.md** — 拆成 topic-specific rule 檔

### 四個 Skills in 重點

1. **★ 診斷 hierarchy 配置問題**：新團員 clone 後 Claude 不遵守規範 → 檢查是不是放在 user-level
2. **`@import` 選擇性引用**：monorepo 不同 package import 不同 standard
3. **拆 monolithic CLAUDE.md**：用 `.claude/rules/` 按 topic 組織
4. **`/memory` 診斷載入問題**：用這個 slash command 看哪些 memory files 被載入

### CLAUDE.md 的實際注入機制（觀念補充）

CLAUDE.md **不直接放在 system prompt**。Claude Code 的 system prompt 對全球所有 user 一樣（共用 cache）。CLAUDE.md 透過 `<system-reminder>` XML tag 注入到 messages 區，但 effect 等同 system prompt（high-priority context）。

Cache 結構分層：
| 層 | 誰 share | Cache 行為 |
|---|---|---|
| Claude Code system prompt | 全球所有 user | 一次計算終身受用 |
| Tool definitions | 看你連什麼 MCP | 不變就 cache |
| CLAUDE.md | 同專案的所有人 | 不變就 cache |
| 對話歷史 | 你自己 session | 每 turn 滑動 breakpoint |

不要在 CLAUDE.md 放會變動的東西（時間、git status）— 會 break cache。

### 考試判斷規則

| 題目線索 | 答案方向 |
|---|---|
| 新團員 clone 後 Claude 沒遵守規範 | 檢查是不是放在 user-level → 移到 project-level |
| 規範都在一個大 CLAUDE.md，難維護 | 拆 `.claude/rules/` 或用 `@import` |
| Conventions 要套用到散在 codebase 的某類檔 | `.claude/rules/` + YAML `paths` glob（3.3）|
| Conventions 是某個目錄專屬的 | 子目錄 CLAUDE.md |
| 不知道哪些 file 被載入導致行為不一致 | `/memory` 診斷 |

---

## Task Statement 3.2 — Custom Slash Commands and Skills

### 兩個概念區分

| | Slash command | Skill |
|---|---|---|
| 觸發方式 | User 輸入 `/foo` | Claude 自動或被請求時 invoke |
| 本質 | Macro / shortcut | 能力包（workflow） |
| 檔案 | `.claude/commands/<name>.md` | `.claude/skills/<name>/SKILL.md` |
| 結構 | 單一 md 檔 | 一個目錄（SKILL.md + 其他資源） |
| 適合 | 人類常用的 macro | 特定 task workflow |
| 比喻 | Shell alias | Plugin / extension |

### 四個 Knowledge of 重點

1. **Slash command scope**：`.claude/commands/`（project，進 git）vs `~/.claude/commands/`（user，個人）
2. **Skill frontmatter**：`context: fork`、`allowed-tools`、`argument-hint`
3. **★ `context: fork`**：在獨立 sub-agent context 跑，防止 verbose output 污染 main conversation
4. **Personal skill customization**：在 `~/.claude/skills/` 用**不同 name** 建個人版本

### 五個 Skills in 重點

1. **Project-scoped slash commands in `.claude/commands/`** — 進 git，團隊共用
2. **`context: fork` 隔離 verbose output**（codebase analysis、brainstorming）
3. **`allowed-tools` 限制 skill 的 tool access**（principle of least privilege）
4. **`argument-hint` 提示參數**
5. **★ Skills vs CLAUDE.md 的選擇**：
   - Always-loaded universal standards → CLAUDE.md
   - On-demand task-specific workflow → Skill

### ★ Sample Question 4

題目：建 `/review` command 給團隊用，要求 clone/pull 後都能用。

**正解 = (A) `.claude/commands/` in project repository**

- (B) `~/.claude/commands/` — user-level 不進 git
- (C) CLAUDE.md — 是 context 不是 command
- (D) `.claude/config.json` commands array — **不存在**

### 容易踩的混淆

- **`context: fork` ≠ `fork_session`**：前者是 skill frontmatter（3.2），後者是程式呼叫（1.7）
- **Skill 的 `allowed-tools` ≠ AgentDefinition 的 `allowedTools`**：前者在 md frontmatter（Claude Code），後者在 SDK code

### 考試判斷規則

| 題目線索 | 答案方向 |
|---|---|
| 團隊共用的 slash command | `.claude/commands/`（project） |
| 我個人的 slash command | `~/.claude/commands/`（user） |
| Skill 產生大量 verbose output | `context: fork` |
| 想限制 skill 不要做 destructive 動作 | `allowed-tools` 限縮 |
| 永遠要遵守的 convention | CLAUDE.md（always-loaded） |
| 特定 task workflow | Skill（on-demand） |
| 想自訂跟團隊版不同的 skill | `~/.claude/skills/` 用不同 name |

---

## Task Statement 3.3 — Path-specific Rules for Conditional Convention Loading

### 核心觀念

`.claude/rules/` 的 md 檔可以加 YAML frontmatter `paths` 欄位，只在 Claude 處理符合 pattern 的檔時才注入。

### 三個 Knowledge of 重點

1. **`paths` glob pattern 做 conditional activation**：`paths: ["**/*.test.tsx"]` → 只有 test 檔才生效
2. **減少 irrelevant context 跟 token**：改 React component 時不需要看 Terraform rule
3. **★ Glob-pattern rules vs directory-level CLAUDE.md**：glob 可以抓散在各處的同類檔，directory CLAUDE.md 只能蓋一個目錄

### 三個 Skills in 重點

1. **建 path-scoped rules**：每個 rule 只在對應路徑生效
2. **用 glob 按檔案類型套用**：`**/*.test.tsx` 的 `**/` = 任何深度任何目錄
3. **★ Path-specific rules vs subdirectory CLAUDE.md 的選擇**

### ★ Sample Question 6

題目：不同 coding conventions（React/API/DB），test files 散在各處，要自動套用正確 conventions。

**正解 = (A) `.claude/rules/` with YAML frontmatter glob patterns**

- (B) Root CLAUDE.md 用 header 分區 — 靠 inference 不可靠
- (C) Skills 每個 code type 各一個 — 要手動 invoke，不 automatic
- (D) 每個子目錄各放 CLAUDE.md — directory-bound，沒法處理散在各處的檔

### 3.1 / 3.2 / 3.3 三個機制並存

```
                Always loaded?    Path conditional?    User invoked?
CLAUDE.md          ✅                 ❌                 ❌
.claude/rules/     ❌                 ✅                 ❌
Skills             ❌                 ❌                 ✅
```

### 注入時機差別

| 機制 | 注入時機 | 注入方式 |
|---|---|---|
| CLAUDE.md | Conversation 開頭一次 | `<system-reminder>` in messages |
| `.claude/rules/` | 每次 Claude 存取符合 path 的檔時 | `<system-reminder>` 動態 re-inject |
| Skill | User invoke 或 Claude 主動決定 invoke 時 | 看 `context: fork` 設定 |

### Decision Tree：Convention 該放在哪

```
Q: 這個 convention...

  是每次都要遵守的？
  ├─ Yes → CLAUDE.md（always-loaded）
  └─ No → 繼續

  是特定類型檔案才要的？
  ├─ Yes → .claude/rules/ + paths glob
  └─ No → 繼續

  是做特定 task 才需要的 workflow？
  ├─ Yes → Skill
```

### 考試判斷規則

| 題目線索 | 答案方向 |
|---|---|
| Convention 針對某類檔案（散在各處） | `.claude/rules/` + glob |
| Convention 要自動套用 | ❌ Skill（要 invoke，不 automatic） |
| Convention 要 deterministic 不靠推測 | ❌ Root CLAUDE.md 用 header 分區 |
| Convention 針對某個目錄下所有檔 | 子目錄 CLAUDE.md 或 `.claude/rules/` |
| Test files spread throughout the codebase | `.claude/rules/` + `**/*.test.*` |

---

## Task Statement 3.4 — Plan Mode vs Direct Execution

### 核心觀念

什麼時候先想再做（plan mode），什麼時候直接做（direct execution）。

### 四個 Knowledge of 重點

1. **Plan mode 適合**：large-scale changes、multiple valid approaches、architectural decisions、multi-file modifications
2. **Direct execution 適合**：simple、well-scoped changes（single-file fix、加一個 validation）
3. **Plan mode 的核心價值**：避免 costly rework（先探索再動手 → 發現問題早）
4. **Explore subagent**：Anthropic 官方內建的 subagent，專做 exploration，在獨立 context 跑，只回 summary

### 四個 Skills in 重點

1. **選 plan mode 的場景**：microservice restructuring、library migration（45+ files）、比較 integration approaches
2. **選 direct execution 的場景**：single-file bug fix with clear stack trace、加 date validation
3. **Explore subagent 用於 verbose discovery**
4. **★ 組合使用**：plan mode for investigation → direct execution for implementation

### ★ Sample Question 5

題目：把 monolithic app 拆成 microservices，跨數十個檔案，要做 service boundary 決策。

**正解 = (A) Plan mode**

- (B) Direct execution + incremental — 做到一半才發現依賴問題 → costly rework
- (C) Direct execution + comprehensive upfront instructions — 假設你已知答案
- (D) Direct execution，遇到問題才切 plan — complexity 已 stated，一開始就該 plan

### 判斷的兩個維度

| | 知道怎麼改 | 不知道怎麼改 |
|---|---|---|
| **小範圍** | Direct execution ✅ | 可能 plan 但通常 direct |
| **大範圍** | Direct execution（按 plan 執行） | Plan mode ✅ |

### 易踩陷阱

**「先 direct 再看情況切 plan」幾乎都是錯的** — 題目已經寫明 complexity，你一開始就知道要 plan。

### 考試判斷規則

| 題目線索 | 答案 |
|---|---|
| Microservice restructuring / library migration / architectural decision | **Plan mode** |
| Multiple valid approaches / choose between X and Y | **Plan mode** |
| Affecting dozens / 45+ files | **Plan mode** |
| Single-file bug fix with clear stack trace | **Direct execution** |
| Adding a validation check to one function | **Direct execution** |
| First plan, then execute | **兩者組合使用** |
| Context window exhaustion during exploration | **Explore subagent** |

---

## Task Statement 3.5 — Iterative Refinement Techniques

### 核心觀念

Direct execution 階段怎麼做迭代優化——邊做邊測邊調整，不是全部做完才驗證。

### 四個 Knowledge of 重點

1. **Iterative refinement 在 execution 階段**：基於 intermediate outputs 的 feedback 改進
2. **★ Mid-execution validation and adaptation**：例如遷移第 8 個檔時發現依賴問題 → 調整策略繼續
3. **Mid-phase validation 避免 rework**：早發現問題比全做完才發現好
4. **Test-driven validation**：把 test 當「進度檢查點」，每個 stage 都驗證

### 四個 Skills in 重點

1. **Break execution into validateable stages**：寫 endpoint → 測 → 更新 caller → 測 → ...
2. **Test-driven validation checkpoints**：每個節點都有驗證
3. **Mid-phase issue detection and strategy adjustment**：調整策略不 revert，繼續做
4. **Incremental testing for confidence building**：通過每個 checkpoint 增加信心

### 發現問題後的反應

| 反應 | 成本 | 何時用 |
|---|---|---|
| Rework（全部 revert 重做） | 高 | ❌ 不該這樣 |
| Local adjustment（調整當前 stage） | 低 | ✅ 通常用 |
| Plan revision（改後面的 plan） | 中 | ✅ 有時用 |

### 考試判斷規則

| 題目線索 | 答案方向 |
|---|---|
| Plan 後發現問題 | 調整策略繼續，不 revert |
| 執行到一半測試失敗 | Mid-phase validation：調整 + 繼續 |
| 應該什麼時候驗證 | 每個 stage（不是最後） |
| Stage 該有多大 | 可驗證的單位（通常 1-5 個檔案） |

---

## Task Statement 3.6 — CI/CD Pipeline Integration

### 核心觀念

用 CLI flag 把 Claude Code 整合進 CI/CD pipeline，無人值守地跑。

### 四個 Knowledge of 重點

1. **CI/CD 用途**：automated code review、test analysis、structured output generation
2. **★ CLI flags**：`--plan`、`--output-format json`、`--json-schema`
3. **`--output-format json` vs `--json-schema` 的區別**（很容易混）
4. **Deterministic CI/CD** 需要 `--plan` + schema validation

### `--output-format json` vs `--json-schema` 區別

| | `--output-format json` | `--json-schema <file>` |
|---|---|---|
| 做什麼 | Wrap stdout 在 JSON envelope | 強制 Claude output 內容符合 schema |
| 層級 | Wrapping layer（CLI 層）| Content layer（Claude 本身）|
| 用途 | CI 系統讀 metadata（cost、session_id）| downstream 工具直接讀結構化 output |
| 對應 Domain | 3.6（CLI config）| 4.3（structured output）|

### 四個 Skills in 重點

1. **`--plan` for reviewable execution**：先產 plan → human review → 再 execute
2. **`--output-format json` for CI parsing**：讓 CI 讀 metadata
3. **★ `--json-schema` for guaranteed structure**：downstream 工具可以直接 access field
4. **CI/CD pipeline 設計模式**：plan → review → execute

### 三個 CLI flag 對照

| Flag | 用途 | 對應概念 |
|---|---|---|
| `--plan` | 先規劃後執行 | 3.4 plan mode |
| `--output-format json` | wrap stdout 在 JSON envelope + metadata | CI 系統讀取 |
| `--json-schema <file>` | 強制 output 內容符合 schema | Guaranteed structure |

三個可以組合用：
```bash
claude analyze-tests \
  --plan \
  --output-format json \
  --json-schema test-report.json
```

### 考試判斷規則

| 題目線索 | 答案方向 |
|---|---|
| CI/CD 要 deterministic | `--plan` flag |
| Downstream 工具要 parse output | `--output-format json` + `--json-schema` |
| Code review 要先規劃 | `--plan` |
| Output 要符合特定 structure | `--json-schema` |
| Output 要含 metadata（cost、duration）| `--output-format json` |
| Human 要看計劃後決定 | `--plan` |

---

# Domain 3 反模式速查表

| 反模式 | 為什麼錯 | 屬於 |
|---|---|---|
| 規範放 user-level（`~/.claude/CLAUDE.md`）期望團隊共用 | 不進 git，新人看不到 | 3.1 |
| 所有 conventions 塞一個大 CLAUDE.md | 難維護、irrelevant rules 佔 context | 3.1 |
| 用 root CLAUDE.md header 分區靠推測套用 | Inference 不可靠 | 3.3 |
| 用 Skill 做「自動套用 convention」 | Skill 要 invoke，不 automatic | 3.3 |
| 用子目錄 CLAUDE.md 管散在各處的同類檔 | Directory-bound，不可維護 | 3.3 |
| Complex task 直接動手不先 plan | Costly rework | 3.4 |
| 「先 direct 再看情況切 plan」 | Complexity 已 stated 就該直接 plan | 3.4 |
| 全部做完才 test | 問題發現太晚，rework 成本高 | 3.5 |
| 發現問題全部 revert 重做 | 浪費已做的工作 | 3.5 |
| CI/CD 只用 `--output-format json` 不加 schema | Output 結構沒保證 | 3.6 |
| CI/CD 文檔寫 expected format 但不 enforce | 文檔不等於保證 | 3.6 |

---

# 高頻必背 Sample Questions

| Sample Q | 考點 | 正解核心 |
|---|---|---|
| **Q4** | 3.2 — Slash command scope | `.claude/commands/` in project repo |
| **Q5** | 3.4 — Plan mode for complex task | Plan mode |
| **Q6** | 3.1 + 3.3 — Path-scoped conventions | `.claude/rules/` with glob patterns |

---

# 關鍵 Term 速記

| Term | 意思 |
|---|---|
| `CLAUDE.md` | 專案 system prompt context（三層 hierarchy） |
| `@import` | CLAUDE.md 裡引用其他 md 檔的語法 |
| `.claude/rules/` | Topic-specific rule 檔，可加 `paths` glob |
| `paths` frontmatter | YAML glob pattern 做 conditional rule activation |
| `.claude/commands/` | 自訂 slash commands |
| `.claude/skills/<name>/SKILL.md` | 自訂 skills |
| `context: fork` | Skill frontmatter，在獨立 sub-agent context 跑 |
| `allowed-tools` | Skill frontmatter，限縮 tool access |
| `argument-hint` | Skill frontmatter，提示 user 給參數 |
| `/memory` | Claude Code 內建 command，檢查載入了哪些 memory files |
| `<system-reminder>` | Claude Code 注入 CLAUDE.md / rules 的 XML tag |
| Explore subagent | 官方內建的探索用 subagent |
| `--plan` | CLI flag，先規劃後執行 |
| `--output-format json` | CLI flag，wrap stdout 在 JSON envelope |
| `--json-schema` | CLI flag，強制 output 內容符合 schema |

---

# 設定檔位置完整對照表

| 設定類型 | User-level | Project-level |
|---|---|---|
| **MCP server 設定** | `~/.claude.json` | `.mcp.json` |
| **CLAUDE.md** | `~/.claude/CLAUDE.md` | `.claude/CLAUDE.md` 或 root `CLAUDE.md` |
| **Slash commands** | `~/.claude/commands/` | `.claude/commands/` |
| **Skills** | `~/.claude/skills/` | `.claude/skills/` |
| **Path-scoped rules** | （沒有 user-level） | `.claude/rules/` |

注意命名混淆：
- `~/.claude.json` = MCP 設定檔（JSON）
- `~/.claude/CLAUDE.md` = 個人 prompt 規範（Markdown）
- 一個是 `.json` 檔，一個是 `.claude/` 目錄底下的 `CLAUDE.md`

---

# 六個 Task Statement 串起來看

```
3.1  CLAUDE.md hierarchy（always-loaded universal context）
 ↓
3.2  Slash commands + Skills（user-triggered macros vs on-demand workflows）
 ↓
3.3  Path-specific rules（auto conditional loading by file type）
 ↓
3.4  Plan mode vs direct execution（complex task 先規劃）
 ↓
3.5  Iterative refinement（execution 時邊驗證邊調整）
 ↓
3.6  CI/CD integration（用 CLI flag 自動化）
```

---

# 終極 Mental Model

> **Domain 3 的核心邏輯：Claude Code 有三層 context 注入（CLAUDE.md always-on / rules conditional / skills on-demand），兩種執行模式（plan 先想 / direct 直接做），一套 CI/CD flag（讓它無人值守也能跑出結構化結果）。**
>
> **選擇機制時看三件事：**
> **1. 這個 convention 要不要永遠生效？（CLAUDE.md vs rules vs skill）**
> **2. 這個 task 要不要先探索？（plan vs direct）**
> **3. 這個 output 要不要被機器讀？（`--json-schema`）**
