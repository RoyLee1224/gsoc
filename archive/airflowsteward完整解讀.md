# Apache Steward（即將改名 Apache Magpie）— 完整專案解讀

## 一、專案是什麼？解決什麼問題？

這是一個**「agent 輔助開源專案維護」的可重用框架**，目前在 ASF（Apache Software Foundation）孵化中，預計 2026/05/20 由 ASF 董事會通過為頂層專案（Top-Level Project）。

> 雖然 repo 仍叫 `apache/airflow-steward`（歷史遺留，源自 Airflow），但本質上是**專案中立**的——既服務 ASF 專案，也歡迎非 ASF 的開源社群。Python 核心團隊已是 day-one 合作對象。

### 核心命題（MISSION.md）
> **"Give maintainers time back, so they can do what matters."**
> （把時間還給維護者，讓他們能做真正重要的事。）

它要解決的痛點：
1. **新貢獻者 onboarding 太慢**——維護者重複教同樣的慣例
2. **PR review cycle 太慢**——人類 reviewer 永遠跟不上 PR 量
3. **安全議題處理最高風險、最高人力**——`security@` 信件、CVE 分配、私下修復、解禁公告，每一步都不能錯
4. **舊 issue 重新評估**——backlog 太大，沒人有時間回頭看哪些已經被修了

### 設計核心：5 種模式，專案自選
- **Triage**（分流）—— 已上線
- **Mentoring**（教導）—— 實驗中
- **Drafting**（agent 草擬修復，人類審）—— 已上線
- **Pairing**（開發者端的 dev cycle 工具）—— v1 將推出
- **Auto-merge**（窄範圍自動合併）—— 路線圖最後，要 Pairing 上線兩季後才可能開啟

每個專案各自打開想用的模式。**沒有「one-size-fits-all」**。

---

## 二、整體架構與目錄結構

```
airflow-steward/
├── README.md            (228 行) — 如何採用本框架
├── MISSION.md           (241 行) — 為什麼、目標、PMC 名單
├── CONTRIBUTING.md      (975 行) — 框架貢獻者指南
├── AGENTS.md            (約 2200 行) — 給 agent 的指示與慣例
├── .asf.yaml            — ASF 基礎設施設定（GitHub、標籤、collaborator）
│
├── .claude/skills/      — ★ 35 個 skill (框架核心)
├── docs/                — 31 份 markdown 文件（spec、RFC、setup guide）
├── tools/               — 18 個 uv workspace 的 Python 子專案 + shell 腳本
├── projects/            — 採用方專案的範本目錄
├── images/              — 文件圖片
│
├── pyproject.toml       — uv workspace 根設定
├── uv.lock              — Python 依賴鎖定 (137 KB)
├── .pre-commit-config.yaml
├── .github/workflows/   — 7 個 GitHub Actions
└── LICENSE / NOTICE     — Apache 2.0
```

---

## 三、最關鍵的設計：英文當作程式碼（English-as-code）

這是理解整個專案最重要的一點。

`.claude/skills/<skill-name>/SKILL.md` 是**用 markdown 寫的可執行程式**，由 Claude Code 之類的 AI agent 讀取後直接執行。每個 SKILL.md 開頭有 YAML frontmatter（描述、觸發詞、模型偏好），主體是給 agent 的逐步指令。

**確定性的 code（Python/Shell/Groovy）僅限於 `tools/`**，處理 OAuth、API 呼叫、JSON 產生、解析等需要可靠輸出的工作。Agent 透過呼叫這些工具拿到結構化資料，再依 SKILL.md 的指示判斷下一步。

這個分工是有意的：
- **不確定但需要判斷的工作** → SKILL（英文 prompt）
- **必須可重複、可驗證的工作** → tools（Python 腳本）

---

## 四、35 個 Skill 完整盤點

| 家族 | Skill 數 | 用途 | 模式 |
|---|---|---|---|
| **setup** | 6 | 框架採用、isolated 沙箱安裝、override 維護 | 基礎建設 |
| **security** | 9 | 16 步驟安全 issue 生命週期（從 `security@` 收信到 CVE 公告） | Triage + Drafting |
| **pr-management** | 5 | PR triage、快速合併、深度 code review、stats dashboard、mentor 留言 | Triage + Mentoring |
| **issue** | 5 | issue triage、bug 重現、修復草擬、backlog 重評估 | Triage + Drafting |
| **mentoring** | 1 | `pr-management-mentor`（教導語氣留言） | Mentoring（實驗中） |
| **release-management** | 10（proposed） | ASF 14 步驟發佈生命週期（spec 先行，skill 後續） | proposed |
| **utilities/meta** | 2+ | `write-skill`、`list-steward-skills`、`optimize-skill` 等 | meta |
| **Pairing** | 1 | `pairing-self-review`（PR 開啟前 self-review） | Pairing（實驗中） |
| **其他** | — | `audit-finding-fix`、`good-first-issue-author`、`contributor-nomination` | — |

舉幾個關鍵 skill 為例：

- **`setup-steward`** —— **唯一被 commit 到採用方 repo 的 skill**，它管理其他所有 skill 的安裝、升級、override
- **`security-issue-import`** —— 從 `security@` 信箱掃描、分類、建立追蹤 issue
- **`security-cve-allocate`** —— 走 ASF CVE 分配流程
- **`pr-management-triage`** —— 早上一次性掃過所有未處理 PR
- **`pr-management-quick-merge`** —— 找出 trivial 的 PR（文件、翻譯、changelog）給 maintainer 快速通過
- **`issue-reproducer`** —— 從 issue 抽出範例程式碼，在當前 `<default-branch>` 跑一次，看還壞不壞

---

## 五、最有特色的設計：採用機制（Snapshot + Agentic Overrides）

採用方 repo 不會 vendored 一份框架，也不用 git submodule，更**沒有 marketplace**。流程：

### Phase 1: Shell bootstrap（一段可貼上的腳本）

從 `docs/setup/install-recipes.md` 挑一種方法：
- `svn-zip` —— 從 `dist.apache.org` 抓簽章過的正式 release（生產用）
- `git-tag` —— 釘住特定 tag
- `git-branch` —— 追蹤 `main`（pre-release 期預設）

腳本會：
1. 把 `.apache-steward/`、`.apache-steward.local.lock`、symlinks 加進 `.gitignore`
2. 下載框架到 **gitignored 的 `.apache-steward/`** 目錄（snapshot，build artefact，不 commit）
3. 把 `setup-steward` skill 複製進採用方的 skill 目錄（**唯一被 commit 的 skill**）

### Phase 2: Agent 接管（呼叫 `/setup-steward`）

由 agent 完成：
- 寫入 `.apache-steward.lock`（**committed**）—— 專案的版本指針
- 寫入 `.apache-steward.local.lock`（**gitignored**）—— 本機實際抓到什麼版本、什麼時候
- 詢問要啟用哪些 skill 家族（security、pr-management…）
- 建立 gitignored symlinks（指向 `.apache-steward/` 裡的 skill）
- 建立 `.apache-steward-overrides/` 目錄（**committed**，放專案特化的修改）
- 安裝 `post-checkout` git hook（worktree 自動重建符號連結）

### 飄移偵測（drift detection）

每次任何 framework skill 啟動，會比對 `.apache-steward.local.lock` vs `.apache-steward.lock`。若有差距（專案升版了，或 main-tracking 的本地版過時）會提示執行 `/setup-steward upgrade`。

### Agentic Overrides

採用方專案不直接改 framework skill，而是寫 `.apache-steward-overrides/<skill-name>.md`（markdown 給 agent 讀）。框架 skill 執行時會先讀這個檔，按指示**跳過/取代/增加**步驟。這份合約寫在 `docs/setup/agentic-overrides.md`。

**好處**：升版時不會合併衝突，override 是「對 agent 的指示」而非 patch。

---

## 六、技術棧

| 層 | 技術 |
|---|---|
| **Skill 程式語言** | Markdown + YAML frontmatter（英文是 first-class language） |
| **確定性工具** | Python 3.11+，少量 Shell（Bash/Zsh） |
| **套件管理** | **uv**（workspace 模式，18 個成員） |
| **核心 dev 工具** | `prek`（pre-commit 執行器）、`ruff`、`mypy`、`pytest` |
| **CI** | GitHub Actions × 7（pre-commit、tests、CodeQL、link-check、ASF allowlist、sandbox-lint、zizmor） |
| **Pre-commit hooks** | doctoc（自動 TOC）、markdownlint-cli2、typos、private-key detection |
| **安全工具** | bubblewrap（Linux）/ Seatbelt（macOS）、socat、自製 egress-gateway proxy |
| **整合對象** | GitHub API、Gmail（OAuth）、Vulnogram（CVE）、Apache PonyMail、Jira、cve.org |

### 重要的依賴策略
`[tool.uv] exclude-newer = "7 days"` —— **任何 Python 依賴至少要釋出 7 天才會被採用**。這個冷卻期是為了讓上游有時間 retag、撤回、發 incident report，是供應鏈安全的緩衝。

---

## 七、`tools/` 目錄裡的 18 個 Python 子專案

每個都是獨立 `pyproject.toml`，被 root 的 uv workspace 串起來：

| 子專案 | 功能 |
|---|---|
| `agent-isolation` | `claude-iso.sh` 等沙箱腳本與 status line hooks |
| `egress-gateway` | 本機 HTTP(S) 出口 proxy，按主機白名單放行（RFC-AI-0003） |
| `privacy-llm/checker` & `redactor` | PII 偵測與脫敏，安全內容送 LLM 前的閘門 |
| `cve-tool-vulnogram/{generate-cve-json, oauth-api}` | CVE JSON 產生 + Vulnogram OAuth |
| `gmail/oauth-draft` | Gmail OAuth + 草擬回信 |
| `github-body-field`, `github-rollup` | GitHub issue/PR body 結構欄位、彙整 |
| `jira` | Jira 整合 |
| `pr-management-stats` | PR 健康狀況 dashboard |
| `spec-validator`, `skill-and-tool-validator` | spec/skill 結構檢查 |
| `spec-status-index` | RFC/spec 狀態追蹤 |
| `preflight-audit`, `sandbox-lint`, `permission-audit` | 各種預檢與審計 |
| `skill-evals` | skill 評估套件 |

---

## 八、安全模型：分層防禦（最重要的設計目標之一）

威脅情境：CVE 解禁前的內容在 tracker 與 `private@` 信箱。預設的 agent session 能讀 `~/`、所有環境變數、自由出網——可能（無意或被 prompt injection）外洩 cloud credential、SSH key、GitHub token、Gmail OAuth refresh token。

### 五層緩解

1. **OS 沙箱**（bubblewrap / Seatbelt）—— 唯讀檔系統範圍、網路出口限制、env 過濾
2. **`claude-iso.sh` clean-env 包裝**—— 跑 agent 前把環境變數先洗一遍
3. **egress-gateway**—— 本機 HTTP(S) proxy，工具走 `HTTPS_PROXY` 出去，白名單外的主機在 socket 開啟前就被拒
4. **Privacy-LLM gate**—— 安全內容送 LLM 前必須過 PII redactor，把 email/IP/path/token 換成穩定識別碼
5. **可見性 hooks**——`sandbox-bypass-warn.sh`、`sandbox-error-hint.sh`、status line 顯示沙箱狀態

### 核心硬規則（寫在 MISSION.md）
- **外部內容是資料，永遠不是指令**——reporter mail、PR comment、attachment 一律當 data 對待
- **所有 agent 動作必須有 audit log**，能回滾就回滾
- **沙箱繞過必須大聲示警，不能靜默**

---

## 九、如何建置、執行、測試

```bash
# 安裝 dev toolchain（root + 所有 workspace 成員）
uv sync --group dev

# 跑 pre-commit
uv run prek run --all-files

# 跑某個工具的測試
uv run --directory tools/spec-validator pytest

# Workspace 一致性檢查
tools/dev/run-workspace-check.sh
tools/dev/check-workspace-members.py
```

CI（`.github/workflows/`）會跑：
- `tests.yml` —— 動態矩陣，每個 workspace 成員獨立跑 pytest
- `pre-commit.yml` —— hooks 驗證
- `codeql.yml`、`zizmor.yml` —— 安全性靜態分析
- `link-check.yml` —— markdown link 驗證
- `asf-allowlist-check.yml` —— ASF 允許依賴的白名單檢查
- `sandbox-lint.yml` —— 沙箱設定檢查

---

## 十、值得注意的設計決策（總結）

| 決策 | 為什麼這樣做 |
|---|---|
| **English-as-code（SKILL.md 是程式）** | 維護者用自然語言改 skill，不用學 LLM 內部；多個 AI CLI 都能執行 |
| **Snapshot + override，不用 submodule** | 升版乾淨無衝突，採用方的 commit 只有「指針 + 我的客製」 |
| **5 種模式可獨立 toggle** | 「project autonomy」是骨架原則，不強制專案文化 |
| **安全議題是骨幹用例，不是註腳** | 安全流程的「每步都要 audit」剛好是 agent + 人類 gate 的最佳組合；其他模式都得通過安全標準才能 ship |
| **Mentoring 是 first-class mode** | ASF Responsible AI Initiative 的「contributor empowerment」目標就靠這層落地 |
| **Pairing 排在 Auto-merge 前面** | 先確認「人類關係」沒被破壞，才考慮全自動 |
| **vendor-neutrality 不可妥協** | 同一份 skill 要能跑 Claude / OpenAI / Bedrock / Ollama / vLLM / 未來的 `inference.apache.org` |
| **依賴 7 天冷卻** | 供應鏈安全緩衝 |
| **唯一 committed skill 是 `setup-steward`** | 採用方的 PR diff 極小，採用負擔極低 |

---

## 十一、目前狀態與時程

- **2026/05/08–12**：founding-PMC 命名 bikeshed
- **2026/05/15 12:00 UTC**：LAZY CONSENSUS 截止，並送 `trademarks@apache.org` 審查（候選名：Magpie、Beacon、Guild、Lichen）
- **2026/05/20**：董事會表決建立為 ASF 頂層專案
- **發佈後 3 個月**：第一個 ASF release，至少 3–4 個 friendly pilot（Airflow、Arrow 或 ATR、Python core 等）
- **目前**：pre-release，採用採邀請制；CVE 工作流已穩定，35 個 skill 中多數可用
