# comfyui-agent：Agent 生圖工作區規範

當使用者在本目錄啟動 session 要求生圖、測試 LoRA 或合成角色時，透過下列兩個 Streamable HTTP MCP Server 操作：
- **`comfyui-local`**（[kime541200/comfy-mcp](https://github.com/kime541200/comfy-mcp)，預設 `http://127.0.0.1:8091/mcp`）：查詢節點與模型、驗證與執行 Workflow、下載模型。
- **`ai-pose-gallery`**（[kime541200/ai-pose-gallery-mcp](https://github.com/kime541200/ai-pose-gallery-mcp)，預設 `http://127.0.0.1:8090/mcp`，選配）：檢索人像／服裝／姿勢參考圖並執行多圖角色合成。

若 MCP 工具無法呼叫，先執行 `curl -s http://127.0.0.1:8091/health` 並參照 `docs/setup.md` 協助使用者檢查服務狀態。

## 0. MCP 調用規範（嚴禁繞道執行宿主機腳本）

- **強制使用 MCP 工具**：本工作區所有素材檢索、工作流驗證、生圖與 ComfyUI 管理**一律必須透過 MCP 工具**操作，僅在 MCP 工具無法達成目標時才停下來向用戶確認。
- **嚴禁繞道執行本機命令**：**嚴格禁止**自行透過 `run_command` 呼叫宿主機 Python 虛擬環境（如 `.venv/bin/python`）、寫臨時 Python 腳本或用 curl 直接存取後端 API 來代替 MCP 工具。
- **Lazy MCP 調用方式**：在 Antigravity 等 Agent 執行環境中，`comfyui-local` 與 `ai-pose-gallery` 均註冊為 **Lazy-loaded MCP Server**（頂層工具清單中不會出現 `mcp_*` 原生前綴）。Agent **絕對不得誤判為「未註冊工具」**，而必須透過 **`call_mcp_tool`** 進行調用：
  - `ServerName`: `"comfyui-local"` 或 `"ai-pose-gallery"`
  - `ToolName`: 對應工具名稱（例如 `search_poses`、`search_costumes`、`generate_image`、`validate_workflow_json` 等）
  - `Arguments`: 對應工具的參數字典
- **工具綱要查閱**：呼叫前若需確認工具參數定義，可先讀取 `<appDataDir>/mcp/<serverName>/<toolName>.json` 查閱 Schema。

## 1. 依任務選擇工作流與工具

| 使用者需求 | 處理流程 |
| --- | --- |
| **一般文生圖（SDXL / Illustrious）** | 讀取 `workflows/sdxl-txt2img.api.json`，先用 `comfyui-local` 的 `search_models` 確認已安裝的 checkpoint 名稱填入節點 `"1"`，修改正負向提示詞與解析度。 |
| **驗證／使用 LoRA 生圖** | 讀取 `workflows/sdxl-lora.api.json`，用 `search_models` 確認 `checkpoints` 與 `loras` 檔名填入節點 `"1"` 與 `"2"`，將 trigger word 置於正向提示詞首位。 |
| **找姿勢／服裝／人像參考圖合成角色** | 優先使用 `ai-pose-gallery` MCP：`search_portraits` / `search_costumes` / `search_poses` 挑選候選給使用者確認後，直接呼叫 `generate_image`（預設使用 `FLUX 3 Image Klein Distilled`，亦可傳入 `workflow_engine="qwen-edit"` 切換；若需自訂雙角色或多圖合成，優先讀取預設模板 `workflows/flux3-klein-edit.api.json`，備選為 `workflows/qwen-image-edit-2509.api.json`）。 |

## 2. 驗證與執行 Workflow（優先使用 JSON 直傳工具）

因為 `comfyui-local` 運行於獨立 Docker 容器，驗證與執行自訂或修改後的 workflow 時，**優先使用直傳 JSON 物件的工具**，不需依賴宿主機暫存檔路徑：

1. **驗證**：呼叫 `validate_workflow_json(workflow=<dict>)`，確認回傳 `valid: true` 且 `error_count: 0`。
2. **執行**：呼叫 `run_workflow_json(workflow=<dict>, wait=False)` 取得 `prompt_id`，再呼叫 `job(action="wait", prompt_id=...)` 等待完成（長任務可重複 wait）。生成結果位於 ComfyUI 的 `/data/comfyui/output`。
   - 若需將參考圖上傳至 ComfyUI `input/`，可將圖片放進 `tmp/` 並呼叫 `upload_file`（或使用 `ai-pose-gallery` 的 `prepare_comfyui_workflow` / `generate_image`）。

## 3. 驗證錯誤與缺模型／缺節點處理

| `code` | 處理方式 |
| --- | --- |
| `unknown_class_type` | 缺少 custom node，見下方「補裝 node」 |
| `unknown_enum_value` | 檢查 `suggestions`：若有相近模型路徑則修正 workflow 內的檔名；若無才詢問使用者是否下載 |
| `no_options_available` | 該模型資料夾為空，下載模型至對應子目錄 |

- **下載模型（`download_model`）**：
  - `relative_path` 第一段必須是 `models`（例如 `models/checkpoints`、`models/loras`、`models/unet`）。
  - 下載前**必須先查明檔案大小與來源並詢問使用者**；若檔案較大，也可提供指令讓使用者在終端機自行下載。
- **補裝 node**：
  - 臨時試用：用 `docker exec` 在 ComfyUI 容器內 `git clone --depth 1 <repo>` 到 `/app/ComfyUI/custom_nodes`（若有 `requirements.txt` 則 `pip install -r`），再 `docker restart comfyui`，待 `/system_stats` 回 200 後重新驗證。
  - 長期保留：回報使用者，經同意後再加入 [kime541200/comfyui-deploy](https://github.com/kime541200/comfyui-deploy) 的 `nodes.yaml` 與對應 profile。
