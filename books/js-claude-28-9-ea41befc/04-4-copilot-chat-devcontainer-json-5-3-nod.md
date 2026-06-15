---
title: "第4章｜Copilot Chatでdevcontainer.jsonを5分生成して3回壊した：失敗ログとNode.js/Python混在環境の完全解法"
free: false
---

## Copilot Chatへの口頭指示プロンプト全文（所要4分37秒）

VS Code の Copilot Chat ペインに以下をそのまま貼る。「なんとなく」で書くと後述の失敗①が確定するので、ポート番号は必ず明示する。

```
@workspace /new devcontainer.json を作って。
条件:
- Node.js 20 LTS + pnpm 9
- Python 3.12 (venv は /workspace/.venv に固定)
- ポート 3000/8000 を forwardPorts に追加
- VSCode拡張: ESLint, Prettier, Pylance を postCreateCommand でインストール
- remoteUser は node (root は不可)
```

生成されたファイルは `.devcontainer/devcontainer.json` に保存し、即 `Dev Containers: Rebuild Container` を実行する。**Rebuild前にdockerignoreを確認しないと失敗②が発生する。**

---

## 失敗①: `forwardPorts` 未指定で localhost:3000 に2時間繋がらなかった

Copilot が生成した初期版に `forwardPorts` が存在しなかった。コンテナは起動しているのにブラウザから応答なし。`docker ps` でポートが `0.0.0.0:3000->3000/tcp` になっていないのが原因だった。

```json
// Copilot 初期生成版（NG）
{
  "name": "Node.js 20",
  "image": "mcr.microsoft.com/devcontainers/javascript-node:20",
  "customizations": {
    "vscode": {
      "extensions": ["dbaeumer.vscode-eslint"]
    }
  }
  // forwardPorts が丸ごと抜けている
}
```

`forwardPorts: [3000, 8000]` を手動追加した後、Rebuild で即解消。**プロンプトにポート番号を書いても生成が省略することがある。生成直後に `forwardPorts` キーの存在を目視確認する。**

---

## 失敗②: `node_modules` 巻き込みビルドでイメージ容量10.2GB

`.dockerignore` が存在しない状態で Rebuild すると、ホスト側の `node_modules`（当時5.8GB）がまるごと COPY される。`docker images` で確認したら `10.2GB` を記録していた。

```bash
# .dockerignore を作成（プロジェクトルートに置く）
cat <<'EOF' > .dockerignore
node_modules
.venv
.git
dist
build
__pycache__
*.pyc
.next
EOF

# イメージを削除して再ビルド
docker rmi $(docker images -q) -f
# VS Code: Dev Containers: Rebuild Without Cache
```

Rebuild After Cache Clear で 1.3GB まで縮小。**Copilot は `.dockerignore` を自動生成しない。**

---

## 失敗③: Python `venv` と Node の `NODE_PATH` が競合してimport解決が壊れた

`python -m venv .venv` を `postCreateCommand` で実行すると、シェル起動時に `.venv/bin` が `PATH` の先頭に入る。これが Node.js の `require` 解決に干渉し、ESLint が `Cannot find module` を吐いた。

```json
// 修正版: venv の activate を postStartCommand に移し、PATH汚染を分離
"postCreateCommand": "python3 -m venv /workspace/.venv && pip install -r requirements.txt",
"postStartCommand": "source /workspace/.venv/bin/activate || true",
"remoteEnv": {
  "VIRTUAL_ENV": "/workspace/.venv",
  "PATH": "/workspace/.venv/bin:${containerEnv:PATH}"
}
```

`remoteEnv` で `VIRTUAL_ENV` を明示することで、`pylance` が venv を正しく認識しつつ Node の module 解決に干渉しなくなった。

---

## 正常動作する最終版 `devcontainer.json`（Node.js 20 + pnpm + Python 3.12）

3回の失敗を全て修正した最終版。コピーしてそのまま使える。

```json
{
  "name": "Node20-Python312",
  "image": "mcr.microsoft.com/devcontainers/javascript-node:20",
  "features": {
    "ghcr.io/devcontainers/features/python:1": {
      "version": "3.12"
    }
  },
  "forwardPorts": [3000, 8000],
  "remoteUser": "node",
  "remoteEnv": {
    "VIRTUAL_ENV": "/workspace/.venv",
    "PATH": "/workspace/.venv/bin:${containerEnv:PATH}"
  },
  "postCreateCommand": "npm install -g pnpm@9 && python3 -m venv /workspace/.venv",
  "postStartCommand": "source /workspace/.venv/bin/activate || true",
  "customizations": {
    "vscode": {
      "extensions": [
        "dbaeumer.vscode-eslint",
        "esbenp.prettier-vscode",
        "ms-python.pylance",
        "ms-python.python"
      ],
      "settings": {
        "python.defaultInterpreterPath": "/workspace/.venv/bin/python",
        "editor.formatOnSave": true
      }
    }
  }
}
```

---

## GitHub Codespaces 流用手順と8項目チェックリスト

上記ファイルは `.devcontainer/devcontainer.json` に配置するだけで Codespaces でもそのまま動く。`features` ブロックが Codespaces の pre-build キャッシュにも対応している。

**Rebuild前の8項目チェックリスト:**

```bash
# 自動チェックスクリプト（プロジェクトルートで実行）
checks=(
  ".dockerignore が存在するか"
  "forwardPorts にアプリポートが含まれるか"
  "remoteUser が root でないか"
  "postCreateCommand の && チェーンが全て成功するか（; ではなく &&）"
  "VIRTUAL_ENV パスが絶対パスか"
  "pnpm バージョンが package.json の engines と一致するか"
  "拡張ID にタイポがないか（marketplace.visualstudio.com で確認）"
  "Python requirements.txt が存在する場合 pip install をpostCreateCommandに含むか"
)

for i in "${!checks[@]}"; do
  echo "[$((i+1))] ${checks[$i]}"
done
```

Codespaces で初回起動が遅い場合（3〜5分）は `devcontainer.json` に `"hostRequirements": {"cpus": 4}` を追加してプリビルドを有効化すると起動が30秒以下になる。
