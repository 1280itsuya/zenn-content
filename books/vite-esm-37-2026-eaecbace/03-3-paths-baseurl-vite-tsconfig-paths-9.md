---
title: "第3章 paths/baseUrl のエイリアスが解決されない: vite-tsconfig-paths併用の落とし穴9件"
free: false
---

本文を出力する。

---

**結論を3行で:** `tsconfig.json` の `paths` はTypeScriptの型解決にしか効かず、Viteの実行時バンドルは無視する。`@/` を本番ビルドで通すには `vite-tsconfig-paths` か `resolve.alias` のどちらか一方に統一する。両方併用すると二重解決で `Failed to resolve import "@/..."` が出る。

## 症状1: dev は通るが build で `Failed to resolve import "@/utils"`

`tsc` は `paths` を読むため `npm run dev` は型エラーゼロで起動する。だが `vite build` は esbuild がエイリアスを知らず落ちる。最短修正は `vite.config.ts` に `resolve.alias` を手書きすることだ。

```ts
// vite.config.ts
import { defineConfig } from "vite";
import { fileURLToPath, URL } from "node:url";

export default defineConfig({
  resolve: {
    alias: {
      "@": fileURLToPath(new URL("./src", import.meta.url)),
    },
  },
});
```

## 症状2: `vite-tsconfig-paths` を入れたのに解決されない

プラグインを `plugins` 配列に push し忘れているか、`tsconfig.json` に `baseUrl` が無いケースが大半。プラグインは `baseUrl` を起点に `paths` を展開するため、省略すると黙ってスキップする。

```ts
// vite.config.ts
import tsconfigPaths from "vite-tsconfig-paths";

export default defineConfig({
  plugins: [tsconfigPaths()],
});
```

```jsonc
// tsconfig.json — baseUrl 必須
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] }
  }
}
```

## 症状3: 二重解決で `[plugin vite:resolve] ... already aliased`

`resolve.alias` と `vite-tsconfig-paths` を同時に有効化すると、同じ `@/` を2回解決して衝突する。下表のとおり**どちらか一方**に絞るのが正解。

| 設定 | dev | build | 判定 |
|---|---|---|---|
| alias手書きのみ | ✅ | ✅ | OK |
| plugin のみ | ✅ | ✅ | OK |
| 両方併用 | ✅ | ❌ | NG |

```bash
# 併用を検知する1行チェック
grep -l "resolve.alias" vite.config.ts && grep -l "tsconfigPaths" vite.config.ts \
  && echo "二重解決の疑い: どちらか削除せよ"
```

## 症状4: Monorepo で workspace パッケージの `paths` が無視される

ルートの `tsconfig.json` の `paths` はサブパッケージのViteに継承されない。`vite-tsconfig-paths` に `root` を明示し、対象 `tsconfig` を指定する。

```ts
tsconfigPaths({
  root: "../../",
  projects: ["packages/ui/tsconfig.json"],
});
```

## 症状5: TS 5.4 以降で `baseUrl` 省略時に `Cannot find module`

TS 5.4 から `baseUrl` 無しでも `paths` 単独指定が許容されたが、相対起点が `tsconfig` のあるディレクトリへ変わった。`./` 起点に書き換える。

```jsonc
{
  "compilerOptions": {
    "paths": { "@/*": ["./src/*"] }  // 先頭の ./ が必須に
  }
}
```
