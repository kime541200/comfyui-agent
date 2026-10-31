# 環境建置

前提（以下步驟可重複執行）：
- 已安裝 `uv` 與 `docker`。
- [kime541200/comfyui-deploy](https://github.com/kime541200/comfyui-deploy) 的容器已啟動：`curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8188/system_stats` 回 `200`。
- host 上存在 [kime541200/comfyui-deploy](https://github.com/kime541200/comfyui-deploy) 掛載的資料目錄：`test -d /data/comfyui/models && echo ok`。

## 1. 安裝 MCP 工具

```bash
uv venv .venv
uv pip install --python .venv/bin/python comfy-mcp "comfy-cli>=1.14"
uv pip list --python .venv/bin/python | grep -E "comfy-cli|comfy-mcp"   # 應列出兩者
```

## 2. 偽 workspace（供模型下載）

`comfy-cli model download` 需要一個有 ComfyUI 特徵的目錄，下載落在 `<STUB>/models`。把它指向 [kime541200/comfyui-deploy](https://github.com/kime541200/comfyui-deploy) 掛載的模型目錄（以下整段在同一個 shell 執行）：

```bash
STUB=~/comfy/ComfyUI
mkdir -p $STUB/comfy $STUB/comfy_extras
touch $STUB/main.py $STUB/nodes.py
ln -sfn /data/comfyui/models $STUB/models
.venv/bin/comfy set-default $STUB
readlink $STUB/models && ls $STUB/models | head -3   # 指向 /data/comfyui/models 並列出內容
comfy_cfg=$(find ~/.config/comfy-cli -name config.ini 2>/dev/null | head -1); grep default_workspace "$comfy_cfg"   # default_workspace = <STUB>
```

## 3. MCP 設定

`mcp_config.example.json` 是範本（標準 `mcpServers` 格式），`<REPO_DIR>` 為本目錄的絕對路徑。產生你自己的設定：

```bash
sed "s|<REPO_DIR>|$PWD|g" mcp_config.example.json > .agents/mcp_config.json
```

不同 harness 讀取專案層 MCP 設定的位置不同；把產生的內容放到你所用 harness 的位置（`.agents/mcp_config.json` 已被 git 忽略），然後重啟 harness 載入。

- `command` 與 `COMFY_BIN`：本目錄 `.venv` 內的可執行檔。
- `COMFYUI_URL=http://127.0.0.1:8188`：ComfyUI 容器位址。
- `COMFY_MCP_REMOTE_SHARED_MODELS=1`：讓 MCP 允許下載（未設則偵測到 `COMFYUI_URL` 就拒絕）。

驗證：重啟 harness 後，MCP 清單應出現 `comfyui-local`，呼叫 `system_stats` 應回傳 ComfyUI 版本。

## 行為背景

- **偽 workspace**：`comfy-cli` 以 `main.py`、`nodes.py`、`comfy/` 判斷 workspace，直接指向 `/data/comfyui` 會報 `not_in_workspace`。
- **只連 `127.0.0.1:8188`**：`nodes search`、`system_stats` 走 `comfy-cli` 預設 host，不走 `COMFYUI_URL`，所以容器必須把 8188 發布在 host。
- **`install_node`／`workflow_deps` 回 `unsupported`**：它們經 host `comfy-cli` 的 ComfyUI-Manager 操作 host workspace，而偽 workspace 沒有 Manager；即使可用也裝在 host 而非容器。不要把 Manager 裝進偽 workspace。
