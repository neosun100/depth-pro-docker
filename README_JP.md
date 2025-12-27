[English](README.md) | [简体中文](README_CN.md) | [繁體中文](README_TW.md) | [日本語](README_JP.md)

<div align="center">

# 🔬 Depth Pro Docker

[![Docker](https://img.shields.io/badge/Docker-Ready-2496ED?logo=docker)](https://hub.docker.com/r/neosun/depth-pro)
[![License](https://img.shields.io/badge/License-Apple%20Sample%20Code-blue)](LICENSE)
[![Python](https://img.shields.io/badge/Python-3.9+-3776AB?logo=python)](https://python.org)
[![CUDA](https://img.shields.io/badge/CUDA-12.1-76B900?logo=nvidia)](https://developer.nvidia.com/cuda-toolkit)

**Apple Depth Pro モデルの本番環境向け Docker デプロイメント**

*ゼロショット単眼深度推定 • 0.3秒で2.25MP深度マップ生成*

![Screenshot](docs/screenshot.png)

</div>

---

## ✨ 機能

| 機能 | 説明 |
|------|------|
| 🚀 **ワンクリックデプロイ** | Docker Compose で即座にデプロイ |
| 🎨 **モダン Web UI** | 美しいインターフェース、複数のカラーマップ |
| 🔌 **REST API** | 完全な API、Swagger ドキュメント |
| 🤖 **MCP サーバー** | AI アシスタント対応 (Claude Desktop) |
| 📊 **複数出力形式** | JPG 可視化、NPZ データ、16-bit PNG |
| 🎛️ **手動焦点距離** | 自動焦点距離推定を上書き可能 |
| 🌐 **多言語対応** | 中国語、英語、日本語 UI |
| 💾 **GPU 管理** | 自動メモリ解放、状態監視 |

## 🚀 クイックスタート

```bash
# 1コマンドで起動 (All-in-One イメージ、モデルダウンロード不要！)
docker run -d --name depth-pro --gpus all -p 8500:8500 neosun/depth-pro:latest

# ブラウザを開く
open http://localhost:8500
```

## 📦 インストール

### 前提条件

- Docker 24.0+ と NVIDIA Container Toolkit
- NVIDIA GPU、VRAM 8GB+ (16GB+ 推奨)
- CUDA 12.1 互換ドライバー

### 方法1: Docker Run（推奨）

**All-in-One イメージにはモデル重み (~5GB) が含まれています。追加ダウンロード不要！**

```bash
# プルして実行 (モデル内蔵)
docker run -d \
  --name depth-pro \
  --gpus all \
  -p 8500:8500 \
  -e GPU_IDLE_TIMEOUT=60 \
  neosun/depth-pro:latest
```

### 方法2: Docker Compose

```bash
# docker-compose.yml を作成
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

# サービスを起動
docker compose up -d
```

### 方法3: ローカル開発

```bash
# conda 環境を作成
conda create -n depth-pro python=3.9 -y
conda activate depth-pro

# 依存関係をインストール
pip install -e .
pip install flask flask-cors flasgger gunicorn

# モデルをダウンロード
source get_pretrained_models.sh

# サーバーを実行
python app.py
```

## ⚙️ 設定

### 環境変数

| 変数 | デフォルト | 説明 |
|------|------------|------|
| `PORT` | `8500` | サーバーポート |
| `GPU_IDLE_TIMEOUT` | `60` | GPU メモリ解放までの秒数 |
| `NVIDIA_VISIBLE_DEVICES` | `0` | GPU デバイスインデックス |

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

### Web インターフェース

`http://localhost:8500` にアクセスしてインタラクティブ UI を使用：

1. 画像をアップロード (JPG/PNG/WebP/HEIC)
2. カラーマップを選択 (Turbo, Viridis, Plasma など)
3. オプションで手動焦点距離を設定
4. 「処理開始」をクリックして結果をダウンロード

### REST API

#### 深度推定

```bash
curl -X POST http://localhost:8500/api/predict \
  -F "file=@image.jpg" \
  -F "colormap=turbo" \
  -F "focal_length=1000"
```

レスポンス：
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

#### GPU ステータス

```bash
curl http://localhost:8500/api/gpu/status
```

#### GPU メモリ解放

```bash
curl -X POST http://localhost:8500/api/gpu/offload
```

### API ドキュメント

Swagger UI: `http://localhost:8500/apidocs/`

### MCP サーバー（AI アシスタント統合）

Claude Desktop 設定に追加：

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

利用可能な MCP ツール：
- `estimate_depth` - 単一画像を処理
- `batch_estimate_depth` - 複数画像をバッチ処理
- `get_gpu_status` - GPU ステータスを確認
- `release_gpu` - GPU メモリを解放

## 📁 プロジェクト構造

```
depth-pro-docker/
├── app.py                 # Flask Web サーバー
├── mcp_server.py          # MCP サーバー
├── gpu_manager.py         # GPU メモリ管理
├── Dockerfile             # コンテナビルドファイル
├── docker-compose.yml     # Docker Compose 設定
├── checkpoints/           # モデル重み（別途ダウンロード）
│   └── depth_pro.pt
├── src/depth_pro/         # コアモデルコード
├── templates/             # HTML テンプレート
├── static/                # CSS/JS アセット
└── docs/                  # ドキュメント
```

## 🛠️ 技術スタック

- **モデル**: Apple Depth Pro (DINOv2 + Multi-scale ViT)
- **バックエンド**: Flask + Gunicorn
- **フロントエンド**: Vanilla JS + Modern CSS
- **コンテナ**: Docker + NVIDIA Container Toolkit
- **GPU**: PyTorch + CUDA 12.1

## 📝 制限事項

- 遠景シーン (>20m) では絶対深度値が不正確な場合があります
- 屋内および近距離の屋外シーンに最適
- 遠景でも相対的な深度順序は一般的に信頼できます

## 🤝 コントリビューション

コントリビューション歓迎！まず [CONTRIBUTING.md](CONTRIBUTING.md) をお読みください。

1. リポジトリをフォーク
2. フィーチャーブランチを作成 (`git checkout -b feature/amazing`)
3. 変更をコミット (`git commit -m 'Add amazing feature'`)
4. ブランチをプッシュ (`git push origin feature/amazing`)
5. Pull Request を作成

## 📄 ライセンス

このプロジェクトは [Apple Depth Pro](https://github.com/apple/ml-depth-pro) に基づいており、[Apple Sample Code License](LICENSE) の下でライセンスされています。

## 🙏 謝辞

- [Apple ML Research](https://github.com/apple/ml-depth-pro) - オリジナル Depth Pro モデル
- [Depth Pro 論文](https://arxiv.org/abs/2410.02073) - 研究論文

---

## ⭐ Star History

[![Star History Chart](https://api.star-history.com/svg?repos=neosun100/depth-pro-docker&type=Date)](https://star-history.com/#neosun100/depth-pro-docker)

## 📱 フォローする

<div align="center">

![WeChat](https://img.aws.xin/uPic/扫码_搜索联合传播样式-标准色版.png)

</div>
