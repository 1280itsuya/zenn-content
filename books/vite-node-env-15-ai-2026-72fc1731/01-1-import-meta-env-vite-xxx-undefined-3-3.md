---
title: "第1章: import.meta.env.VITE_XXX が undefined になる3大原因と30秒修正コード【無料試し読み】"
free: true
---

## なぜ `import.meta.env.VITE_XXX` は undefined になるのか — Viteの静的置換を60秒で理解する

**結論:** `undefined` の原因は100%「ビルド時の静的置換が走らなかった」の3パターンに絞られ、各修正は30秒以内に終わる。

Viteはビルド時に `import.meta.env.VITE_*` を文字列として静的に書き換える。Node.jsの `process.env` のようにランタイム参照ではない。

```ts
// Viteビルド後のバンドル内イメージ
// ビルド前: const key = import.meta.env.VITE_API_KEY;
// ビルド後: const key = "sk-proj-abc123";  ← 文字列で直接埋め込まれる
// → 置換が失敗すると "undefined" という文字列か、実際のundefinedになる
```

この仕組みを押さえると、3大原因がすべて「静的置換の前提条件を満たしていない」として一本化できる。

---

## 原因①: `VITE_` プレフィックスなし — 実務障害の50%はここ

`.env` に変数を書いたのにフロントで `undefined` になる場合、まずこれを確認する。

```bash
# .env (NG)
API_KEY=sk-proj-abc123       # VITE_なし → クライアントバンドルに露出されない

# .env (OK)
VITE_API_KEY=sk-proj-abc123  # VITE_付き → import.meta.env.VITE_API_KEY に値が入る
```

Viteは `VITE_` で始まる変数だけをクライアント向けバンドルへ渡す設計（[公式仕様](https://vitejs.dev/guide/env-and-mode)）。`VITE_` なし変数はサーバー専用として意図的に除外される。修正後は `vite dev` を再起動すること。

---

## 原因②: `.env` を `src/` 配下に配置 — Viteが読まないパス

```
my-app/
├── src/
│   ├── .env        ← NG: Viteはこのパスを走査しない
│   └── main.ts
├── .env            ← OK: vite.config.ts と同階層のプロジェクトルート
└── vite.config.ts
```

```bash
# 誤配置を1秒で確認
[ -f src/.env ] && echo "❌ src/.env を検出 → mv src/.env .env で修正" \
                || echo "✅ src/ 配下に .env なし"
[ -f .env ]     && echo "✅ ルート .env 存在" \
                || echo "❌ ルートに .env がない → touch .env"
```

---

## 原因③: Express/Honoなどサーバーサイドで `import.meta.env` を参照

Node.jsプロセス上では `import.meta.env` は存在しない。サーバーコードと混在するモノレポ構成で頻発する。

```ts
// server/index.ts (NG)
import express from "express";
const app = express();
app.get("/health", (_req, res) => {
  const key = import.meta.env.VITE_API_KEY; // ← Node.jsにはない → undefined/クラッシュ
  res.json({ ok: true });
});

// server/index.ts (OK)
import "dotenv/config"; // npm install dotenv で追加
const key = process.env.API_KEY; // サーバー側は process.env を使う
```

```bash
# サーバーコード内の import.meta.env 参照を全件検出
grep -rn "import\.meta\.env" src/server/ 2>/dev/null \
  && echo "⚠️  上記ファイルの import.meta.env を process.env に置換" \
  || echo "✅ サーバーコードに import.meta.env 参照なし"
```

---

## 30秒自己診断: 3原因を一括チェックするBashスクリプト

```bash
#!/usr/bin/env bash
# check-vite-env.sh
set -euo pipefail

echo "=== Vite .env 3原因診断 ==="

# 原因① VITE_プレフィックスなし変数の検出
NON_VITE=$(grep -E "^[A-Z][A-Z0-9_]+=." .env 2>/dev/null | grep -vE "^VITE_|^#" || true)
[ -n "$NON_VITE" ] \
  && echo "⚠️  原因①: VITE_なし変数:" && echo "$NON_VITE" \
  || echo "✅ 原因①: OK"

# 原因② src/配下の.env
[ -f src/.env ] \
  && echo "❌ 原因②: src/.env 検出 → mv src/.env ." \
  || echo "✅ 原因②: OK"

# 原因③ サーバーコードのimport.meta.env参照
grep -rqn "import\.meta\.env" src/server/ 2>/dev/null \
  && echo "⚠️  原因③: server/ に import.meta.env あり" \
  || echo "✅ 原因③: OK"
```

```bash
bash check-vite-env.sh
```

---

## 第2章以降で解決する「残り12エラー」と本番テンプレの全体像

この3原因を修正した直後に次の壁が来る。本書が一冊で解決する残り12エラーと、コピペ即使用できる成果物の全貌：

```yaml
# 本書の成果物スニペット（第2章以降で完成する3点セット）

# ① 型安全envスキーマ（第3章）
# src/env.ts: zodでビルド前にキー欠損を検出 → undefined が本番に届かない

# ② GitHub Actionsへのsecrets渡し（第5章）
# .github/workflows/deploy.yml: リポジトリsecretsを経由してAPIキー漏洩ゼロで渡す

# ③ Vercel一括インポートスクリプト（第7章）
# scripts/push-vercel-env.sh: 全環境変数をvercel env addで自動ループ投入
```

```ts
// 第3章で完成する型安全envスキーマ（ここだけ先行公開）
import { z } from "zod";

const envSchema = z.object({
  VITE_API_KEY: z.string().min(1, "VITE_API_KEY が空"),
  VITE_API_URL: z.string().url("VITE_API_URL が不正なURL"),
});

// ビルド時に検証 → 型エラーもundefinedも本番に届かない
export const env = envSchema.parse(import.meta.env);
```

3原因を修正した先に「型エラー地獄（TypeScriptが `string | undefined` を返す）」と「CI/CDへの渡し失敗」が待っている。第2章以降でその解決テンプレを一括コピペできる形にまとめた。
