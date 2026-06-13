## 一、問題陳述

Airflow 目前的 agent 指引架構：

- `AGENTS.md`（468 行，`CLAUDE.md` 是指向它的 symlink）把所有規則塞在一個檔案裡
- Agent session 開始時全部載入，改 UI 的人也被灌了 Helm test 的指令
- 長 session 後 agent 忘記早期規則（long-context degradation）
- 沒有任何方式驗證 agent 是否真的遵守了規則

根本原因：三種不同性質的東西混在一起了。

| 類型                           | 例子                                                                | 更新頻率 | 載入時機   |
| ------------------------------ | ------------------------------------------------------------------- | -------- | ---------- |
| **知識**（Knowledge）          | 「worker 不能直接存取 DB」「架構邊界」                              | 低       | 永遠需要   |
| **決策規則**（Decision rules） | 「db_test 就用 breeze」「改 provider 用 `--project providers/xxx`」 | 中       | 取決於任務 |
| **操作程序**（Procedures）     | 「開 PR 的 12 個步驟」「跑 selective checks 的流程」                | 高       | 按需       |

兩個獨立的 failure mode：

1. **Long-context degradation**：skill 在 session 開始時載入，但被後續 context 埋掉，agent 忘記規則
2. **Discovery failure**：agent 根本沒去讀 skill（~48% 讀取率），決策時缺少指引

這兩個 failure mode 的機制不同、介入方式不同、eval 方式也必須不同。

### 這個問題有多嚴重？相關研究

| 研究                                                   | 核心發現                                                                         | 對本設計的啟示                                                |
| ------------------------------------------------------ | -------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| **Lost in the Middle** (Liu et al., 2023)              | LLM 對 context 中間位置的資訊利用呈 **U 形曲線**——開頭和結尾記得住，中間大量丟失 | Skill 裡的規則位置很重要——關鍵規則不能埋在中間                |
| **Context Rot** (Chroma, 2025)                         | 測了 18 個前沿模型，**全部**隨 context 長度退化。根因是 RoPE 位置編碼的長期衰減  | 光靠「寫得更好」不夠，需要結構性介入（reload）                |
| **Context Length Alone Hurts** (Du et al., EMNLP 2025) | 即使完美檢索，僅長度增加就導致 **13.9%-85%** 性能下降                            | Skill 必須模組化——載入不需要的 context 本身就在造成傷害       |
| **SlopCodeBench** (2025.3)                             | 77% 軌跡出現結構侵蝕。**質量指導可減少初始問題但不改變退化速率**                 | 更好的 skill 只能改善起點，不能阻止衰退。需要主動 reload 機制 |
| **OctoBench** (2026.1)                                 | 把「任務完成」和「規則遵循」解耦評估。兩者之間存在**系統性差距**                 | Eval 必須分開量測「code 對不對」和「workflow 對不對」         |
| **BABILong** (NeurIPS 2024)                            | LLM 僅能有效利用 **10-20%** 的 context                                           | 468 行的 AGENTS.md 可能只有 ~50-90 行真正被利用               |

這些研究指向兩個結論：

1. **被動防禦（寫更好的 skill）只能改善起點，不能阻止衰退**——SlopCodeBench 明確證明了這一點
2. **主動介入（在衰退拐點 reload skill）是必要的**——衰退曲線的拐點就是 reload 時機，但必須有實驗數據驗證 reload 確實有效

## 二、與現有框架的關係

### Apache Magpie（airflow-steward）

管的是**維護者側**——PR triage、code review、security workflow。透過 snapshot + lock file 從上游分發，skill 帶 `magpie-` 前綴。

**注意**：Magpie 的 roadmap 已包含 Pairing（developer-side dev-cycle）和 Mentoring（contributor guidance），正在往貢獻者側擴展。本設計與 Magpie 的邊界需要在實作前與 steward 的維護者（Jarek）對齊，不能當成既定事實寫死。

### 本設計

管的是**貢獻者側**——agent 在幫貢獻者寫 code 時，有沒有選對指令、遵守架構邊界、正確跑測試。Skill **不帶** `magpie-` 前綴，是 Airflow 自有的、committed 的。

### 目錄結構

Airflow 目前的 skill 目錄結構（截至 2026-06-07，commit `942de38a35`）：

```
.agents/skills/                        ← canonical home
├── magpie-setup/                      ← Magpie（committed bootstrap）
├── magpie-pr-management-*             ← Magpie（gitignored symlink → snapshot）
├── magpie-security-*                  ← Magpie（gitignored symlink → snapshot）
├── aip-user-stories/                  ← Airflow 自有
├── airflow-translations/              ← Airflow 自有
└── prepare-providers-documentation/   ← Airflow 自有

.claude/skills/                        ← Claude Code relay symlink
.github/skills/                        ← GitHub skill loader relay symlink
```

本設計新增的 skill 放在 canonical home：

```
.agents/skills/
├── ...（現有）
├── airflow-test/       ← 本設計（committed，Airflow 自有）
├── airflow-pr/         ← 本設計
├── airflow-provider/   ← 本設計
└── airflow-ui/         ← 本設計
```

需要同步建立 `.claude/skills/` 和 `.github/skills/` 的 relay symlink，遵循 Magpie 的 canonical + relay 模型。

## 三、分層架構

```
Layer 0: Source of Truth（人類維護）
│   contributing-docs/*.rst        ← 指令的權威來源
│   airflow-core/docs/             ← 架構文件
│   AGENTS.md（CLAUDE.md → symlink）← 精簡後只留 judgment rules
├──→ prek hook（自動注入）
Layer 1: Skills（生成 + 按需載入）
│   .agents/skills/airflow-*/SKILL.md
├──→ CI diff check（漂移偵測）
Layer 2: Eval（驗證 agent 行為）
│   dev/skill-evals/
│   ├── static/      ← Level A: 靜態結構檢查
│   ├── pressure/    ← Level B: context-pressure eval（主力）
│   ├── discovery/   ← Level D: skill discovery eval
│   └── replay/      ← Level C: real session replay
├──→ CI 自動跑（on skill/docs change）
Layer 3: Benchmark Report（彙總）
│   衰退曲線（含 baseline arm + reload arm）
│   discovery rate / pass^k reliability metrics
```

## 四、Layer 0 → Layer 1：Skill 生成機制

### 4.1 AGENTS.md 瘦身

| 留在 AGENTS.md 的（~100 行）       | 移入 SKILL.md 的     |
| ---------------------------------- | -------------------- |
| Naming 慣例（Dag vs DAG）          | Commands 完整列表    |
| Architecture boundaries            | 測試 workflow 決策樹 |
| Security model 摘要                | PR 建立完整流程      |
| Coding standards（judgment rules） | Provider 開發流程    |
| Commit message 規範                | Helm test 流程       |

原則：**AGENTS.md 只留「不管做什麼任務都需要知道」的 judgment rules。**

### 4.2 生成方式：模板 + 注入

在 `contributing-docs` 裡加 marker，人工維護 `SKILL.template.md`（workflow 邏輯），prek hook 把 marker 內容注入 template，產出最終的 `SKILL.md`。

### 4.3 CI 漂移偵測

觸發：`contributing-docs/**/*.rst` 或 `.agents/skills/**/SKILL.template.md` 有變更時，prek hook 重新生成 SKILL.md，diff vs committed 版本，有差異就 fail。

## 五、Layer 2：Eval 設計

### 5.1 四層 Eval 設計

| 層級                    | 測什麼                        | 怎麼測                              | 需要 LLM？ | 成本     |
| ----------------------- | ----------------------------- | ----------------------------------- | ---------- | -------- |
| **A: Clarity Check**    | skill 文件結構品質            | 靜態 linter                         | 不需要     | 秒級     |
| **B: Context-Pressure** | skill 在長 session 下的存活率 | 三臂實驗：skill / no-skill / reload | 需要       | 分鐘級   |
| **D: Discovery**        | agent 有沒有去讀 skill        | skill 以檔案存在但不塞進 context    | 需要       | 分鐘級   |
| **C: Session Replay**   | 完整 workflow 正確性          | 真實 PR revert + 隔離 checkout      | 需要       | 十分鐘級 |

### 5.2 Level A：Skill Clarity Check（靜態，不用 LLM）

純靜態檢查：決策規則必須在文件前 1/3、必須是 if/then 結構、inject marker 必須解析完畢、YAML frontmatter 必須完整。

### 5.3 Level B：Context-Pressure Eval（主力）

#### 為什麼需要三臂實驗

只有一條衰退曲線（有 skill）是不夠的。「skill 在 15k token 後開始失效」——這個下降可能是模型本身在長 context 下的退化，跟 skill 無關。沒有對照線，你分不出這兩者。

**必須有三條線：**

| Arm                   | 設定                                       | 量什麼                          |
| --------------------- | ------------------------------------------ | ------------------------------- |
| **Skill arm**         | skill 在 turn 1 載入，後續正常干擾         | 有 skill 時的衰退曲線           |
| **No-skill baseline** | 同樣的干擾 context，但不載入 skill         | 模型本身的 baseline（歸因控制） |
| **Reload arm**        | skill 在 turn 1 載入，在拐點處重新注入一次 | reload 是否有效                 |

**兩條線的差值 = skill 可歸因的價值。** reload arm 把「建議在 turn 10 reload」從猜測變成數據。

#### Skill 的放置位置：turn 1 user message，不是 system

把 skill 放在 `system` message 是錯的。真實情況是 skill 在 turn 1 被讀進 context，然後被後續對話埋掉（lost-in-the-middle）。`system` prompt 被模型特權對待，衰退較慢。放 `system` 會**低估**真實衰退。

#### 衰退曲線的產出

每壓力級跑 k=10 次，報告 pass@1（平均）和 pass^k（10 次全對）。要求 LLM 回傳 structured JSON `{"tool": "breeze", "project": "airflow-core"}`，解析欄位判定，避免子字串比對的脆弱性。

三條線各自回答的問題：

| 比較                           | 回答的問題                                     |
| ------------------------------ | ---------------------------------------------- |
| skill arm vs no-skill baseline | **Skill 有沒有用？** 差值 = skill 可歸因的提升 |
| skill arm 的衰退斜率           | **Skill 多快失效？** 拐點在哪                  |
| reload arm vs skill arm        | **Reload 有沒有效？** 曲線有沒有回升           |
| no-skill baseline 的斜率       | **模型本身退化多少？** 歸因控制變數            |

#### 干擾 context 的來源

初期策略：**合成 + 少量真實混合**，迭代替換。合成語料從 Airflow repo 取真實原始碼、CI log、人工構造 debug 對話。真實 session transcript 需要 consent、匿名化、格式化，工程量大，後續逐步替換。

### 5.4 Level D：Discovery Eval

Level B 假設 skill 已在 context 裡。但真實情況下 skill 是一個**檔案**，agent 必須主動去讀它。

Level D 測的是：**skill 以檔案形式存在，agent 在決策前有沒有去讀它？**

|                  | Level B                  | Level D                        |
| ---------------- | ------------------------ | ------------------------------ |
| **Skill 在哪？** | 已在 context 裡          | 以檔案存在，未載入             |
| **測什麼？**     | Skill 被埋後還記不記得   | Agent 會不會主動去讀           |
| **Failure mode** | Long-context degradation | Discovery failure              |
| **介入方式**     | Reload skill             | 改善 frontmatter / 位置 / 命名 |

### 5.5 Level C：Real Session Replay（少量，隔離環境）

用真實 merged PR 的 revert 狀態，讓 agent 從原始 issue description 出發，跑完整 workflow。

**隔離要求**：Level C 必須在**不含 `dev/skill-evals/` 的隔離 checkout** 中跑。Fixture 裡有 prompt 和預期行為，agent 在同一 checkout 會讀到答案，造成外洩。

Level C **不在 CI 跑**。太慢、太貴、太 flaky。只在週末 cron 或手動觸發。

## 六、Layer 3：CI 整合與 Benchmark Report

### 6.1 目錄結構

```
dev/skill-evals/
├── conftest.py           ← 共用 fixture（llm_client, eval_logger, pass^k）
├── scoring.py            ← 三臂衰退曲線生成
├── static/               ← Level A（秒級，每次 CI 必跑）
├── pressure/             ← Level B（分鐘級，skill 變更時跑）
│   ├── test_decay_curve.py  ← 三臂實驗
│   ├── fixtures/            ← 干擾語料庫
│   └── report.py            ← 報告生成器
├── discovery/            ← Level D（分鐘級）
└── replay/               ← Level C（十分鐘級，週級，隔離 checkout）
    └── fixtures/
```

### 6.2 觸發條件摘要

| 事件                              | Level A | Level B | Level D | Level C |
| --------------------------------- | ------- | ------- | ------- | ------- |
| PR 改了 contributing-docs         | 跑      | 跑      | 不跑    | 不跑    |
| PR 改了 .agents/skills/airflow-\* | 跑      | 跑      | 跑      | 不跑    |
| PR 改了 AGENTS.md                 | 跑      | 跑      | 不跑    | 不跑    |
| 每週日凌晨 cron                   | 跑      | 跑      | 跑      | 跑      |
| 手動觸發                          | 跑      | 跑      | 跑      | 跑      |

## 七、Community Failure Case 收集

### 7.1 PR Template 新增欄位

```
## AI Agent Experience (optional)
- Tool used:
- What went wrong (if anything):
- Command agent used incorrectly:
- Correct command:
- Approximate session length:
```

### 7.2 收集流程

1. 貢獻者提 PR，填寫 AI Agent Experience
2. 標記 `kind:agent-case-study` label
3. GitHub Action 定期彙總到 `dev/agent-cases/YYYY-MM.md`
4. 維護者定期 review，發現 pattern
5. pattern → 改善 SKILL.md 或 AGENTS.md
6. 嚴重的 pattern → 轉成新的 Level B eval scenario

## 八、Feedback Loop：完整迴路

**迴路 1：contributing-docs 變更**

1. prek hook 重新生成 SKILL.md
2. Level A：結構 OK？
3. Level B（三臂）：skill lift 有沒有惡化？reload recovery 有效嗎？
4. 惡化 → 改 SKILL.template.md（規則位置、結構）→ 回到步驟 1
5. 改善或持平 → merge

**迴路 2：社群回報 failure case**

1. 歸檔到 dev/agent-cases/
2. 辨識 pattern：這是 degradation（Level B）還是 discovery（Level D）？
3. 轉成對應的 eval scenario
4. 改善 SKILL.md → 回到迴路 1

**三臂衰退曲線是 feedback loop 的核心度量。** 每次改 skill 都可以比較：skill lift 有沒有增加、拐點有沒有右移、reload recovery 是否有效。

## 九、交付里程碑

| 週次    | Deliverable                                                                                       |
| ------- | ------------------------------------------------------------------------------------------------- |
| Week 2  | SKILL.template.md（airflow-test）、prek hook、CI drift check                                      |
| Week 4  | Level A static linter + Level B eval framework（三臂、structured output、pass^k）+ 合成干擾語料庫 |
| Week 6  | Level B 三臂 eval 完整覆蓋 + Level D discovery eval                                               |
| Week 8  | Community failure case 收集機制 + 開始收集真實干擾語料                                            |
| Week 10 | AGENTS.md 瘦身 + 第二批 skill + 對應 Level A/B/D eval                                             |
| Week 12 | Level C replay fixtures（隔離環境）、三臂 benchmark report + 基線數據                             |

## 十、相關基準與工具

### 10.1 直接相關的 Agent 長 context 基準

| 基準                             | 測什麼                             | 本設計借鑑了什麼                             |
| -------------------------------- | ---------------------------------- | -------------------------------------------- |
| **SlopCodeBench** (2025.3)       | 編碼 Agent 迭代開發中的退化        | 衰退曲線設計——量測退化速率而非只看 pass/fail |
| **OctoBench** (2026.1)           | 「任務完成」vs「規則遵循」解耦     | Eval 核心框架——測規則遵循，不是 code 品質    |
| **LoCoBench-Agent** (Salesforce) | SE Agent 在 10K-1M tokens 下的表現 | 可控 context 長度 + 工具使用評估             |
| **ProcCtrlBench** (2026)         | 11 類 agent 執行缺陷               | 缺陷分類法——用於歸類 failure case            |
| **LOCA-bench** (2026.2)          | 可控 context 增長下的 agent 評估   | 「可控增長」= 我們的 PRESSURE_LEVELS 設計    |

### 10.2 可用的開源工具

- Chroma Context Rot Toolkit — 測量模型 context 衰退曲線
- LoCoBench-Agent — SE Agent 長 context 評估框架
- Promptfoo — 開源 prompt 測試 CLI

## 十一、設計決策摘要

| 決策                            | 為什麼                                           | 支撐證據                                       |
| ------------------------------- | ------------------------------------------------ | ---------------------------------------------- |
| **三臂實驗**                    | 沒有 baseline 就無法歸因 skill 的價值            | 實驗設計基本原則；SlopCodeBench                |
| **Skill 放 turn 1 而非 system** | system 被特權對待，低估衰退                      | Lost in the Middle                             |
| **Level D 獨立測 discovery**    | degradation 和 discovery 是兩個不同 failure mode | Level B 完全繞過 discovery                     |
| **Structured JSON output**      | 子字串比對脆弱（"use breeze, not uv" 誤判）      | —                                              |
| **pass^k（k=10）**              | 單次結果不可靠                                   | —                                              |
| **Level C 隔離 checkout**       | 防止答案外洩                                     | —                                              |
| **與 Magpie 邊界需先對齊**      | Magpie 正在往貢獻者側擴展                        | 需與 steward 維護者確認                        |
| **模組化 skill 是必要的**       | 載入不需要的 context 本身就有害                  | Context Length Alone Hurts：13.9%-85% 性能下降 |
| **衰退曲線數字標為 TBD**        | 未跑 eval 前沒有實測數據                         | —                                              |

## 十二、References

### 基礎研究

- Liu, N. F. et al. (2023). "Lost in the Middle." _TACL_.
- Chroma Research (2025). "Context Rot." link
- Du, Y. et al. (2025). "Context Length Alone Hurts." _EMNLP 2025_. arXiv

### Agent 長 context 基準

- SlopCodeBench (2025). arXiv
- OctoBench (2026). arXiv
- LoCoBench-Agent (Salesforce, 2025). arXiv
- ProcCtrlBench (2026). arXiv
- LOCA-bench (2026). arXiv

### LLM 長 context 基準

- RULER (NVIDIA, COLM 2024). arXiv
- BABILong (NeurIPS 2024). arXiv
- LongBench v2 (2024). arXiv
- HELMET (2024). arXiv

### 開源工具

- Chroma Context Rot Toolkit
- LoCoBench-Agent
- Promptfoo
