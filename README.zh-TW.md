# Moltbook Dataset Creator

[English](./README.md)

從 Moltbook 的多 agent 討論中建立可用於 SFT 與 DPO 的訓練資料集。

Moltbook Dataset Creator 是一個 platform-agnostic 的 skill，能協助 AI
agent 完成從問題規劃、發文、收割回覆、評分、幻覺偵測，到最終資料集匯出的完整流程。

## 為什麼做這個專案

Moltbook 的特殊價值在於，一篇貼文通常會收到多個獨立 AI agent 的回覆，因此可以：

- 比較同一問題下的不同解法與回答風格
- 在多個 agent 之間做交叉驗證
- 在訓練前先篩掉低訊號樣本
- 產生 DPO 所需的 chosen / rejected 配對

這個專案的目的，就是把這種多 agent 訊號轉成可重複執行的 dataset pipeline。

## 功能特色

- 根據使用者主題生成平衡的問題集
- 自動探索近期論文，並轉成適合 Moltbook 的討論貼文
- 為每個問題選擇合適的 Moltbook submolt
- 本地追蹤 `post_id`、收集狀態與 follow-up
- 依排程收割回覆並去重
- 根據回覆多樣性動態決定是否追問
- 依多個品質維度評分回覆
- 在資料納入前先做 hallucination 偵測
- 匯出 ChatML、Alpaca、ShareGPT、DPO pairs 與 raw metadata

## 快速開始

### 1. 先看 skill 主要內容

主 skill 定義位於：

- [`SKILL.md`](./SKILL.md)

輸出格式參考位於：

- [`references/output-formats.md`](./references/output-formats.md)

論文探索參考位於：

- [`references/paper-discovery.md`](./references/paper-discovery.md)

### 2. 安裝 skill

此 repo 直接包含 source skill 內容本身。

請依照你使用的 agent 環境選擇安裝方式。

#### Codex

將這個 skill 資料夾複製到：

```text
~/.codex/skills/moltbook-dataset-creator
```

如果有設定 `CODEX_HOME`，實際路徑則為：

```text
$CODEX_HOME/skills/moltbook-dataset-creator
```

新增 skill 目錄後，建議重新啟動 Codex，讓它完整載入。

#### Claude Code

若要全域安裝、讓所有專案都能使用，請複製到：

```text
~/.claude/skills/moltbook-dataset-creator
```

若只想在單一專案中使用，請複製到：

```text
.claude/skills/moltbook-dataset-creator
```

Claude Code 會從該目錄中的 `SKILL.md` 載入 skill，並同時讀取同資料夾內的支援檔案。

如果 session 啟動時頂層 skills 目錄尚不存在，新增後建議重新啟動 Claude Code。

#### OpenClaw

若要安裝到目前 workspace，請複製到：

```text
./skills/moltbook-dataset-creator
```

OpenClaw 也支援其他層級的位置，例如：

```text
~/.openclaw/skills/moltbook-dataset-creator
~/.agents/skills/moltbook-dataset-creator
<workspace>/.agents/skills/moltbook-dataset-creator
```

如果之後這個 skill 發布到 ClawHub，也可以直接用 OpenClaw 原生命令安裝：

```bash
openclaw skills install <skill-slug>
```

安裝完成後，請開新的 OpenClaw session，讓 workspace skill 清單刷新。

### 3. 讓你的 agent 執行工作流程

常見需求例如：

- 「用 Moltbook 建立一批 Kubernetes security 的 SFT 資料」
- 「幫我發 20 題、收割 agent 回覆，並輸出 ChatML 與 DPO pairs」
- 「我已經收好 Moltbook comments，請幫我評分、抓 hallucination，並保留高品質樣本」

## 工作流程概覽

### Phase 0: Planning

先收集：

- 主題
- 規模
- 各階段模型選擇
- 需要的輸出格式
- 問題來源（`topic`、`paper`、`mixed`）

### Phase 1: Question Generation and Posting

- 生成平衡的問題集
- 為每題選擇合適的 submolt
- 發布問題到 Moltbook
- 在本地保存追蹤資訊

### Phase 2: Harvesting Replies

- 依排程輪詢貼文
- 只收集新增回覆
- 判斷是否值得追問
- 在回覆收斂或檢查次數用完後停止

### Phase 3: Quality Assessment

每則回覆都會依下列維度評分：

- relevance
- depth
- accuracy
- actionability
- uniqueness

這一階段也會包含交叉驗證與 hallucination 偵測。

### Phase 4: Dataset Generation

將篩選後的資料輸出成指定格式，並保留 metadata 與 provenance。

## 支援的輸出格式

- Messages / ChatML
- Alpaca
- ShareGPT
- DPO chosen/rejected pairs
- Raw Q&A with metadata

詳細格式請見：

- [`references/output-formats.md`](./references/output-formats.md)

## Moltbook 官方限制

本 repo 已對齊目前公開的 Moltbook 官方文件：

- 官方 skill：<https://www.moltbook.com/skill.md>
- 官方 rules：<https://www.moltbook.com/rules.md>

重要限制如下：

- 一律使用 `https://www.moltbook.com`，必須帶 `www`
- API 預算：`100 requests / minute`
- 已建立超過 24 小時的 agent：`1 post / 30 minutes`
- 新 agent 前 24 小時：`1 post / 2 hours`
- 已建立超過 24 小時的 agent：`1 comment / 20 seconds`，每日最多 `50`
- 新 agent 前 24 小時：`1 comment / 60 seconds`，每日最多 `20`
- post / comment 可能需要先完成 verification challenge，內容才會顯示

如果無法確認帳號年齡，工作流程應預設採用較嚴格的新 agent 限制。

## 使用前提

執行此 skill 的 agent 應已具備：

- Moltbook API key
- 對 `https://www.moltbook.com/api/v1` 的存取能力
- 寫入本地追蹤檔與輸出檔案的能力

本專案本身不處理 Moltbook 註冊流程。

## 專案結構

```text
.
├── README.md
├── README.zh-TW.md
├── SKILL.md
├── agents/
│   └── openai.yaml
└── references/
    ├── output-formats.md
    └── paper-discovery.md
```

## 專案狀態

目前 repo 狀態如下：

- skill source 已完成
- 英文與繁體中文 README 已建立
- `.skill` 打包產物已生成
- 專案文檔已更新為符合最新公開 Moltbook 限制

## 備註

- skill 核心流程採 platform-agnostic 寫法
- `agents/openai.yaml` 是給 Codex / OpenAI 類 skill metadata 使用
- 若有測試資料或打包產物，建議放在 skill repo 外部，避免影響實際分發內容
