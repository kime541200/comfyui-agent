# 環境建置

前提：`comfyui-deploy` 的容器已啟動，8188 發布於 `127.0.0.1`；已安裝 `uv`。

## 1. 安裝 MCP 工具

```bash
uv venv .venv
uv pip install --python .venv/bin/python comfy-mcp "comfy-cli>=1.14"
uv pip list --python .venv/bin/python | grep -E "comfy-cli|comfy-mcp"   # 應列出兩者
```

## 2. 偽 workspace（供模型下載）

`comfy-cli model download` 需要一個有 ComfyUI 特徵的目錄，下載落在 `<STUB>/models`。把它指向共用的模型目錄：

```bash
STUB=~/comfy/ComfyUI
mkdir -p $STUB/comfy $STUB/comfy_extras
touch $STUB/main.py $STUB/nodes.py
ln -sfn /data/comfyui/models $STUB/models
.venv/bin/comfy set-default $STUB
cat ~/.config/comfy-cli/config.ini   # default_workspace = <STUB>
```

## 3. MCP 設定

`.agents/mcp_config.json` 是範本，用標準 `mcpServers` 格式。不同 harness 讀取的位置不同（例如專案根的 `.mcp.json` 或各自的設定目錄）；把同一份內容放到你所用 harness 的專案層設定即可，並重啟 harness 載入。

- `command` 與 `COMFY_BIN`：本目錄 `.venv` 內的絕對路徑；搬動目錄後更新。
- `COMFYUI_URL=http://127.0.0.1:8188`：ComfyUI 容器位址。
- `COMFY_MCP_REMOTE_SHARED_MODELS=1`：讓 MCP 允許下載（未設則偵測到 `COMFYUI_URL` 就拒絕）。

## 行為背景

- **偽 workspace**：`comfy-cli` 以 `main.py`、`nodes.py`、`comfy/` 判斷 workspace，直接指向 `/data/comfyui` 會報 `not_in_workspace`。
- **只連 `127.0.0.1:8188`**：`nodes search`、`system_stats` 走 `comfy-cli` 預設 host，不走 `COMFYUI_URL`，所以容器必須把 8188 發布在 host。
- **`install_node`／`workflow_deps` 回 `unsupported`**：它們經 host `comfy-cli` 的 ComfyUI-Manager 操作 host workspace，而偽 workspace 沒有 Manager；即使可用也裝在 host 而非容器。不要把 Manager 裝進偽 workspace。
