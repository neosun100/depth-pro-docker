[English](README.md) | [简体中文](README_CN.md) | [繁體中文](README_TW.md) | [日本語](README_JP.md)

<div align="center">

# 🔬 Depth Pro Docker

[![Docker](https://img.shields.io/badge/Docker-Ready-2496ED?logo=docker)](https://hub.docker.com/r/neosun/depth-pro)
[![License](https://img.shields.io/badge/License-Apple%20Sample%20Code-blue)](LICENSE)
[![Python](https://img.shields.io/badge/Python-3.9+-3776AB?logo=python)](https://python.org)
[![CUDA](https://img.shields.io/badge/CUDA-12.1-76B900?logo=nvidia)](https://developer.nvidia.com/cuda-toolkit)

**Apple Depth Pro 模型的生產級 Docker 部署方案**

*零樣本單目深度估計 • 0.3秒生成 2.25MP 深度圖*

![Screenshot](docs/screenshot.png)

</div>

---

## ✨ 功能特性

| 功能 | 說明 |
|------|------|
| 🚀 **一鍵部署** | Docker Compose 快速啟動 |
| 🎨 **現代化 Web UI** | 精美介面，多種顏色映射 |
| 🔌 **REST API** | 完整 API，Swagger 文檔 |
| 🤖 **MCP 伺服器** | 支援 AI 助手調用 (Claude Desktop) |
| 📊 **多種輸出** | JPG 視覺化、NPZ 資料、16-bit PNG |
| 🎛️ **手動焦距** | 可覆蓋自動焦距估計 |
| 🌐 **多語言** | 中文、英文、日文介面 |
| 💾 **GPU 管理** | 自動顯存釋放、狀態監控 |

## 🚀 快速開始

```bash
# 一條命令啟動 (All-in-One 映像，無需下載模型！)
docker run -d --name depth-pro --gpus all -p 8500:8500 neosun/depth-pro:latest

# 開啟瀏覽器
open http://localhost:8500
```

## 📦 安裝部署

### 前置條件

- Docker 24.0+ 及 NVIDIA Container Toolkit
- NVIDIA GPU，顯存 8GB+ (推薦 16GB+)
- CUDA 12.1 相容驅動

### 方式一：Docker Run（推薦）

**All-in-One 映像已包含模型權重 (~5GB)，無需額外下載！**

```bash
# 拉取並執行 (模型已內建)
docker run -d \
  --name depth-pro \
  --gpus all \
  -p 8500:8500 \
  -e GPU_IDLE_TIMEOUT=60 \
  neosun/depth-pro:latest
```

### 方式二：Docker Compose

```bash
# 建立 docker-compose.yml
cat > docker-compose.yml << 'EOF'
services:
  depth-pro:
    image: neosun/depth-pro:latest
    container_name: depth-pro
    ports:
      - "8500:8500"
    environment:
      - GPU_IDLE_TIMEOUT=60
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    restart: unless-stopped
EOF

# 啟動服務
docker compose up -d
```

### 方式三：本地開發

```bash
# 建立 conda 環境
conda create -n depth-pro python=3.9 -y
conda activate depth-pro

# 安裝依賴
pip install -e .
pip install flask flask-cors flasgger gunicorn

# 下載模型
source get_pretrained_models.sh

# 執行服務
python app.py
```

## ⚙️ 配置說明

### 環境變數

| 變數 | 預設值 | 說明 |
|------|--------|------|
| `PORT` | `8500` | 服務埠號 |
| `GPU_IDLE_TIMEOUT` | `60` | 閒置多少秒後釋放顯存 |
| `NVIDIA_VISIBLE_DEVICES` | `0` | GPU 裝置索引 |

### docker-compose.yml

```yaml
services:
  depth-pro:
    image: neosun/depth-pro:latest
    container_name: depth-pro
    ports:
      - "8500:8500"
    environment:
      - PORT=8500
      - GPU_IDLE_TIMEOUT=60
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8500/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

## 📖 使用方法

### Web 介面

訪問 `http://localhost:8500` 使用互動式介面：

1. 上傳圖像 (JPG/PNG/WebP/HEIC)
2. 選擇顏色映射 (Turbo, Viridis, Plasma 等)
3. 可選設定手動焦距
4. 點擊「開始處理」並下載結果

### REST API

#### 深度估計

```bash
curl -X POST http://localhost:8500/api/predict \
  -F "file=@image.jpg" \
  -F "colormap=turbo" \
  -F "focal_length=1000"
```

回應：
```json
{
  "task_id": "abc12345",
  "focal_length_px": 1000.0,
  "min_depth_m": 0.5,
  "max_depth_m": 10.2,
  "mean_depth_m": 3.4,
  "image_size": "1920x1080",
  "depth_image_base64": "...",
  "download_jpg": "/api/download/abc12345/color.jpg",
  "download_npz": "/api/download/abc12345/depth.npz",
  "download_16bit": "/api/download/abc12345/depth16.png"
}
```

#### GPU 狀態

```bash
curl http://localhost:8500/api/gpu/status
```

#### 釋放顯存

```bash
curl -X POST http://localhost:8500/api/gpu/offload
```

### API 文檔

Swagger UI 地址：`http://localhost:8500/apidocs/`

### MCP 伺服器（AI 助手整合）

新增至 Claude Desktop 配置：

```json
{
  "mcpServers": {
    "depth-pro": {
      "command": "docker",
      "args": ["exec", "-i", "depth-pro", "python3", "mcp_server.py"]
    }
  }
}
```

可用 MCP 工具：
- `estimate_depth` - 處理單張圖像
- `batch_estimate_depth` - 批次處理圖像
- `get_gpu_status` - 查看 GPU 狀態
- `release_gpu` - 釋放顯存

## 📁 專案結構

```
depth-pro-docker/
├── app.py                 # Flask Web 服務
├── mcp_server.py          # MCP 伺服器
├── gpu_manager.py         # GPU 顯存管理
├── Dockerfile             # 容器建構檔案
├── docker-compose.yml     # Docker Compose 配置
├── checkpoints/           # 模型權重（需單獨下載）
│   └── depth_pro.pt
├── src/depth_pro/         # 核心模型程式碼
├── templates/             # HTML 範本
├── static/                # CSS/JS 資源
└── docs/                  # 文檔
```

## 🛠️ 技術棧

- **模型**: Apple Depth Pro (DINOv2 + Multi-scale ViT)
- **後端**: Flask + Gunicorn
- **前端**: Vanilla JS + Modern CSS
- **容器**: Docker + NVIDIA Container Toolkit
- **GPU**: PyTorch + CUDA 12.1

## 📝 使用限制

- 遠景場景 (>20m) 的絕對深度值可能不準確
- 最適合室內和近距離戶外場景
- 即使遠景，相對深度排序通常仍然可靠

## 🤝 參與貢獻

歡迎貢獻！請先閱讀 [CONTRIBUTING.md](CONTRIBUTING.md)。

1. Fork 本倉庫
2. 建立特性分支 (`git checkout -b feature/amazing`)
3. 提交更改 (`git commit -m 'Add amazing feature'`)
4. 推送分支 (`git push origin feature/amazing`)
5. 發起 Pull Request

## 📄 授權條款

本專案基於 [Apple Depth Pro](https://github.com/apple/ml-depth-pro)，採用 [Apple Sample Code License](LICENSE)。

## 🙏 致謝

- [Apple ML Research](https://github.com/apple/ml-depth-pro) - 原始 Depth Pro 模型
- [Depth Pro 論文](https://arxiv.org/abs/2410.02073) - 研究論文

---

## ⭐ Star History

[![Star History Chart](https://api.star-history.com/svg?repos=neosun100/depth-pro-docker&type=Date)](https://star-history.com/#neosun100/depth-pro-docker)

## 📱 關注公眾號

<div align="center">

![WeChat](https://img.aws.xin/uPic/扫码_搜索联合传播样式-标准色版.png)

</div>
