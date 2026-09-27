# vllm-serve

用 Docker Compose 部署 [vLLM](https://github.com/vllm-project/vllm)，提供兩個 OpenAI 相容的 API 服務：

| 服務 | 容器名稱 | 本機位址 | 用途 |
|---|---|---|---|
| `vllm` | `vllm` | `127.0.0.1:8000` | 聊天 / 文字生成（Gemma 4，支援圖片輸入與 tool calling） |
| `embedding-gemma` | `embedding-gemma` | `127.0.0.1:8001` | 文字向量（Embedding） |

兩個服務都只綁定在 `127.0.0.1`，對外存取需要另外透過反向代理（例如 nginx）轉發。

## 需求

- NVIDIA GPU，且已安裝 [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
- Docker 與 Docker Compose v2
- Hugging Face 帳號與 access token（下載 Gemma 等需同意授權的模型時需要）

## 快速開始

```bash
# 1. 複製設定檔並填入實際值
cp .env.example .env
chmod 600 .env

# 2. 啟動（第一次會下載模型，需要一段時間）
docker compose up -d

# 3. 查看啟動進度
docker compose logs -f vllm
```

模型會下載到 `./hf-cache/`，之後重啟就不會重新下載。

## 設定（`.env`）

| 變數 | 說明 | 範例 |
|---|---|---|
| `HF_TOKEN` | Hugging Face access token | `hf_xxx` |
| `VLLM_API_KEY` | 呼叫 API 時使用的金鑰，客戶端要帶 `Authorization: Bearer <key>` | `sk-xxx` |
| `VLLM_IMAGE` | vLLM 容器映像檔 | `vllm/vllm-openai:latest` |
| `PUBLIC_BASE_URL` | 對外的 API 網址（僅供參考與測試用） | `https://example.com/llm` |
| **聊天模型** | | |
| `MODEL_ID` | Hugging Face 模型 ID | `nvidia/Gemma-4-26B-A4B-NVFP4` |
| `MAX_MODEL_LEN` | 最大 context 長度（tokens） | `262144` |
| `GPU_MEM_UTIL` | 可使用的 GPU 記憶體比例 | `0.75` |
| `DTYPE` | 權重精度 | `auto` |
| **Embedding 模型** | | |
| `EMBED_MODEL_ID` | Hugging Face 模型 ID | `google/embeddinggemma-300m` |
| `EMBED_MAX_MODEL_LEN` | 最大輸入長度（tokens） | `2048` |
| `EMBED_GPU_MEM_UTIL` | 可使用的 GPU 記憶體比例 | `0.05` |
| `EMBED_DTYPE` | 權重精度 | `float32` |
| `EMBED_MAX_NUM_BATCHED_TOKENS` | 單批次最多處理的 tokens 數 | `16384` |

兩個服務共用同一張 GPU，`GPU_MEM_UTIL` 加上 `EMBED_GPU_MEM_UTIL` 不能超過 `1.0`。

> `.env` 含有金鑰，已被 `.gitignore` 排除，請勿提交到 repo。

## 使用方式

### 健康檢查

```bash
curl http://localhost:8000/health
curl http://localhost:8001/health
```

### 聊天

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Authorization: Bearer $VLLM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/Gemma-4-26B-A4B-NVFP4",
    "messages": [{"role": "user", "content": "用一句話自我介紹"}],
    "max_tokens": 100
  }'
```

### Embedding

```bash
curl http://localhost:8001/v1/embeddings \
  -H "Authorization: Bearer $VLLM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "google/embeddinggemma-300m",
    "input": ["第一段文字", "第二段文字"]
  }'
```

### 使用 OpenAI Python SDK

```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8000/v1", api_key="<VLLM_API_KEY>")
resp = client.chat.completions.create(
    model="nvidia/Gemma-4-26B-A4B-NVFP4",
    messages=[{"role": "user", "content": "你好"}],
)
print(resp.choices[0].message.content)
```

## 常用指令

```bash
docker compose up -d                  # 啟動
docker compose down                   # 停止並移除容器
docker compose restart vllm           # 重啟單一服務（例如修改 .env 之後）
docker compose logs -f vllm           # 查看聊天服務 log
docker compose logs -f embedding-gemma
docker compose ps                     # 查看狀態與健康檢查結果
```

聊天模型啟動時需要載入大量權重，健康檢查給了 10 分鐘的啟動時間（`start_period: 600s`），在這段時間內顯示 `starting` 是正常的。

## 注意事項

- **換模型**：修改 `.env` 的 `MODEL_ID` 後執行 `docker compose up -d`。如果不是 Gemma 4 系列，要一併修改 `docker-compose.yml` 裡的 `--tool-call-parser`。
- **反向代理**：如果經過 nginx 等代理並使用串流輸出（`"stream": true`），要關閉代理的 buffering，並把讀取逾時調長，否則回應會卡住或被中途切斷。
