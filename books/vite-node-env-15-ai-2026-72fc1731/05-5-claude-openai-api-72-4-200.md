---
title: "第5章: Claude/OpenAI APIキー漏洩ゼロ運用—72時間で$4,200請求された失敗から設計した防御策"
free: false
---

**章の目的**: $4,200実被害を起点に、pre-commitフック→CI健全性確認→シークレット管理比較→1コマンドオンボーディングの4層防御を実装させる。

**章末で読者が手に動かせるもの**: `npm run env:setup` 1コマンドで新メンバーが全変数を安全取得するオンボーディングスクリプト。

---

## $4,200請求が届いた72時間—git push 1回でAnthropicキーが世界に拡散した経緯

結論: publicリポジトリへのキーpushはボットに平均3分で検出される。コードレビューだけでは防げない理由と被害額の実数をここで示す。

Viteプロジェクトの初回コミットに`.env`を含めたまま`git push`した翌朝、Anthropicのダッシュボードには自分では実行していないAPIコール数千万件のグラフが立っていた。GitHubのシークレットスキャンが検出する前に、OSSボットが`ANTHROPIC_API_KEY=sk-ant-`パターンを3分以内に拾う事例はセキュリティ界隈で繰り返し報告されている。被害額は72時間で$4,200。Anthropicへの問い合わせで最終的に$0に戻せたが、返金まで14日かかった。

`.gitignore`に`.env`を書いても、初回コミット前にファイルが追加されていれば意味をなさない。唯一有効な防御はコミット前の機械的遮断だ。

---

## gitleaks v8 + pre-commit フック: 5分で全メンバーの環境に配布する

gitleaks v8はAnthropicキー（`sk-ant-`）・OpenAIキー（`sk-`）を標準ルールセットに含む。

```bash
# macOS/Linux 共通インストール
brew install gitleaks
# CI/Docker 環境向けバイナリ直接取得
curl -sSfL https://github.com/gitleaks/gitleaks/releases/download/v8.18.4/gitleaks_8.18.4_linux_x64.tar.gz \
  | tar -xz -C /usr/local/bin
```

```toml
# .gitleaks.toml — プロジェクトルートに配置
[extend]
useDefault = true   # Anthropic/OpenAI/GitHub Token 等 100+ パターン込み

[[rules]]
id = "custom-anthropic"
description = "Anthropic API Key"
regex = '''sk-ant-[A-Za-z0-9\-_]{93}'''
tags = ["anthropic", "api-key"]
```

```js
// scripts/setup-hooks.mjs — npm install 時に自動実行
import { writeFileSync, chmodSync } from 'node:fs'

const hookPath = '.git/hooks/pre-commit'
const script = `#!/bin/sh\ngitleaks protect --staged --config .gitleaks.toml\nif [ $? -ne 0 ]; then\n  echo "❌ gitleaks: シークレット検出。コミット中止。"\n  exit 1\nfi\n`

writeFileSync(hookPath, script)
chmodSync(hookPath, 0o755)
console.log('✅ pre-commit hook installed')
```

```json
// package.json
{
  "scripts": {
    "prepare": "node scripts/setup-hooks.mjs",
    "env:setup": "node scripts/env-setup.mjs"
  }
}
```

`npm install` を実行するだけでフックが全員に配布される。

---

## GitHub Actions cron でClaudeキーの有効性を毎朝JST 9時に自動確認する

ローテーション後に「実際に使えるか」を確認しないまま本番デプロイすると、突然401エラーで止まる。下記ワークフローは毎朝`/v1/models`を叩いてレスポンスを検証し、失敗時はSlackに通知する。無料枠（プライベートリポジトリ月2,000分）でこのチェックは1回約10秒、月31回で合計5分以下に収まる。

```yaml
# .github/workflows/check-api-keys.yml
name: API Key Health Check
on:
  schedule:
    - cron: '0 0 * * *'   # UTC 0:00 = JST 9:00
  workflow_dispatch:

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - name: Check Anthropic Key
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          status=$(curl -s -o /dev/null -w "%{http_code}" \
            https://api.anthropic.com/v1/models \
            -H "x-api-key: $ANTHROPIC_API_KEY" \
            -H "anthropic-version: 2023-06-01")
          echo "HTTP Status: $status"
          [ "$status" = "200" ] || { echo "::error::Anthropic key invalid (HTTP $status)"; exit 1; }

      - name: Check OpenAI Key
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          status=$(curl -s -o /dev/null -w "%{http_code}" \
            https://api.openai.com/v1/models \
            -H "Authorization: Bearer $OPENAI_API_KEY")
          [ "$status" = "200" ] || { echo "::error::OpenAI key invalid (HTTP $status)"; exit 1; }

      - name: Notify Slack on failure
        if: failure()
        uses: slackapi/slack-github-action@v1.27.0
        with:
          payload: '{"text":"🚨 APIキー検証失敗: 即ローテーションを"}'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## Infisical無料枠 vs dotenv-vault $11/月 vs Vault OSS: 3軸で選ぶ比較表

| 比較軸 | Infisical Free | dotenv-vault $11/月 | Vault OSS (自前) |
|---|---|---|---|
| **チーム人数** | 無制限 | 3名まで | 無制限 |
| **CI/CD連携** | GitHub Actions / Vercel / Railway ネイティブ | `dotenv-vault pull` コマンド | Vault Agent or API |
| **監査ログ** | 30日分 | なし（$29プランから） | 無制限（自己管理） |
| **キーローテーション** | 自動 (Dynamic Secrets) | 手動 | 自動 (lease機能) |
| **セットアップ時間** | 15分 | 5分 | 2〜4時間 |
| **推奨ケース** | 5名以下スタートアップ | ソロ開発者 | 100名超・コンプラ要件あり |

dotenv-vaultに月$11を払うなら、同額帯でInfisical Teamにアップグレードして監査ログを得る方が合理的だ。

---

## `npm run env:setup` 1コマンドで新メンバーが全変数を安全取得するオンボーディングスクリプト

Infisicalにアクセス権を付与された新メンバーは、このスクリプト1つで`.env.local`を生成できる。

```js
// scripts/env-setup.mjs
import { execSync } from 'node:child_process'
import { writeFileSync } from 'node:fs'

const REQUIRED = [
  'ANTHROPIC_API_KEY',
  'OPENAI_API_KEY',
  'DATABASE_URL',
  'VITE_PUBLIC_API_BASE',
]

function fetchFromInfisical() {
  try {
    const raw = execSync(
      'infisical export --format=dotenv --env=development',
      { encoding: 'utf-8' }
    )
    return Object.fromEntries(
      raw.split('\n')
        .filter(l => l.includes('='))
        .map(l => l.split('=', 2))
    )
  } catch {
    console.error('❌ infisical login を先に実行してください。')
    process.exit(1)
  }
}

const secrets = fetchFromInfisical()
const missing = REQUIRED.filter(k => !secrets[k])
if (missing.length > 0) {
  console.error(`❌ 未設定のシークレット: ${missing.join(', ')}`)
  process.exit(1)
}

const output = REQUIRED.map(k => `${k}=${secrets[k]}`).join('\n') + '\n'
writeFileSync('.env.local', output)
console.log(`✅ .env.local を生成 (${REQUIRED.length}件)`)
```

新メンバーのセットアップはこの4コマンドで完結する。

```bash
git clone https://github.com/your-org/your-repo
cd your-repo
npm install          # prepare フックで gitleaks も自動設置
npm run env:setup    # Infisical から .env.local を自動生成
npm run dev
```

$4,200の請求から設計した防御の全レイヤーが、この4コマンドに収まった。pre-commitフックが初回コミット前の漏洩を止め、GitHub Actions cronが失効を翌朝検知し、Infisicalがチーム全員のシークレットを一元管理する。本書で積み上げてきた「undefined地獄→型エラー地獄→CI/CD渡し失敗」の3段階詰まりに対するすべての解決策は、このオンボーディング1コマンドに集約される。
