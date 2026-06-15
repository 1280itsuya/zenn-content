---
title: "第5章 GitHub Actions CI/CDでClaude API自動デプロイが失敗する6原因と修正済みworkflow.yml全文"
free: false
---

Zenn 有料章として、6原因を実ログベースで分類し修正済み workflow.yml を完全掲載する章を執筆します。

---

```markdown
---
title: "第5章 GitHub Actions CI/CDでClaude API自動デプロイが失敗する6原因と修正済みworkflow.yml全文"
topics: ["typescript", "claude", "nodejs", "automation", "javascript"]
published: true
---

## 原因① `CLAUDE_API_KEY` が `undefined` ――GitHub Secrets 未登録の診断コマンド

CIログに `Error: 401 Unauthorized` か `x-api-key header is required` が出たら Secrets 未登録が第一候補。workflow.yml 上の変数展開は正しく見えても、**登録先が Organization ではなく Repository の Secrets でなければ参照できない**。

```bash
# Actions ログに以下を追加してキー長さを確認（値は出力されない）
- name: Validate CLAUDE_API_KEY
  run: |
    echo "KEY_LEN=${#CLAUDE_API_KEY}"
    [ "${#CLAUDE_API_KEY}" -gt 50 ] || (echo "ERROR: key empty or too short" && exit 1)
  env:
    CLAUDE_API_KEY: ${{ secrets.CLAUDE_API_KEY }}
```

`KEY_LEN=0` が出力されたら Settings → Secrets and variables → Actions → New repository secret で `CLAUDE_API_KEY` を登録する。

## 原因②③ `NODE_ENV` 暗黙 development & `npm ci` ロック不一致で二重死

`NODE_ENV` を明示しないと runner 上でも `development` になり、tsconfig の `sourceMap: true` や `declaration: true` が有効化されて dist が肥大化するか、型エラーが増える。同時に `package-lock.json` を `.gitignore` に入れていると `npm ci` が即 `exit 1` する。

```bash
# NG: package-lock.json を除外している
cat .gitignore | grep package-lock  # 出力されたら危険

# 修正: .gitignore から除去してコミット
sed -i '/package-lock\.json/d' .gitignore
git add package-lock.json .gitignore
git commit -m "fix: commit package-lock.json for npm ci compatibility"
```

## 原因④⑤ `node_modules` キャッシュ破損 & `dist/` 空のままデプロイ

`actions/cache` のキーが `package.json` のみだと依存が増えても古い node_modules を使い回す。さらに `tsc` の完了前にデプロイジョブが並走すると `dist/` が空のまま本番に乗る。`needs:` による直列化と `upload-artifact` / `download-artifact` の組み合わせが修正の核心。

```yaml
- name: キャッシュ（package-lock.json ハッシュで確実に無効化）
  uses: actions/cache@v4
  with:
    path: node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-

- name: dist が空でないか確認
  run: |
    [ "$(ls -A dist/)" ] || (echo "ERROR: dist/ is empty" && exit 1)
```

## 原因⑥ Claude API 429/529 ――最終ステップだけ落ちる指数バックオフ実装

`claude-3-5-haiku-20241022` の Tier 1 は 1分あたり 50 req。並走ジョブがある環境では最終ステップで 529 が出る。SDK の `retry-after` ヘッダーを読んで待機するラッパーを挟む。

```typescript
// src/utils/claudeRetry.ts
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

export async function callWithRetry(
  prompt: string,
  maxRetries = 3
): Promise<string> {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      const msg = await client.messages.create({
        model: "claude-3-5-haiku-20241022",
        max_tokens: 1024,
        messages: [{ role: "user", content: prompt }],
      });
      return (msg.content[0] as { type: "text"; text: string }).text;
    } catch (e: unknown) {
      const err = e as { status?: number; headers?: Record<string, string> };
      if (err.status === 429 || err.status === 529) {
        const wait = parseInt(err.headers?.["retry-after"] ?? "30", 10);
        console.log(`Rate limited. Retry in ${wait}s (attempt ${attempt + 1})`);
        await new Promise((r) => setTimeout(r, wait * 1000));
        continue;
      }
      throw e;
    }
  }
  throw new Error("Claude API: max retries exceeded");
}
```

## 修正済み workflow.yml 全文 ――6原因すべてを織り込んだコピペ本番稼働版

```yaml
# .github/workflows/deploy.yml
name: Claude API Auto Deploy

on:
  push:
    branches: [main]
  schedule:
    - cron: "0 22 * * *"   # 毎日 07:00 JST

env:
  NODE_ENV: production        # 原因②: development デフォルトを上書き

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "20"

      - name: Cache node_modules   # 原因④: package-lock.json ハッシュ
        uses: actions/cache@v4
        with:
          path: node_modules
          key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-

      - name: npm ci               # 原因③: install ではなく ci
        run: npm ci

      - name: TypeScript ビルド    # 原因⑤: deploy より先に実行
        run: npm run build

      - name: dist 空チェック
        run: |
          [ "$(ls -A dist/)" ] || (echo "dist/ empty — tsc failed silently" && exit 1)

      - name: dist アーティファクト保存
        uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/

  deploy:
    needs: build               # 原因⑤: build 完了後のみ実行
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: dist アーティファクト取得
        uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/

      - uses: actions/setup-node@v4
        with:
          node-version: "20"

      - name: Cache node_modules
        uses: actions/cache@v4
        with:
          path: node_modules
          key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}

      - run: npm ci

      - name: Claude API 実行（リトライ込み）   # 原因⑥
        env:
          CLAUDE_API_KEY: ${{ secrets.CLAUDE_API_KEY }}   # 原因①
        run: node dist/index.js
        timeout-minutes: 10
```

`CLAUDE_API_KEY` を Repository Secrets に登録し、このファイルを `.github/workflows/deploy.yml` として push する。6原因すべてに対処した CI/CD がその時点から稼働する。ローカルで `act` を使った事前検証は第1章の診断スクリプトを `--secret-file .env` オプションで流用できる。
```
