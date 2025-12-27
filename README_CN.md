[English](README.md) | [简体中文](README_CN.md) | [繁體中文](README_TW.md) | [日本語](README_JP.md)

<div align="center">

# 🔬 Depth Pro Docker

[![Docker](https://img.shields.io/badge/Docker-Ready-2496ED?logo=docker)](https://hub.docker.com/r/neosun/depth-pro)
[![License](https://img.shields.io/badge/License-Apple%20Sample%20Code-blue)](LICENSE)
[![Python](https://img.shields.io/badge/Python-3.9+-3776AB?logo=python)](https://python.org)
[![CUDA](https://img.shields.io/badge/CUDA-12.1-76B900?logo=nvidia)](https://developer.nvidia.com/cuda-toolkit)

**Apple Depth Pro 模型的生产级 Docker 部署方案**

*零样本单目深度估计 • 0.3秒生成 2.25MP 深度图*

![Screenshot](docs/screenshot.png)

</div>

---

## ✨ 功能特性

| 功能 | 说明 |
|------|------|
| 🚀 **一键部署** | Docker Compose 快速启动 |
| 🎨 **现代化 Web UI** | 精美界面，多种颜色映射 |
| 🔌 **REST API** | 完整 API，Swagger 文档 |
| 🤖 **MCP 服务器** | 支持 AI 助手调用 (Claude Desktop) |
| 📊 **多种输出** | JPG 可视化、NPZ 数据、16-bit PNG |
| 🎛️ **手动焦距** | 可覆盖自动焦距估计 |
| 🌐 **多语言** | 中文、英文、日文界面 |
| 💾 **GPU 管理** | 自动显存释放、状态监控 |

## 🚀 快速开始

```bash
# 一条命令启动 (All-in-One 镜像，无需下载模型！)
docker run -d --name depth-pro --gpus all -p 8500:8500 neosun/depth-pro:latest

# 打开浏览器
open http://localhost:8500
```

## 📦 安装部署

### 前置条件

- Docker 24.0+ 及 NVIDIA Container Toolkit
- NVIDIA GPU，显存 8GB+ (推荐 16GB+)
- CUDA 12.1 兼容驱动

### 方式一：Docker Run（推荐）

**All-in-One 镜像已包含模型权重 (~5GB)，无需额外下载！**

```bash
# 拉取并运行 (模型已内置)
docker run -d \
  --name depth-pro \
  --gpus all \
  -p 8500:8500 \
  -e GPU_IDLE_TIMEOUT=60 \
  neosun/depth-pro:latest
```

### 方式二：Docker Compose

```bash
# 创建 docker-compose.yml
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

# 启动服务
docker compose up -d
```

### 方式三：本地开发

```bash
# 创建 conda 环境
conda create -n depth-pro python=3.9 -y
conda activate depth-pro

# 安装依赖
pip install -e .
pip install flask flask-cors flasgger gunicorn

# 下载模型
source get_pretrained_models.sh

# 运行服务
python app.py
```

## ⚙️ 配置说明

### 环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `PORT` | `8500` | 服务端口 |
| `GPU_IDLE_TIMEOUT` | `60` | 空闲多少秒后释放显存 |
| `NVIDIA_VISIBLE_DEVICES` | `0` | GPU 设备索引 |

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

### Web 界面

访问 `http://localhost:8500` 使用交互式界面：

1. 上传图像 (JPG/PNG/WebP/HEIC)
2. 选择颜色映射 (Turbo, Viridis, Plasma 等)
3. 可选设置手动焦距
4. 点击"开始处理"并下载结果

### REST API

#### 深度估计

```bash
curl -X POST http://localhost:8500/api/predict \
  -F "file=@image.jpg" \
  -F "colormap=turbo" \
  -F "focal_length=1000"
```

响应：
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

#### GPU 状态

```bash
curl http://localhost:8500/api/gpu/status
```

#### 释放显存

```bash
curl -X POST http://localhost:8500/api/gpu/offload
```

### API 文档

Swagger UI 地址：`http://localhost:8500/apidocs/`

### MCP 服务器（AI 助手集成）

添加到 Claude Desktop 配置：

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
- `estimate_depth` - 处理单张图像
- `batch_estimate_depth` - 批量处理图像
- `get_gpu_status` - 查看 GPU 状态
- `release_gpu` - 释放显存

## 📁 项目结构

```
depth-pro-docker/
├── app.py                 # Flask Web 服务
├── mcp_server.py          # MCP 服务器
├── gpu_manager.py         # GPU 显存管理
├── Dockerfile             # 容器构建文件
├── docker-compose.yml     # Docker Compose 配置
├── checkpoints/           # 模型权重（需单独下载）
│   └── depth_pro.pt
├── src/depth_pro/         # 核心模型代码
├── templates/             # HTML 模板
├── static/                # CSS/JS 资源
└── docs/                  # 文档
```

## 🛠️ 技术栈

- **模型**: Apple Depth Pro (DINOv2 + Multi-scale ViT)
- **后端**: Flask + Gunicorn
- **前端**: Vanilla JS + Modern CSS
- **容器**: Docker + NVIDIA Container Toolkit
- **GPU**: PyTorch + CUDA 12.1

## 📝 使用限制

- 远景场景 (>20m) 的绝对深度值可能不准确
- 最适合室内和近距离户外场景
- 即使远景，相对深度排序通常仍然可靠

## 🤝 参与贡献

欢迎贡献！请先阅读 [CONTRIBUTING.md](CONTRIBUTING.md)。

1. Fork 本仓库
2. 创建特性分支 (`git checkout -b feature/amazing`)
3. 提交更改 (`git commit -m 'Add amazing feature'`)
4. 推送分支 (`git push origin feature/amazing`)
5. 发起 Pull Request

## 📄 许可证

本项目基于 [Apple Depth Pro](https://github.com/apple/ml-depth-pro)，采用 [Apple Sample Code License](LICENSE)。

## 🙏 致谢

- [Apple ML Research](https://github.com/apple/ml-depth-pro) - 原始 Depth Pro 模型
- [Depth Pro 论文](https://arxiv.org/abs/2410.02073) - 研究论文

---

## ⭐ Star History

[![Star History Chart](https://api.star-history.com/svg?repos=neosun100/depth-pro-docker&type=Date)](https://star-history.com/#neosun100/depth-pro-docker)

## 📱 关注公众号

<div align="center">

![WeChat](https://img.aws.xin/uPic/扫码_搜索联合传播样式-标准色版.png)

</div>
