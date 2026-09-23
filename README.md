# n8n AI Agent Workflows（修正版）

對應教學文：  
https://leosplain.com/n8n-ai-agent-workflow-tutorial/

本目錄為**可匯入、結構已修正**的範本。原始 ZIP 有多處無法直接執行的問題，已在此修正（見下方「修正清單」）。

## 內建 3 組流程

| 檔案 | 用途 | 需準備 |
|------|------|--------|
| `workflows/01-rss-ai-digest.json` | Schedule → RSS → 彙整 → AI 摘要 → Telegram | OpenAI（或改其他 Chat Model）、Telegram Bot |
| `workflows/02-form-inquiry-notion.json` | Webhook 表單 → AI 抽 JSON → 解析 → Notion | OpenAI、Notion Integration + Database |
| `workflows/03-ollama-privacy-pipeline.json` | 監看 `/data/incoming` → 讀檔 → Ollama 摘要 → 寫入 `/data/processed` | Docker 內 Ollama、已 `ollama pull` 的模型 |

## 快速上手

### 1. 啟動 n8n + Ollama

```bash
docker compose up -d
```

- n8n：http://localhost:5678  
- Ollama API：http://localhost:11434  

第一次使用 Ollama 請進容器拉模型，例如：

```bash
docker exec -it n8n-ollama ollama pull gemma2:2b
# 或文章建議的 gemma4 系列（依你本機硬體選擇）
```

### 2. 匯入工作流

n8n → **Add workflow** → **Import from File** → 選 `workflows/*.json`。

### 3. 設定憑證（必做）

匯入後節點上的 Credential 會是 placeholder，請在 n8n UI：

1. **Credentials** 建立 OpenAI / Telegram / Notion  
2. 打開對應節點，改選你建立的憑證  
3. Workflow 01：環境變數或 Expression 中的 `TELEGRAM_CHAT_ID`  
4. Workflow 02：Notion Database 需有對應屬性（見下），並把 Integration 分享給該 Database；`NOTION_DATABASE_ID` 可寫在環境變數或節點內  

**Notion 建議欄位（與節點對應）：**

- `Name`（Title）  
- `Summary`（Rich text）  
- `Category`（Select：技術諮詢 / 合作邀約 / 報價需求 / 其他）  
- `Urgency`（Select：高 / 中 / 低）  

若你的 DB 欄位名稱不同，請在 Notion 節點裡改 mapping。

### 4. Workflow 03 本機檔案

`docker-compose.yml` 已掛載：

- 主機 `./data` → 容器 `/data`  

放入檔案測試：

```bash
echo "這是一份內部會議紀錄，討論預算與時程。" > data/incoming/test.txt
```

摘要會出現在 `data/processed/`。

## 原始附件問題與修正清單

| 問題 | 原始狀態 | 修正後 |
|------|----------|--------|
| AI Agent 未接 Language Model | 01、02 只有 Agent，執行會報「必須連接 Chat Model」 | 已加 OpenAI Chat Model 並連 `ai_languageModel` |
| RSS 多筆直接進 Agent | 每筆各跑一次、prompt 不像「清單」 | 加 Code 節點彙整前 15 筆再摘要 |
| Notion 吃 `$json.output.name` | Agent 輸出多半是字串，不是物件 | 加 Parse AI JSON（清 markdown fence + JSON.parse） |
| Local File Trigger 無內容 | 只給 `event`/`path`，prompt 用 `$json.content` 會空 | Trigger → Read/Write File → Extract From File → Agent |
| writeBinaryFile + 純文字 | 舊節點且需要 binary；參數也不對 | 改用 Convert to File + Read/Write Files from Disk |
| docker-compose 無 `/data` | 03 無法真正監看/寫入 | 掛載 `./data:/data` |
| README / .env 做成 .docx | 不利版控與 `cp .env.example .env` | 改為純文字 `README.md`、`.env.example` |
| Ollama 位址 | 文章寫 `host.docker.internal`（n8n 在 Docker、Ollama 在 host 時） | 同一 compose 網路用 `http://ollama:11434`（已寫入 03） |

## 與文章（leosplain.com）的差異／注意

文章觀念正確（先無 AI 三節點、再掛模型、對外停在草稿、Docker 連 Ollama 不要用容器內 localhost）。  
附件與文章仍有落差：

1. **文章 TEMPLATE 的 Prompt** 比原始 JSON 完整；修正版 01/02 已對齊文章語氣與 JSON 規範。  
2. **文章建議** Docker 內 n8n 連 host Ollama 用 `host.docker.internal`；本 compose 把 Ollama 當同網路服務，應用 **`http://ollama:11434`**。若你只跑 n8n 容器、Ollama 在宿主機，再改回 `host.docker.internal:11434`。  
3. **文章寫 gemma4**；修正版 03 預設 `gemma2:2b`（較小、易拉）。請依 `ollama list` 改模型名。  
4. **文章 TEMPLATE 03 客服草稿** 附件沒有對應 JSON（只有隱私檔案管線）；客服流程需自行加 Gmail「建立草稿」。  
5. **README 原文** 寫「均已驗證相容」——在未接 Model、未掛 volume 前**無法直接跑通**，故不應宣稱已驗證。  

## 仍需你本機測試的項目

以下無法在此環境代你完整跑通（需你的 API Key、Telegram、Notion、實際檔案）：

- [ ] 01：Schedule + 真實 RSS + OpenAI + Telegram 收到訊息  
- [ ] 02：Webhook POST `{"message":"..."}` → Notion 出現一列  
- [ ] 03：`ollama pull` 後丟檔到 `data/incoming`，`data/processed` 有摘要  
- [ ] Notion 欄位名稱 / Select 選項是否與節點一致  
- [ ] 節點 typeVersion 隨你安裝的 n8n 版本是否提示升級（可在 UI 一鍵更新）  

## License

MIT（與原專案意圖一致）
