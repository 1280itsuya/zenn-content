---
title: "第5章 14地雷を二度踏まない：型安全なenvスキーマ検証とCIで起動前に落とすvalidate-env自動化"
free: false
---

## zodで14地雷を1枚のenvスキーマに封じる

防御の起点は「envの型をコードで宣言する」こと。第2〜4章の14地雷（接頭辞`VITE_`漏れ・本番で空文字・型不一致）は、`env.schema.ts`に必須キーを列挙した瞬間8割が死ぬ。

```ts
// src/env.schema.ts
import { z } from "zod";

export const envSchema = z.object({
  VITE_API_BASE: z.string().url(),
  VITE_SENTRY_DSN: z.string().min(1),
  VITE_FEATURE_FLAG: z.enum(["on", "off"]),
});

export function parseEnv(raw: Record<string, unknown>) {
  return envSchema.parse(raw); // 不一致なら即throw
}
```

`z.string().url()`が地雷7（URLのつもりが空文字）を、`z.enum`が地雷11（typoした値）を起動前に弾く。

## viteプラグインで起動時にfail-fastさせる

`import.meta.env`は欠損してもundefinedで黙って進むのが地雷の温床。`config`フックで`parseEnv`を呼び、欠落時は`vite dev`自体を落とす。

```ts
// plugins/validate-env.ts
import { loadEnv, type Plugin } from "vite";
import { envSchema } from "../src/env.schema";

export function validateEnv(): Plugin {
  return {
    name: "validate-env",
    config(_, { mode }) {
      const env = loadEnv(mode, process.cwd(), "VITE_");
      const r = envSchema.safeParse(env);
      if (!r.success) {
        console.error(r.error.flatten().fieldErrors);
        throw new Error("env検証NG: 起動を中止");
      }
    },
  };
}
```

`loadEnv(mode, cwd, "VITE_")`の第3引数で接頭辞を固定し、地雷1（接頭辞漏れ）をビルド前に可視化する。

## .env.exampleとの差分を欠損キーで検出する

新メンバーが`.env`を作り損ねる地雷4は、`.env.example`のキー集合との差分でCI落としする。

```bash
# scripts/check-env-diff.sh
ex=$(grep -oE '^VITE_[A-Z_]+' .env.example | sort -u)
cur=$(grep -oE '^VITE_[A-Z_]+' .env | sort -u)
missing=$(comm -23 <(echo "$ex") <(echo "$cur"))
[ -n "$missing" ] && { echo "未定義: $missing"; exit 1; } || echo "OK"
```

## GitHub Actionsで起動前に落とす

`vite build`の前段に検証ジョブを置き、PRマージ前に14地雷を遮断する。

```yaml
# .github/workflows/env.yml
on: [pull_request]
jobs:
  validate-env:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: bash scripts/check-env-diff.sh
      - run: npx tsx -e "import('./plugins/validate-env').then(m=>m.validateEnv())"
```

## ビルド成果物に機密が混入していないかgrep監査

最後の地雷14は「`VITE_`を付けたせいでAPIキーが`dist/`にバンドルされる」事故。`dist`を正規表現で走査し、検出したら`exit 1`。

```bash
# scripts/audit-dist.sh
if grep -rEn 'sk-[A-Za-z0-9]{20,}|AKIA[0-9A-Z]{16}' dist/; then
  echo "機密混入を検出: 公開中止"; exit 1
fi
echo "dist監査クリア"
```

この5本を`npm run guard`に束ね、`validate-env → diff → build → audit-dist`をひと続きにすれば、個人開発でも14地雷の再発を0件で回せる。
