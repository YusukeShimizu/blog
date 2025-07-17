---
date: '2025-07-17T11:36:14+09:00'
title: "Python開発でのHot Reload環境構築：Docker + Watchdogによる爆速開発体験"
---

## はじめに

Pythonアプリケーション開発では、コードを変更するたびにコンテナを再起動するのは時間がかかって面倒ですよね。そこで今回は、ファイルを保存するだけで自動的にアプリケーションが再起動されるHot Reload環境を構築した話をまとめます。

この設定は、WebアプリケーションやAPI、バックグラウンドサービス、Bot開発など、様々なPythonアプリケーションに適用可能です。

## 技術スタック

- **言語**: Python 3.12
- **フレームワーク**: 任意のPythonフレームワーク（FastAPI、Flask、Django等）
- **コンテナ**: Docker + Docker Compose
- **ファイル監視**: Python watchdog + watchmedo
- **開発環境**: Ubuntu 24.04 (コンテナ内)

## Hot Reload環境の設計

### 1. Docker Compose設定

開発環境では`docker-compose.override.yml`を使用してHot Reload機能を有効化：

```yaml
version: '3.8'

services:
  app:
    build:
      dockerfile: Dockerfile.dev
    command: >
      watchmedo auto-restart
        --patterns='*.py'
        --recursive
        --signal SIGTERM
        --directory=/app
        --interval=0.5
        --ignore-patterns='*/__pycache__/*;*/.*/*;*/.git/*;*/node_modules/*;*/build/*;*/dist/*'
        --ignore-directories
        -- python -m your_app.main
    volumes:
      - .:/app
      - /app/node_modules
      - /app/.git
    environment:
      - PYTHONUNBUFFERED=1
      - PYTHONDONTWRITEBYTECODE=1
      - WATCHDOG_ENABLED=true
      - LOG_LEVEL=DEBUG
```

### 2. ファイル監視システム

`watchmedo`を使用した高速ファイル監視の特徴：

- **監視対象**: `.py`ファイルのみ
- **監視間隔**: 0.5秒（爆速検知）
- **再起動方式**: `SIGTERM`による優雅な終了
- **除外パターン**: キャッシュ、ビルド成果物、Gitファイル等

### 3. 開発用Dockerfile

```dockerfile
FROM ubuntu:24.04

# 開発用パッケージのインストール
RUN pip install --no-cache-dir -r requirements.txt watchdog[watchmedo]

# 開発環境用の設定
ENV PYTHONPATH=/app
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

# ヘルスチェック最適化（10秒間隔）
HEALTHCHECK --interval=10s --timeout=5s --start-period=30s --retries=3 \
  CMD python -c "import requests; requests.get('http://localhost:8080/health')"
```

## 便利なMakefileコマンド

開発効率を上げるためのコマンドセット：

```makefile
# 開発環境起動
dev:
	@echo "Building development Docker images..."
	@git submodule update --init --recursive
	@DOCKER_BUILDKIT=1 docker-compose build --parallel
	@echo "Starting development containers..."
	@docker-compose up -d
	@echo "Hot reload enabled! 🔥"

# リアルタイムログ表示
dev-logs:
	@echo "Showing development logs (use Ctrl+C to exit)..."
	@docker-compose logs -f

# 開発環境再起動
dev-restart:
	@echo "Restarting development containers..."
	@docker-compose restart

# 開発環境の状態確認
dev-status:
	@docker-compose ps
	@echo "Recent logs:"
	@docker-compose logs --tail=20
```

## 実装のポイント

### 1. ファイル監視の最適化

```python
# 除外パターンの工夫
IGNORE_PATTERNS = [
    '*/__pycache__/*',  # Pythonキャッシュ
    '*/.*/*',           # 隠しファイル
    '*/.git/*',         # Gitファイル
    '*/node_modules/*', # Node.jsファイル
    '*/build/*',        # ビルド成果物
    '*/dist/*'          # 配布用ファイル
]
```

### 2. 優雅な再起動

```python
# SIGTERMによる優雅な終了
def signal_handler(signum, frame):
    logger.info("Received termination signal, shutting down gracefully...")
    # クリーンアップ処理
    app.stop()
    sys.exit(0)

signal.signal(signal.SIGTERM, signal_handler)
```

### 3. 開発環境の視覚的フィードバック

```python
# 起動時にHot Reload状態を表示
if os.getenv('WATCHDOG_ENABLED') == 'true':
    logger.info("🔥 HOT RELOAD IS WORKING! 🔥")
    logger.info(f"🕒 Started at: {datetime.now()}")
```

## 開発フロー

1. **環境起動**
   ```bash
   make dev
   ```

2. **ログ監視**
   ```bash
   make dev-logs
   ```

3. **コード編集** → **自動再起動** → **即座にテスト**

## 様々なフレームワークでの適用例

### FastAPI
```python
# main.py
from fastapi import FastAPI
app = FastAPI()

@app.get("/")
def read_root():
    return {"message": "Hello World"}
```

### Flask
```python
# app.py
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
    return {'message': 'Hello World'}

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000, debug=True)
```

### Django
```bash
# コマンド例
watchmedo auto-restart --patterns='*.py' --recursive --signal SIGTERM --directory=/app --interval=0.5 -- python manage.py runserver 0.0.0.0:8000
```

## トラブルシューティング

### よくある問題と解決法

1. **Hot Reloadが働かない**
   ```bash
   make dev-restart
   ```

2. **ファイル変更が検知されない**
   - Docker Desktopの場合、ファイル監視設定を確認
   - `interval`を短く設定（0.1秒など）

3. **メモリ使用量が多い**
   - 除外パターンを追加
   - 監視対象ディレクトリを限定

## パフォーマンス指標

| 項目          | 従来    | Hot Reload |
| ------------- | ------- | ---------- |
| 変更→反映時間 | 30-60秒 | 1-2秒      |
| 開発サイクル  | 長い    | 短い       |
| 開発者体験    | 😫       | 🚀          |

## まとめ

Docker + watchdog + Makefileの組み合わせで、Python開発の効率が劇的に向上しました。特に：

- **即座のフィードバック**: 0.5秒でコード変更を検知
- **優雅な再起動**: SIGTERMによる安全な終了
- **開発者フレンドリー**: 直感的なコマンド体系
- **フレームワーク非依存**: FastAPI、Flask、Django等どれでも適用可能

この環境により、「コード変更 → 保存 → 即座にテスト」という快適な開発フローが実現できました。

Pythonアプリケーション開発でのHot Reload環境、ぜひお試しください！ 🔥

---

*この記事で紹介した設定は、WebアプリケーションからAPI、バックグラウンドサービス、Bot開発まで、様々なPythonアプリケーションに適用可能です。*