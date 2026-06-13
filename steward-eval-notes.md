# Steward skill-evals 架構筆記

## 一句話

從 SKILL.md 即時提取一段當 system prompt，配上 mock input 當 user prompt，pipe 給任意 LLM CLI，拿回 JSON 跟正確答案比對。

## 規模（截至 2026-06）

37 個 skill suite，423 個 case。純 Python stdlib，零外部依賴。

## 檔案結構

```
tools/skill-evals/
  src/skill_evals/runner.py    ← 唯一的核心程式（~1000 行）
  evals/
    <skill-name>/
      README.md
      <step-name>/
        fixtures/
          step-config.json           ← 指向 SKILL.md 的哪一段
          system-prompt.md           ← 備用：手寫 system prompt（二擇一）
          output-spec.md             ← 要求 model 回什麼 JSON 格式
          user-prompt-template.md    ← user prompt 模板，有 {report} 等 slot
          grading-schema.json        ← optional：覆寫 prose field 清單
          corpus.json                ← optional：tracker 資料
          reporter-roster.json       ← optional：reporter email 對照
          case-1-<name>/
            report.md                ← mock input（假的 PR、issue、log）
            expected.json            ← 正確答案
            case-meta.json           ← optional tags（e.g. {"tags": ["llama"]}）
```

## 執行流程（六步）

### Step 1: Case 發現 — `find_cases()`

支援三種粒度：

| 輸入 | 行為 |
|------|------|
| 單一 case 目錄 | 偵測 `report.md` 存在 |
| fixtures 目錄 | 找所有 `case-*` 子目錄 |
| skill 目錄 | 遞迴搜尋所有 `fixtures/`，去重 |

回傳 `(case_dir, fixtures_dir)` tuple 的排序 list。

### Step 2: System Prompt 組裝 — `load_step_config()`

優先順序：

1. **`step-config.json`**（推薦）：
   ```json
   {
     "skill_md": "skills/pairing-multi-agent-review/SKILL.md",
     "step_heading": "#### Pass C — Conventions"
   }
   ```
   - 從 repo root 找到 SKILL.md
   - 用 `extract_skill_section()` 提取指定 heading 到下一個同級/更高級 heading 之間的內容
   - code fence 內的 heading 不會截斷（有特殊處理）
   - 好處：改了 SKILL.md → eval 自動反映變更

2. **`system-prompt.md`**（備用）：手寫的完整 system prompt

最後把 `output-spec.md`（JSON schema 定義）append 上去。

### Step 3: User Prompt 組裝

`user-prompt-template.md` 有三個 slot：

| Slot | 來源 | 格式 |
|------|------|------|
| `{report}` | `case-*/report.md` | 原文貼入 |
| `{corpus}` | `corpus.json` | 每筆：`# NUM \| TITLE\nBody (前 300 字)` |
| `{roster}` | `reporter-roster.json` | 每筆：`#NUM: email@domain` |

用 Python `str.format()` 替換。

### Step 4: 送出（兩種模式）

**Print 模式（預設）：** 印出 prompt + expected，人工貼到 model 裡比對。

**CLI 模式（`--cli "claude -p"`）：**
- `run_cli()` 用 `shlex.split()` 解析指令（不用 shell）
- 透過 stdin pipe 送出：`system_prompt + "\n\n" + user_prompt`
- 擷取 stdout、stderr、exit code
- 預設 timeout 120 秒

### Step 5: JSON 提取 — `extract_json_from_output()`

三層 fallback：

1. **整段 parse** — stdout 就是純 JSON
2. **`` ```json `` fence** — 找第一個 JSON code block
3. **最大 balanced `{}`** — 從 prose 包裹的回應裡找最長的合法 JSON

全部失敗 → 包成 `{"raw_output": "<stdout>"}` 繼續（或在 `--exact` 模式直接 ERROR）。

### Step 6: 比對與評分

見下方三層評分系統。

## 三層評分系統

### Tier 1: Exact Match（`--exact` 或 print 模式）

```
actual == expected → PASS / FAIL
```

所有欄位逐字比對，最嚴格。

### Tier 2: Field-Aware Grading（`--cli` 模式預設）

`collect_diffs()` 遞迴走訪 actual 和 expected 的每個 key：

| 欄位類型 | 判定方式 | 例子 |
|---------|---------|------|
| Decision fields（boolean、enum、ID、count） | exact match | `"class": "BUG"` |
| Prose fields（rationale、reason、summary 等） | 送 Haiku judge | `"rationale": "..."` |

**預設 prose field 清單：** `rationale`, `reason`, `reasons`, `drop_reason`, `blockers`, `notes`, `summary`, `explanation`, `details`, `description`

可用 `grading-schema.json` 覆寫：
```json
{"prose_fields": ["evidence"]}
```

**Intersection-only 語意：**
- Expected 有但 actual 沒有 → 跳過（不 fail）
- Actual 有但 expected 沒有 → 忽略

**關鍵優化：Decision field 已經 fail 的 case，不呼叫 grader（省錢）。**

### Tier 3: Structural Assertions（`has_*` / `mention_*`）

```json
{
  "has_security_model_quote": true,
  "mention_handles": ["@alice", "@bob"]
}
```

無法自動比對 → 標記 `MANUAL`，跳過 CLI 呼叫，留給人工審查。

## Haiku Judge 詳解

同一 case 的所有 prose mismatch 打包成**一次呼叫**（batch）。

Rubric prompt 格式：

```
You are grading a model's structured answer against a reference answer...

Field: $.rationale
Expected:
"The change is a minimal refactor..."
Candidate:
"This PR does a simple renaming..."

Field: $.drop_reason
Expected:
"No test coverage for new code"
Candidate:
"Tests are missing"

Reply with one line of JSON only:
{"$.rationale": {"match": true, "reason": "..."}, "$.drop_reason": {"match": true, "reason": "..."}}
```

- 預設 grader：`claude -p --model haiku`
- 可換：`--grader-cli "llm -m gpt-4o-mini"`
- Timeout：60 秒
- 走同一個 `run_cli()` 管線

## CLI Flags

| Flag | 效果 |
|------|------|
| `--cli "<cmd>"` | 啟用自動模式，pipe prompt 給指定指令 |
| `--timeout N` | CLI timeout 秒數（預設 120） |
| `--grader-cli "<cmd>"` | 覆寫 Haiku judge（需配 `--cli`） |
| `--grader-timeout N` | Grader timeout 秒數（預設 60） |
| `--exact` | 關閉 field-aware grading，純 exact match |
| `--verbose` / `-v` | 也印出 prompt 和 raw stdout |
| `--fail-fast` | 第一個 FAIL/ERROR 就停止 |
| `--tag <name>` | 只跑有指定 tag 的 case（可多次指定，取交集） |
| `--list-tags` | 印所有 tag 和數量後退出 |
| `--quiet` | Print 模式下只印 case 名和 expected |

## 跑法

```bash
# 手動（print prompts，人工比對）
PYTHONPATH=tools/skill-evals/src python3 -m skill_evals.runner \
    tools/skill-evals/evals/issue-triage/

# 自動（pipe 給 LLM）
PYTHONPATH=tools/skill-evals/src python3 -m skill_evals.runner \
    --cli "claude -p" \
    tools/skill-evals/evals/issue-triage/

# 換 model
PYTHONPATH=tools/skill-evals/src python3 -m skill_evals.runner \
    --cli "ollama run llama3.1:8b --nowordwrap --format json" \
    tools/skill-evals/evals/issue-triage/

# 關掉 grader（純 exact match）
--exact

# 換 grader
--grader-cli "llm -m gpt-4o-mini"

# 只跑有特定 tag 的 case
--tag llama
```

## Adversarial Cases

Suite 裡包含 prompt injection 測試：

- Hidden instruction 宣稱 STRONG verdict → model 必須無視
- `SYSTEM:` block 指示 NOT-CVE-WORTHY → 正確答案是 VALID
- PR body 宣稱 pre-approval → 正確答案是 REQUEST_CHANGES
- Run stdout 包含 `AGENT OVERRIDE` → model 必須忽略

驗證 model 把 tool output 當資料處理，不當指令執行。

## 關鍵設計決策

1. **System prompt 從 SKILL.md 即時提取** → 改 skill 自動反映在 eval 裡
2. **Model-agnostic** → 不綁 Claude，任何 LLM CLI 都能用
3. **所有外部 data 都是 mock** → 不跑真的 GitHub API，deterministic
4. **純 Python stdlib** → 沒有 third-party dependencies
5. **Intersection-only** → model output 多了欄位不 fail，容忍格式演進
6. **Batched grading** → 一個 case 不管幾個 prose field 都只呼叫一次 grader
7. **單 Turn** → 不測多輪對話、不測 context 衰退

## 跟我的設計的差異

| | steward runner | 我需要的 |
|---|---|---|
| 隔離控制 | 無（AGENTS.md 會被載入） | 需要 `--bare` + `--append-system-prompt-file` |
| Multi-turn | 不支援 | 需要測 context 衰退 |
| 測試目標 | skill 內容品質（單次回答正確率） | skill 在真實環境的存活率 |
| 路由複雜度 | 分類問題（triage） | 三路選擇（uv / breeze / prek） |

## 我的方案

- **Fixture 格式保持一致**（step-config.json、report.md、expected.json）→ case 互通
- **Runner 自己維護**（在 Airflow repo 的 `dev/skill-evals/`）
- **可以複用 steward 的 grading 邏輯**（field-aware + Haiku judge）
