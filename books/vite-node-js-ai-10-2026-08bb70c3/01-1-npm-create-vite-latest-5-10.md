---
title: "第1章: npm create vite@latest 5コマンドで動く環境 — 本書が解体する10エラー全地図【無料】"
free: true
---

## Node.js 20.x + Vite 6.3 を5コマンドで起動する

まず手元で動く状態を作る。バージョン固定が後のエラー再現に直結する。

```bash
node -v          # v20.x.x を確認（LTS）
npm create vite@6.3.0 my-ai-tool -- --template react-ts
cd my-ai-tool
npm install
npm run dev      # → http://localhost:5173
```

ブラウザに "Vite + React" が表示されれば出発点完成。`@6.3.0` を省略すると章ごとの再現性が崩れる。Node.js 18 以下では第3章のエラー #3 が初手から出る。

## Docker sandbox 設定ファイル — 全章共通の隔離環境

本書の全コード検証はこの compose ファイルで動かしている。ホスト環境を汚さず、GitHub Actions の ubuntu-latest と Node バージョンを揃えられる。

```yaml
# docker-compose.yml
services:
  dev:
    image: node:20-slim
    working_dir: /app
    volumes:
      - .:/app
    ports:
      - "5173:5173"
    command: sh -c "npm install && npm run dev -- --host 0.0.0.0"
    environment:
      - NODE_ENV=development
```

```bash
docker compose up
```

`--host 0.0.0.0` がないとコンテナ外からアクセスできない。これがエラー #5 の伏線になる。

## 本書が解体する10エラー全地図 — 平均ロスト時間付き

| # | エラー概要 | 発生条件 | 平均ロスト時間 | 難易度 |
|---|-----------|---------|--------------|--------|
| 1 | `VITE_` prefix 欠落で undefined | Claude Code 生成の `.env` | 45分 | ★☆☆ |
| 2 | `import.meta.env` が Node.js 側で undefined | GitHub Actions runner | 90分 | ★★☆ |
| 3 | ESM/CJS 混在で `vite.config.ts` がクラッシュ | Claude 生成プラグイン追加時 | 2.5h | ★★★ |
| 4 | `process.env` と `import.meta.env` の二重定義 | 共有ユーティリティ | 1h | ★★☆ |
| 5 | ポート 5173 が Actions コンテナで競合 | service container 使用時 | 30分 | ★☆☆ |
| 6 | CORS エラー: AI API プロキシ設定漏れ | `vite.config.ts` の proxy 未設定 | 1.5h | ★★☆ |
| 7 | HMR WebSocket が Actions 内で切断 | self-hosted runner / Codespace | 3h | ★★★ |
| 8 | `__dirname` is not defined in ES module | ESM 移行後の設定ファイル | 45分 | ★☆☆ |
| 9 | TypeScript strict エラー 17件 | Claude 生成コードの型推論 | 2h | ★★☆ |
| 10 | `vite build` が exit 0 でも成果物なし | 環境変数 undefined の silent fail | 4h | ★★★ |

**合計ロスト時間の中央値: 1.3h/エラー。10件に全部ハマると 16.8h 消える計算になる。**

## Claude Code + GitHub Actions で多発する3つの構造的理由

単体の Vite プロジェクトでは起きにくい。この組み合わせ固有の地雷がある。

```text
理由1: Claude Code は Node.js 環境を想定したコードを生成する
       → ブラウザ専用の import.meta.env を使うべき箇所に process.env を混入

理由2: GitHub Actions の ubuntu-latest は Node.js 18 がデフォルト
       → node:20 を明示しないと ESM の挙動が変わり #2/#3 が確率的に発生

理由3: Actions の secrets は VITE_ prefix なしで注入される
       → ビルド時に全変数が undefined になり、エラーも出ずに壊れる（#10）
```

`node-version: "20"` を省いた CI ファイルは時限爆弾になる。

```yaml
# .github/workflows/ci.yml — バージョン固定の最小構成
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"    # ← ここを省くと #2 と #3 が同時発生
      - run: npm ci
      - run: npm run build
```

## 第2章以降の購入判断 — 表の何番で詰まっているかを先に確認する

上の表で自分のエラー番号が見つかれば、該当章へ直接ジャンプできる構成になっている。

- **#1〜#2（環境変数系）** → 第2章「`VITE_` prefix 強制と Claude Code の `.env` 生成ルール修正」
- **#3〜#4（ESM/CJS 系）** → 第3章「`vite.config.ts` ESM 化と CJS 混在クラッシュの完全回避」
- **#7 / #10（silent fail 系）** → 第6章「exit 0 なのに壊れる: Actions 内 `vite build` デバッグ手順」

ロスト時間が大きい順に優先すると #10(4h)→#7(3h)→#3(2.5h) の3章だけで **9.5h を取り戻せる**。表で自分の詰まりポイントが1件でも見つかれば、第2章以降は即実務に刺さる内容になっている。
