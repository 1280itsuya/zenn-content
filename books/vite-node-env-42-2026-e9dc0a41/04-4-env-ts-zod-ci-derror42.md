---
title: "第4章 型付きenv.ts+zod検証でCI落としderror42個を起動前に殺す"
free: false
---

## env.ts を zod で起動前に殺す: 必須キー欠落と空文字を例外化

`import.meta.env.VITE_API_URL` を直に触ると型は `string | undefined`、空文字 `""` もすり抜けてランタイムで初めて死ぬ。zod でパースして起動時に throw させれば、error42個のうち「未定義参照」「空文字fetch」系の17個をビルド前に潰せる。

```ts
// src/env.ts
import { z } from "zod";

const schema = z.object({
  VITE_API_URL: z.string().url(),
  VITE_SENTRY_DSN: z.string().min(1),
  VITE_FEATURE_FLAG: z.enum(["on", "off"]).default("off"),
});

const parsed = schema.safeParse(import.meta.env);
if (!parsed.success) {
  console.error(parsed.error.flatten().fieldErrors);
  throw new Error("env検証に失敗。起動を中止");
}
export const env = parsed.data; // 以後 env.VITE_API_URL は string確定
```

`.url()` が `http://`欠落を、`.min(1)` が空文字を弾く。`safeParse` の `fieldErrors` でどのキーが原因か即特定できる。

## vite-env.d.ts で ImportMetaEnv を拡張し any 地獄を消す

zod を通しても `import.meta.env` 自体の型は any のまま補完が効かない。`ImportMetaEnv` を拡張すると、`env.ts` を経由しない直アクセスもエディタで赤線になる。

```ts
// src/vite-env.d.ts
/// <reference types="vite/client" />
interface ImportMetaEnv {
  readonly VITE_API_URL: string;
  readonly VITE_SENTRY_DSN: string;
  readonly VITE_FEATURE_FLAG: "on" | "off";
}
interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

zod schema と手書きでズレるのが事故元なので、`z.infer<typeof schema>` を真実とし、型生成は後述の検証スクリプトに寄せる。

## GitHub Actions に「build前」env検証ジョブを足す

本番デプロイ後にエラーへ気づく事故の根治には、ビルド本体とは別の軽量ジョブで env だけ先に検証する。`tsx` で env.ts を import するだけで zod が走り、失敗なら exit 1。

```yaml
# .github/workflows/env-check.yml
name: env-check
on: [pull_request]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm dlx tsx -e "import('./src/env.ts')"
        env:
          VITE_API_URL: ${{ secrets.VITE_API_URL }}
          VITE_SENTRY_DSN: ${{ secrets.VITE_SENTRY_DSN }}
```

ビルド3分を待たず約20秒でCIが落ちる。Secrets未登録のキーは `undefined` のまま検証に入るので、Settings側の登録漏れもこのジョブが暴く。

## pnpm workspace で env.ts を共有し、process.env と import.meta.env を分離

monorepo では Node側API(`process.env`)とVite側(`import.meta.env`)が混在し、片方の前提で書いた env.ts を共有すると参照先がズレて死ぬ。`packages/env` に source 別の export を切る。

```ts
// packages/env/src/index.ts
import { z } from "zod";
const base = { API_URL: z.string().url() };

export const viteEnv = (src: ImportMetaEnv) =>
  z.object({ VITE_API_URL: base.API_URL }).parse(src);

export const nodeEnv = (src: NodeJS.ProcessEnv) =>
  z.object({ API_URL: base.API_URL }).parse(src);
```

```jsonc
// pnpm-workspace.yaml
packages:
  - "apps/*"
  - "packages/*"
```

apps/web は `viteEnv(import.meta.env)`、apps/api は `nodeEnv(process.env)` を呼ぶ。スキーマ定義 `base` は1か所、入口だけ source 別にすることで「VITE_接頭辞なしをブラウザで参照」事故を構造的に封じる。

## Turborepo キャッシュ汚染を防ぐ env 宣言

Turborepo は env 変更を検知しないと古いビルドをキャッシュ再利用し、`.env` を直したのに反映されない事故(error42の#39)が起きる。`turbo.json` に `env` を明示し、ハッシュ入力に含める。

```jsonc
// turbo.json
{
  "tasks": {
    "build": {
      "env": ["VITE_API_URL", "VITE_SENTRY_DSN"],
      "outputs": ["dist/**"]
    }
  }
}
```

これで `VITE_API_URL` が変われば自動で再ビルド。`turbo build --dry=json` を実行すると、各タスクの `globalEnv`/`env` 解決結果が確認でき、宣言漏れキーを一覧で洗い出せる。env を入力に含めた瞬間から「直したのに古い」が消える。
