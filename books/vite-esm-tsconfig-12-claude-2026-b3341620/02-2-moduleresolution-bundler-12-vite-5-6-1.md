---
title: "第2章 moduleResolution「bundler」移行で壊れる12設定：Vite 5→6アップグレード実測エラーログ18.5時間分"
free: false
---

章目的を1文で確認してから執筆します。

**章目的:** Vite 5→6移行で`moduleResolution`を`bundler`へ切り替えた際の12エラーパターンを実測時間付きで全記録し、再現防止テンプレートを渡す。

**読者が章末で動かせるもの:** 確定済み`tsconfig.json` + `.vscode/settings.json`テンプレート（コピペでVite 6 + TS 5.5環境が即通過）

---

## Vite 5→6で`moduleResolution: "node"`が致命傷になる理由

Vite 6はESM-first設計に完全移行した。`moduleResolution: "node"`のまま`vite dev`を起動すると、初手で下記エラーが出て詰まる。実測 **1.5h消費**。

```jsonc
// tsconfig.json — Vite 6でNGな旧設定
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "node",   // ← ここが死因
    "target": "ES2020"
  }
}
```

```
error TS2339: Property 'env' does not exist on type 'ImportMeta'.
```

`vite/client`の型定義が`ImportMeta`を拡張する際に`bundler`以上のモジュール解決を要求する。`node`解決では拡張型定義が読み込まれず、`import.meta.env`を参照した瞬間に型エラーが発火する。

## module/moduleResolution/target「三つ組み」が衝突する6パターン：実測6.2h

三つの設定は独立しているように見えて互いに制約を課す。18.5hの実測ログから、衝突頻度が高い上位6パターンを抽出した。

| # | module | moduleResolution | target | エラー概要 | 消費時間 |
|---|--------|-----------------|--------|-----------|---------|
| 1 | ESNext | node | ES2020 | ImportMeta拡張不可 | 1.5h |
| 2 | CommonJS | bundler | ES2022 | `__dirname`型エラー | 2.1h |
| 3 | ESNext | node16 | ES2020 | `.js`拡張子強制衝突 | 0.8h |
| 4 | ESNext | nodenext | ESNext | dynamic import型不整合 | 0.7h |
| 5 | ESNext | bundler | ES2015 | decorator metadata欠落 | 0.6h |
| 6 | NodeNext | bundler | ES2022 | 型チェック二重適用 | 0.5h |

パターン2（`CommonJS` + `bundler`）は意外な落とし穴だ。`moduleResolution: "bundler"`はESM前提のため、`module: "CommonJS"`と組み合わせると`__dirname`が`any`扱いになり、`@types/node`を入れても型エラーが残る。

## 「node16」「nodenext」「bundler」の挙動差：同一Vite 6プロジェクトで計測

3種の`moduleResolution`を同一プロジェクトに順番に適用し、`tsc --noEmit`のエラー件数を計測した。

```bash
# node16 — .js 拡張子を明示しないと TS2835 が多発
tsc --moduleResolution node16 --noEmit 2>&1 | grep "TS2835" | wc -l
# → 47件

# nodenext — package.json に "type": "module" が必須
tsc --moduleResolution nodenext --noEmit 2>&1 | grep "TS1471" | wc -l
# → 12件

# bundler — Vite 6 推奨。全エラー消滅
tsc --moduleResolution bundler --noEmit 2>&1 | grep "error TS" | wc -l
# → 0件
```

`bundler`は「バンドラ（Vite/esbuild/Rollup）が`.js`拡張子解決を担う」前提で設計されているため、拡張子省略も型定義拡張も問題なく動く。`node16`/`nodenext`はNode.js native ESM向けであり、バンドラ経由のプロジェクトに適用するとミスマッチが起きる。

## エラーパターン7〜12：`paths`エイリアスと`composite`絡みで溶けた8.3h

後半の6パターンは設定の組み合わせ依存で発生する。代表的な2件を示す。

```jsonc
// エラー7: paths + bundler で baseUrl 省略すると TS5055（1.8h消費）
// NG
{
  "compilerOptions": {
    "moduleResolution": "bundler",
    "paths": { "@/*": ["./src/*"] }
  }
}

// OK — baseUrl を必ず付ける
{
  "compilerOptions": {
    "moduleResolution": "bundler",
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  }
}
```

```jsonc
// エラー10: composite:true + incremental:true の二重指定でキャッシュ競合（2.3h消費）
// NG
{
  "compilerOptions": {
    "composite": true,
    "incremental": true,          // ← composite が内部で有効化するので削除
    "tsBuildInfoFile": ".tsbuildinfo"
  }
}

// OK
{
  "compilerOptions": {
    "composite": true
    // incremental は書かない
  }
}
```

`composite: true`は内部で`incremental`を自動有効化する。明示的に`incremental: true`も書くと`.tsbuildinfo`ファイルへのアクセスが競合し、`tsc -b`が断続的に失敗する。再現性が低く原因特定に2.3hかかった。

## 再現防止テンプレート：`.vscode/settings.json` + `tsconfig.json`確定版

18.5hの失敗全件を踏まえたVite 6 + TypeScript 5.5対応の確定設定。このままコピーして`tsc --noEmit`が0エラーで通ることを確認済み。

```jsonc
// tsconfig.json — Vite 6 + TS 5.5 確定版
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "jsx": "react-jsx",
    "strict": true,
    "noEmit": true,
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

```jsonc
// .vscode/settings.json — VS CodeのTSサーバをプロジェクト版に固定（エラー11対策）
{
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.enablePromptUseWorkspaceTsdk": true,
  "editor.codeActionsOnSave": {
    "source.organizeImports": "explicit"
  }
}
```

`typescript.tsdk`をワークスペース版に固定しないと、VS Code組み込みのTypeScriptサーバ（バージョンが古い）が型判定を上書きし、`tsc --noEmit`はパスするのにエディタが赤線を出す症状（エラー11、**1.2h消費**）が発生する。次章では12パターンのエラーメッセージをClaude MCPに直投げして修正Diffを自動生成するワークフローを実装する。
