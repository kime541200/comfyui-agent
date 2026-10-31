# comfyui-agent：Agent 生圖工作區（Studio Workspace）

開箱即用的 AI 繪圖 Agent 工作區。使用者只需在本目錄啟動 Coding Agent session，Agent 即可根據需求自動查詢模型、套用工作流模板、挑選姿勢／服裝參考圖並驅動本地 ComfyUI 生成圖片。

## 架構與搭配服務

本工作區透過 **Streamable HTTP MCP** 串接以下容器服務（不需在本目錄安裝 Python `.venv` 或建立偽 workspace）：

1. **ComfyUI 引擎（必備）**：[kime541200/comfyui-deploy](https://github.com/kime541200/comfyui-deploy)（預設 `http://127.0.0.1:8188`）。
2. **`comfyui-local` MCP Server（必備）**：[kime541200/comfy-mcp](https://github.com/kime541200/comfy-mcp)（預設 `http://127.0.0.1:8091/mcp`），負責節點／模型探索、Workflow 驗證與執行、模型下載。
3. **`ai-pose-gallery` MCP Server（選配）**：[kime541200/ai-pose-gallery-mcp](https://github.com/kime541200/ai-pose-gallery-mcp)（預設 `http://127.0.0.1:8090/mcp`），搭配 [kime541200/ai-pose-gallery-zh](https://github.com/kime541200/ai-pose-gallery-zh) 圖庫提供姿勢／服裝／人像檢索與多圖合成。

## 內建資源

- `workflows/`：已驗證的 ComfyUI API 格式工作流模板：
  - `workflows/sdxl-txt2img.api.json`：SDXL / Illustrious 1024 級標準文生圖。
  - `workflows/sdxl-lora.api.json`：SDXL / Illustrious 載入 LoRA 驗證與生圖（可搭配 [kime541200/lora-training](https://github.com/kime541200/lora-training) 產出的模型）。
  - `workflows/qwen-image-edit-2509.api.json`：Qwen-Image-Edit-2509 三圖參考（人像＋服裝＋姿勢 DWPose 骨架）合成工作流。
- `mcp_config.example.json`：預先配置好 `comfyui-local` 與 `ai-pose-gallery` 的 Streamable HTTP MCP 設定範本。
- `docs/setup.md`：環境建置與連線檢查說明。
- `AGENTS.md`：Agent 操作 ComfyUI 與工作流模板的完整規則。
