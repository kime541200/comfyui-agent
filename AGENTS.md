# comfyui-agent

讓 Agent 經 `comfy-mcp`（MCP server 名稱 `comfyui-local`）操作 [kime541200/comfyui-deploy](https://github.com/kime541200/comfyui-deploy) 啟動的 ComfyUI 容器：驗證 workflow、補齊 node 與模型、生成。環境建置（venv、偽 workspace、MCP 設定）見 `docs/setup.md`；MCP 無法連線、下載被拒、node 或模型找不到時先讀它。

## 生成

- 規劃 workflow 前，先用 MCP 的 `nodes` 與 `search_models` 工具查實際已安裝的 node 和模型，以此為準。
- 流程：`validate_workflow`（參數 `workflow_path` 是檔案路徑；`valid: true`、`error_count: 0`）→ `run_workflow(wait=false)` → `job(action="wait")`；久的工作重複呼叫 `wait`。輸出在 `/data/comfyui/output`。
- 文生圖用最小 workflow（`CheckpointLoaderSimple` → `CLIPTextEncode` ×2 → `EmptyLatentImage` → `KSampler` → `VAEDecode` → `SaveImage`）。SDXL／Illustrious 用 1024 級解析度（例 1216×832）、約 28 步、`euler_ancestral`。`generate_image` 預設是 SD1.5 範本，SDXL 系不適用。
- Civitai 頁面 Agent 讀不到：請使用者下載 `.json`，路徑放 `tmp/`（git 忽略）。有些 Civitai 頁面是 checkpoint 而非 workflow，下載到 `models/checkpoints` 即可。

## 驗證錯誤的處理

| `code` | 處理 |
| --- | --- |
| `unknown_class_type` | 缺 node，見「補裝 node」 |
| `unknown_enum_value` | 看 `suggestions`：有相近路徑就改 workflow 內的檔名，沒有才下載 |
| `no_options_available` | 該資料夾沒有模型，下載到對應資料夾 |

- `download_model` 的 `relative_path` 第一段必須是 `models`（如 `models/loras`）。下載前先確認檔案大小與來源，並讓使用者確認。
- 連遠端 ComfyUI 時 `download_model` 預設拒絕；`/data/comfyui/models` 實際是共用的，已於 MCP 設定 `COMFY_MCP_REMOTE_SHARED_MODELS=1`。

## 補裝 node

- 用 `docker exec` 在容器內 `git clone --depth 1 <repo>` 到 `/app/ComfyUI/custom_nodes`，有 `requirements.txt` 就 `pip install -r`，再 `docker restart comfyui`（容器名稱見 [kime541200/comfyui-deploy](https://github.com/kime541200/comfyui-deploy) 的 `docker-compose.yml`），等 `/system_stats` 回 200 再重新驗證。視為試用：容器重建即清除。
- `install_node`、`workflow_deps` 回報 unavailable／unsupported（見 `docs/setup.md`）。repo 網址從 [kime541200/comfyui-deploy](https://github.com/kime541200/comfyui-deploy) 的 `nodes.yaml` 或節點作者頁面查。
- 確定要長期使用的 node，回報使用者；加入 [kime541200/comfyui-deploy](https://github.com/kime541200/comfyui-deploy) 的 `nodes.yaml` 與 profile 前先問使用者。

## MCP

- 沒有 `comfyui-local` 工具、或呼叫 `system_stats` 失敗時，MCP 尚未就緒：先確認容器（`curl http://127.0.0.1:8188/system_stats`）與 `docs/setup.md` 的設定，把缺少的步驟告訴使用者；設定完成後，請使用者重啟 coding agent harness（Agent 無法自行重新載入）。
- `comfy-mcp` 等 stdio 輸入，直接執行會卡住；驗證安裝用 `uv pip list --python .venv/bin/python`，或透過 MCP 工具呼叫。
