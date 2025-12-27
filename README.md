[English](README.md) | [简体中文](README_CN.md) | [繁體中文](README_TW.md) | [日本語](README_JP.md)

<div align="center">

# 🔬 Depth Pro Docker

[![Docker](https://img.shields.io/badge/Docker-Ready-2496ED?logo=docker)](https://hub.docker.com/r/neosun/depth-pro)
[![License](https://img.shields.io/badge/License-Apple%20Sample%20Code-blue)](LICENSE)
[![Python](https://img.shields.io/badge/Python-3.9+-3776AB?logo=python)](https://python.org)
[![CUDA](https://img.shields.io/badge/CUDA-12.1-76B900?logo=nvidia)](https://developer.nvidia.com/cuda-toolkit)

**Production-ready Docker deployment for Apple's Depth Pro model**

*Zero-shot monocular metric depth estimation • 2.25MP depth map in 0.3s*

![Screenshot](docs/screenshot.png)

</div>

---

## ✨ Features

| Feature | Description |
|---------|-------------|
| 🚀 **One-Click Deploy** | Docker Compose for instant deployment |
| 🎨 **Modern Web UI** | Beautiful interface with multiple colormaps |
| 🔌 **REST API** | Full-featured API with Swagger docs |
| 🤖 **MCP Server** | Model Context Protocol support for AI assistants |
| 📊 **Multiple Outputs** | JPG visualization, NPZ data, 16-bit PNG |
| 🎛️ **Manual Focal Length** | Override auto focal length estimation |
| 🌐 **Multi-language** | Chinese, English, Japanese UI |
| 💾 **GPU Management** | Auto memory offload, status monitoring |

## 🚀 Quick Start

```bash
# One command to run (All-in-One image, no downloads needed!)
docker run -d --name depth-pro --gpus all -p 8500:8500 neosun/depth-pro:latest

# Open browser
open http://localhost:8500
```

## 📦 Installation

### Prerequisites

- Docker 24.0+ with NVIDIA Container Toolkit
- NVIDIA GPU with 8GB+ VRAM (16GB+ recommended)
- CUDA 12.1 compatible driver

### Method 1: Docker Run (Recommended)

**All-in-One image includes model weights (~5GB), no additional downloads required!**

```bash
# Pull and run (model included in image)
docker run -d \
  --name depth-pro \
  --gpus all \
  -p 8500:8500 \
  -e GPU_IDLE_TIMEOUT=60 \
  neosun/depth-pro:latest
```

### Method 2: Docker Compose

```bash
# Create docker-compose.yml
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

# Start service
docker compose up -d
```

### Method 3: Local Development

```bash
# Create conda environment
conda create -n depth-pro python=3.9 -y
conda activate depth-pro

# Install dependencies
pip install -e .
pip install flask flask-cors flasgger gunicorn

# Download model
source get_pretrained_models.sh

# Run server
python app.py
```

## ⚙️ Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `8500` | Server port |
| `GPU_IDLE_TIMEOUT` | `60` | Seconds before GPU memory release |
| `NVIDIA_VISIBLE_DEVICES` | `0` | GPU device index |

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

## 📖 Usage

### Web Interface

Visit `http://localhost:8500` for the interactive UI:

1. Upload an image (JPG/PNG/WebP/HEIC)
2. Select colormap (Turbo, Viridis, Plasma, etc.)
3. Optionally set manual focal length
4. Click "Process" and download results

### REST API

#### Depth Estimation

```bash
curl -X POST http://localhost:8500/api/predict \
  -F "file=@image.jpg" \
  -F "colormap=turbo" \
  -F "focal_length=1000"
```

Response:
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

#### GPU Status

```bash
curl http://localhost:8500/api/gpu/status
```

#### Release GPU Memory

```bash
curl -X POST http://localhost:8500/api/gpu/offload
```

### API Documentation

Swagger UI available at: `http://localhost:8500/apidocs/`

### MCP Server (for AI Assistants)

Add to your Claude Desktop config:

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

Available MCP tools:
- `estimate_depth` - Process single image
- `batch_estimate_depth` - Process multiple images
- `get_gpu_status` - Check GPU status
- `release_gpu` - Free GPU memory

## 📁 Project Structure

```
depth-pro-docker/
├── app.py                 # Flask web server
├── mcp_server.py          # MCP server for AI assistants
├── gpu_manager.py         # GPU memory management
├── Dockerfile             # Container build file
├── docker-compose.yml     # Docker Compose config
├── checkpoints/           # Model weights (download separately)
│   └── depth_pro.pt
├── src/depth_pro/         # Core model code
├── templates/             # HTML templates
├── static/                # CSS/JS assets
└── docs/                  # Documentation
```

## 🛠️ Tech Stack

- **Model**: Apple Depth Pro (DINOv2 + Multi-scale ViT)
- **Backend**: Flask + Gunicorn
- **Frontend**: Vanilla JS + Modern CSS
- **Container**: Docker + NVIDIA Container Toolkit
- **GPU**: PyTorch + CUDA 12.1

## 📝 Limitations

- Far-field scenes (>20m) may have inaccurate absolute depth values
- Best suited for indoor and close-range outdoor scenes
- Relative depth ordering is generally reliable even for far scenes

## 🤝 Contributing

Contributions are welcome! Please read [CONTRIBUTING.md](CONTRIBUTING.md) first.

1. Fork the repository
2. Create feature branch (`git checkout -b feature/amazing`)
3. Commit changes (`git commit -m 'Add amazing feature'`)
4. Push to branch (`git push origin feature/amazing`)
5. Open a Pull Request

## 📄 License

This project is based on [Apple's Depth Pro](https://github.com/apple/ml-depth-pro) and is licensed under the [Apple Sample Code License](LICENSE).

## 🙏 Acknowledgements

- [Apple ML Research](https://github.com/apple/ml-depth-pro) - Original Depth Pro model
- [Depth Pro Paper](https://arxiv.org/abs/2410.02073) - Research paper

---

## ⭐ Star History

[![Star History Chart](https://api.star-history.com/svg?repos=neosun100/depth-pro-docker&type=Date)](https://star-history.com/#neosun100/depth-pro-docker)

## 📱 Follow Me

<div align="center">

![WeChat](https://img.aws.xin/uPic/扫码_搜索联合传播样式-标准色版.png)

</div>
