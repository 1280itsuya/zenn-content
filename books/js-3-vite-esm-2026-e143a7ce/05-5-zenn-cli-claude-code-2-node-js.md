---
title: "第5章 zenn-cli+Claude Codeで技術記事を週2本半自動生成するNode.jsワークフロー実装"
free: false
---

第5章を執筆します。

---

## zenn-cli 2.x をESM化する3ステップとpackage.jsonの落とし穴

`"type": "module"` を入れた直後に踏むエラーはほぼ決まっている。

```
Error [ERR_REQUIRE_ESM]: require() of ES Module
.../zenn-cli/dist/index.mjs not supported
```

**根本原因:** zenn-cliはv0.1.140以降でESM配布に切り替わっているが、呼び出し元スクリプトが `.js` のままだとNode.jsがCJSとして解釈して衝突する。

**修正diff:**
```diff
- "scripts": { "new:article": "node scripts/article-gen.js" }
+ "scripts": { "new:article": "node scripts/article-gen.mjs" }
```

```json
// package.json — 最小構成
{
  "type": "module",
  "scripts": {
    "new:article": "node scripts/article-gen.mjs",
    "preview": "npx zenn preview"
  },
  "devDependencies": {
    "zenn-cli": "^0.1.157"
  }
}
```

```bash
npx zenn init
mv scripts/article-gen.js scripts/article-gen.mjs
node --input-type=module < scripts/article-gen.mjs  # 事前検証
```

`.mjs` を明示することで、`"type": "module"` によるプロジェクト全体ESM化とzenn-cli内部の解釈が衝突しない。

---

## Claude Code MCP補完が効くjsconfig.jsonのVite+zenn-cli同居設計

Viteとzenn-cliを同一リポジトリに置くと、Claude CodeのMCP補完がzenn API型定義を見失うケースがある。原因は `moduleResolution` の不一致。

```json
// jsconfig.json
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "target": "ESNext",
    "baseUrl": ".",
    "paths": {
      "@zenn/*": ["node_modules/zenn-cli/src/*"],
      "@/*": ["src/*"]
    },
    "checkJs": true
  },
  "include": ["scripts/**/*.mjs", "src/**/*"],
  "exclude": ["articles", "books", "node_modules"]
}
```

`"Bundler"` モードはViteのエイリアス解決とNode.jsのESM解決を共存させられる唯一の設定値。`articles/` と `books/` をexcludeに入れると、Markdownへの誤補完が止まりClaude CodeのAgentモードで「この章の動作確認コードを生成して」と投げたときの補完精度が上がる。

---

## Front Matter自動生成スクリプト article-gen.mjs の全実装

週2本の最大ボトルネックは記事着手前の手作業。以下を実行すれば0秒でZenn記事が開ける状態になる。

```javascript
// scripts/article-gen.mjs
import { createHash } from 'node:crypto'
import { writeFileSync, mkdirSync } from 'node:fs'
import { resolve } from 'node:path'

const TITLE  = process.argv[2] ?? '新しい技術記事'
const TOPICS = (process.argv[3] ?? 'javascript,nodejs').split(',')

const slug = createHash('sha256')
  .update(TITLE)           // タイトル固定→同一slugを再生成可能
  .digest('hex')
  .slice(0, 12)

const frontMatter = `---
title: "${TITLE}"
emoji: "⚡"
type: "tech"
topics: [${TOPICS.map(t => `"${t}"`).join(', ')}]
published: false
---

## TL;DR

<!-- Claude Code: この行以下に章構成を3点で列挙 -->
`

const filePath = resolve('articles', `${slug}.md`)
mkdirSync('articles', { recursive: true })
writeFileSync(filePath, frontMatter, 'utf-8')
console.log(`created: articles/${slug}.md`)
```

```bash
node scripts/article-gen.mjs "Vite+ESMで3時間溶かした話" "vite,esm,javascript"
# → articles/8f3a1c2e904b.md
```

`<!-- Claude Code: -->` コメントがプロンプト起点として機能する。`checkJs: true` 状態のjsconfigと組み合わせると、Claude Codeが型推論しながらコード補完を展開する。

---

## SHA-256決定論的slugでURL衝突と書き直しコストをゼロにする

zenn-cliのデフォルトslugはタイムスタンプ乱数なので、同一タイトルで記事を作り直すとURLが変わりSEO評価がリセットされる。上記スクリプトではSHA-256をタイトルから計算しているため、再実行しても同じslugが出る。

```javascript
import { createHash } from 'node:crypto'

const deterministicSlug = (title) =>
  createHash('sha256').update(title).digest('hex').slice(0, 12)

// 何度実行しても同値
console.log(deterministicSlug('Vite+ESMで3時間溶かした話'))
// → "8f3a1c2e904b"
```

「書き直し→別slug公開→重複記事」というよくある事故が物理的に起きなくなる。

---

## zenn publish + Git push を1コマンドに束ねるBashラッパー

```bash
#!/usr/bin/env bash
# scripts/publish.sh
set -euo pipefail

ARTICLE=${1:?Usage: publish.sh articles/<slug>.md}

sed -i 's/^published: false$/published: true/' "$ARTICLE"
git add "$ARTICLE"
git commit -m "publish: $(basename "$ARTICLE" .md)"
git push origin main

echo "✓ ${ARTICLE} → Zennに反映（mainブランチpushでトリガー）"
```

```bash
chmod +x scripts/publish.sh
./scripts/publish.sh articles/8f3a1c2e904b.md
```

ZennはGitHub連携設定済みならmain pushを検知して自動公開する。このスクリプト1本で「`published: false → true` 書き換え → コミット → push → Zenn反映」が完結する。月曜と木曜にカレンダーアラームを設定してこのコマンドを叩くだけで、週2本ペースは維持できる。本書で構築したVite+ESM環境への投資を、Zenn記事という収益導線に還元するエンドツーエンドの自動化がここで完成する。
