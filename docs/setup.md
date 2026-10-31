# 環境建置與連線設定

本工作區採用純 **Streamable HTTP MCP** 架構，不需要在本機建立 Python `.venv` 或偽 `~/comfy/ComfyUI` 目錄。

## 1. 啟動後端服務

1. **ComfyUI 容器**（[kime541200/comfyui-deploy](https://github.com/kime541200/comfyui-deploy)）：
   ```bash
   curl -s http://127.0.0.1:8188/system_stats   # 確認回傳 200
   ```
2. **`comfy-mcp` 容器**（[kime541200/comfy-mcp](https://github.com/kime541200/comfy-mcp)，預設 Port `8091`）：
   ```bash
   curl -s http://127.0.0.1:8091/health         # 應回 {"status":"ok","service":"comfy-mcp",...}
   ```
3. **（選配）`ai-pose-gallery-mcp` 容器**（[kime541200/ai-pose-gallery-mcp](https://github.com/kime541200/ai-pose-gallery-mcp)，預設 Port `8090`，需搭配 [kime541200/ai-pose-gallery-zh](https://github.com/kime541200/ai-pose-gallery-zh)）：
   ```bash
   curl -s http://127.0.0.1:8090/health         # 應回 {"status":"ok","service":"ai-pose-gallery-mcp",...}
   ```

## 2. 設定 Agent MCP

將 `mcp_config.example.json` 的內容加入你使用的 Coding Agent 設定檔（例如全域 `~/.gemini/config/mcp_config.json` 或專案層 `.agents/mcp_config.json`）：

```json
{
  "mcpServers": {
    "comfyui-local": {
      "url": "http://127.0.0.1:8091/mcp"
    },
    "ai-pose-gallery": {
      "url": "http://127.0.0.1:8090/mcp"
    }
  }
}
```

> 若你的 MCP Server 部署於不同 Port，請將 URL 中的 `8091`／`8090` 改為實際埠號。設定完成後請重啟 Agent harness 載入工具。
