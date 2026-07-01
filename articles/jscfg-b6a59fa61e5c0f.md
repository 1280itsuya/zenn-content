---
title: "tsconfig moduleResolution bundlerでimportできない"
emoji: "⚙️"
type: "tech"
topics: ["JavaScript", "TypeScript", "設定"]
published: true
---

## 発生条件

- TypeScript 5.0 以降で `moduleResolution: "bundler"` を設定しているプロジェクト
- Vite・esbuild・Webpack 5 など、バンドラー側のモジュール解決に依存している構成
- `paths` エイリアスや `exports` フィールドを持つパッケージを拡張子なしでインポートしている場合

## 原因

```json
// tsconfig.json（壊れる例）
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "bundler",
    "paths": {
      "@/utils": ["./src/utils"]
    }
  }
}
```

`moduleResolution: "bundler"` はバンドラーの挙動を模倣する設定で、TypeScript 5.0 で追加された。この設定は拡張子の省略や `package.json` の `exports` フィールドを解釈できる一方、**`paths` に登録したエイリアスの解決をバンドラー側に委ねる形になる**。

問題が起きやすいのは、`paths` のマッピング先がディレクトリ指定になっているケースだ。`"@/utils": ["./src/utils"]` のように末尾が `.ts` ファイルではなくディレクトリを指している場合、TypeScript は `index.ts` の存在を自動で補完しない。`moduleResolution: "bundler"` では Node.js の暗黙的なディレクトリ解決（`index` ファイルの自動探索）が保証されないため、`Cannot find module '@/utils'` のエラーが発生する。

また、`module` が `"CommonJS"` のまま `moduleResolution: "bundler"` を指定すると組み合わせが不正として TS エラーになるケースも多い。`"bundler"` は `module: "ESNext"` または `"Preserve"` との組み合わせが前提となる。

## 修正方法

```json
// tsconfig.json（修正例）
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "bundler",
    "paths": {
      "@/utils": ["./src/utils/index.ts"],
      "@/utils/*": ["./src/utils/*"]
    },
    "allowImportingTsExtensions": true,
    "noEmit": true
  }
}
```

修正のポイントは3点ある。

**① `paths` のマッピングをファイル単位で明示する。** ディレクトリ指定では `index.ts` を自動探索しないため、`./src/utils/index.ts` と直接指定するか、ワイルドカードパターン `@/utils/*` → `./src/utils/*` を追加してサブモジュールも解決できるようにする。

**② `allowImportingTsExtensions: true` を追加する。** バンドラー経由でビルドする場合、TypeScript はトランスパイルのみを担当しファイル出力を行わないことが多い。`noEmit: true` と組み合わせることで、`.ts` 拡張子付きのインポートを許可できる。

**③ `module` と `moduleResolution` の組み合わせを合わせる。** `moduleResolution: "bundler"` を使う場合は `module` を `"ESNext"` か `"Preserve"` にする。`"CommonJS"` との組み合わせは TS 5.x でエラーになるため注意する。

Vite プロジェクトでは `vite.config.ts` 側の `resolve.alias` と `tsconfig.json` の `paths` を両方定義する必要がある点も見落としやすい。TypeScript の型チェックとバンドラーの実行時解決は独立しているため、片方だけ設定しても動作しない。

---

## JS/TS設定エラーをAIが30秒で根本診断
→ [診断キット](https://zenn.dev/itsuya_auto/books/jsconfig-diag-kit)
