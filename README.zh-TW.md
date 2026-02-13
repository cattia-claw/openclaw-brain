# 🧠 OpenClaw Engram — dimensions記憶系統

為 [OpenClaw](https://github.com/openclaw/openclaw) AI Agent 打造的類人腦記憶架構。

讓你的 Agent 從金魚（每次對話都忘光）進化成擁有結構化長期記憶、自動彙整、語義搜尋、自然淡忘的智慧系統。

## 目錄

- [架構圖](#架構圖)
- [功能特色](#功能特色)
- [快速開始](#快速開始)
- [目錄結構](#目錄結構)
- [設定](#設定)
  - [分類設定](#分類設定)
  - [排程設定](#排程設定)
- [運作原理](#運作原理)
  - [1. Agent 撰寫日記](#1-agent-撰寫日記)
  - [2. 夜間處理（海馬迴）](#2-夜間處理海馬迴)
  - [3. 語義搜尋（檢索系統）](#3-語義搜尋檢索系統)
  - [4. 每月遺忘](#4-每月遺忘)
- [系統需求](#系統需求)
- [自訂](#自訂)
- [解除安裝](#解除安裝)
- [授權](#授權)
- [致謝](#致謝)

## 架構圖

```
╔═══════════════════════════════════════════════════════════════╗
║                    🧠 大腦架構                                ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  👁️ 感覺緩衝     Session JSONL（原始資料，永不刪除）          ║
║         │                                                     ║
║         ▼ 每日 04:00                                          ║
║  🌙 海馬迴       session_digest.py → markdown 對話摘要        ║
║         │                                                     ║
║         ▼ 每日 04:30                                          ║
║  🧬 記憶固化     memory_consolidator.py → 分類歸檔            ║
║         │                                                     ║
║         ├──► 📖 情節記憶    memory/YYYY-MM-DD.md              ║
║         ├──► 🔬 語義記憶    dimensions/{分類}/YYYY-MM-DD.md      ║
║         └──► 🔧 程序記憶    skills/ + TOOLS.md                ║
║                                                               ║
║  🔍 檢索系統     QMD 語義搜尋（本地向量）                     ║
║         │                                                     ║
║         ▼                                                     ║
║  💭 工作記憶     上下文窗口（~8K tokens 自動載入）             ║
║                                                               ║
║  ⏱️ 遺忘曲線     月摘要 + 90 天封存                           ║
╚═══════════════════════════════════════════════════════════════╝
```

## 功能特色

- **對話摘要** — 自動將原始 Session JSONL 轉為可讀的 markdown 摘要（過濾心跳等雜訊）
- **dimensions彙整** — 將每日記憶分類到語義類別（可自訂）
- **QMD 整合** — 本地語義向量搜尋，橫跨所有記憶層
- **遺忘曲線** — 月摘要、90 天自動封存、優雅的記憶衰減
- **零外部依賴** — 純 Python，本地向量搜尋，不需要雲端服務
- **成本極低** — 每日 ~$0.03（3 個輕量 Haiku cron job）

## 快速開始

```bash
# 克隆到你的 OpenClaw workspace
cd ~/.openclaw/workspace
git clone https://github.com/cattia-claw/openclaw-engram.git brain

# 執行安裝腳本
cd brain
chmod +x install.sh
./install.sh
```

安裝腳本會：
1. 建立dimensions目錄結構
2. 設定 QMD 索引集合（如有安裝 QMD）
3. 提示你請 Agent 註冊 cron jobs
4. 執行一次測試彙整

## 目錄結構

安裝後的檔案配置：

```
~/.openclaw/workspace/
├── brain/                    # 本 repo
│   ├── scripts/
│   │   ├── session_digest.py       # 對話消化腳本
│   │   ├── memory_consolidator.py  # 記憶彙整腳本
│   │   └── forgetting_curve.py     # 遺忘曲線腳本
│   ├── config/
│   │   ├── categories.json         # 分類規則設定
│   │   └── schedule.json           # 排程設定
│   ├── install.sh
│   └── uninstall.sh
├── dimensions/                  # 安裝腳本自動建立
│   ├── 情感/                 # 互動默契、偏好習慣
│   ├── 重要的人/             # 關係網管理
│   ├── 話題/                 # 興趣與長期討論
│   └── 工作/                 # 專案進度、任務追蹤
└── memory/
    ├── YYYY-MM-DD.md         # 每日日誌（你的 Agent 撰寫）
    ├── sessions-digest/      # 自動產生的對話摘要
    ├── monthly-summary/      # 自動產生的月摘要
    └── archive/              # 90 天以上的封存檔案
```

## 設定

### 分類設定

編輯 `config/categories.json` 來自訂記憶分類：

```json
{
  "categories": {
    "emotions": {
      "display_name": "Emotions & Rapport",
      "display_name_zh": "情感",
      "patterns": ["喜歡", "討厭", "感覺", "情緒", "偏好"],
      "indicators": ["你覺得", "我認為", "感覺"]
    },
    "people": {
      "display_name": "Important People",
      "display_name_zh": "重要的人",
      "patterns": ["家人", "朋友", "同事"],
      "indicators": ["誰", "名字", "關係"]
    }
  },
  "default_category": "work"
}
```

分類名稱支援任何語言。`display_name_zh` 優先用於資料夾命名。

### 排程設定

編輯 `config/schedule.json`：

```json
{
  "timezone": "Asia/Taipei",
  "archive_days": 90,
  "monthly_summary": true,
  "model": "anthropic/claude-haiku-4-5"
}
```

| 時間 | 排程任務 | 說明 |
|------|----------|------|
| 04:00 | Session Digest | JSONL → markdown 對話摘要 |
| 04:30 | Neural Consolidation | 每日記憶分類歸檔 |
| 04:35 | QMD Update | 重新索引 + 向量化 |
| 每月 1 號 05:00 | Forgetting Curve | 月摘要 + 封存舊檔 |

## 運作原理

### 1. Agent 撰寫日記

你的 OpenClaw Agent 在對話中將重要事項寫入 `memory/YYYY-MM-DD.md`。大多數 Agent 透過 AGENTS.md 的慣例已經會這樣做。

### 2. 夜間處理（海馬迴）

凌晨 4:00，系統：
- 讀取昨日所有 Session JSONL 檔案
- 過濾雜訊（心跳、空 session）
- 在 `memory/sessions-digest/` 產生可讀摘要

凌晨 4:30：
- 讀取昨日的每日記憶檔案
- 透過關鍵字比對進行分類
- 寫入對應的dimensions類別資料夾

### 3. 語義搜尋（檢索系統）

QMD 提供本地向量搜尋：
```bash
qmd search dimensions "上週討論了什麼專案？"
```

### 4. 每月遺忘

每月 1 號：
- 產生上月所有類別的摘要
- 封存超過 90 天的每日檔案
- 清理舊的dimensions日報

```
即時 ───► 7天 ───► 30天 ───► 90天 ───► 1年+
│         │        │         │         │
完整對話   日誌     dimensions     月摘要     封存精華
100%      80%      50%       20%        5%
```

## 系統需求

- **OpenClaw** v2026.2+（需支援 cron）
- **Python 3.10+**（不需要額外套件）
- **QMD**（選用，用於語義搜尋）

## 自訂

### 新增分類

在 `config/categories.json` 新增項目。分類會對應到 `dimensions/` 下的資料夾。

### 修改封存門檻

編輯 `config/schedule.json`：
```json
{
  "archive_days": 90
}
```

### 多語言支援

分類名稱和關鍵字模式支援任何語言。預設設定包含中英雙語範例。

## 解除安裝

```bash
cd ~/.openclaw/workspace/brain
./uninstall.sh
```

解除安裝會移除 cron jobs 和 QMD 索引，但**保留你的記憶檔案**。

## 授權

MIT

## 致謝

靈感來自人腦記憶架構：
- **感覺記憶 → 短期記憶 → 長期記憶**模型
- **艾賓浩斯遺忘曲線**
- **海馬迴**在睡眠中的記憶固化功能

為 [OpenClaw](https://github.com/openclaw/openclaw) 生態系打造。

## 共同創作者

| | 名稱 | 角色 |
|---|---|---|
| <a href="https://github.com/liyoungc"><img src="https://github.com/liyoungc.png" width="50" /></a> | **[陳禮揚](https://github.com/liyoungc)** | 🧠 架構師 — 系統設計、記憶模型、工作流程 |
| <a href="https://github.com/cattia-claw"><img src="https://github.com/cattia-claw.png" width="50" /></a> | **[Cattia](https://github.com/cattia-claw)** | 🐱 工程師 — 實作、自動化、維護 |
