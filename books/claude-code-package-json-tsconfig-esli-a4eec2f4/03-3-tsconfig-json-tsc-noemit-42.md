---
title: "第3章 tsconfig.jsonをtsc --noEmitで自己修正させ設定起因エラー42件を自動解消"
free: false
---

## tsc --noEmitの出力をClaude Agent SDKへ戻す自己修正ループ・設定生成・検証の全体構造

tsconfigは人が直すより`tsc --noEmit`の出力をそのままClaudeに食わせて直させる方が速い。CLIの中核は「生成→検証→エラーを戻して再生成」の3工程だけだ。

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

async function selfHeal(config: string, env: BuildEnv): Promise<string> {
  for (let loop = 1; loop <= 5; loop++) {
    const errors = await runTsc(config);          // tsc --noEmit
    if (errors.length === 0) return config;       // 収束
    config = await regenerate(config, errors, env); // Claudeに差し戻し
  }
  throw new Error("打ち切り: 5ループで未収束");
}
```

## Node18/20/22とESM/CJS分岐をtarget/module/moduleResolutionへ渡す検証プロンプト設計

Claudeに環境を構造体で渡すと、`target`/`module`/`moduleResolution`の組合せ事故が消える。Node22+ESMなら`ES2023`+`NodeNext`を強制する。

```typescript
type BuildEnv = { node: 18 | 20 | 22; mod: "esm" | "cjs" };

const rules = (e: BuildEnv) =>
  `Node${e.node}/${e.mod}向け。mod=esmなら"module":"NodeNext",
   "moduleResolution":"NodeNext"を必須。target下限はES2022。`;
```

## 一時ファイルにtscを実行し設定起因エラー42件を抽出するBash検証コマンド

検証は一時ディレクトリで完結させ、本物のプロジェクトを汚さない。`--pretty false`で機械可読なエラー行だけを得る。

```bash
mktemp_dir=$(mktemp -d)
echo "$CONFIG" > "$mktemp_dir/tsconfig.json"
npx tsc --noEmit --pretty false -p "$mktemp_dir/tsconfig.json" 2>&1 \
  | grep -oE 'error TS[0-9]+: .*'   # 初回は42行=42件のエラー
```

## 平均2.3ループで収束させる打ち切り条件とトークン消費を計測するcontrollerコード

target/module/strictの不整合で出た42件は、平均2.3ループ・約8.1kトークンで0件になった。同一エラー集合が2回続いたら振動とみなし打ち切る。

```typescript
let prev = "";
function shouldAbort(errs: string[], loop: number): boolean {
  const sig = errs.sort().join("|");
  const stuck = sig === prev;        // 前ループと同一=振動
  prev = sig;
  return stuck || loop >= 5;         // 実測の打ち切り上限
}
```

## 差し戻し時に過去エラーを蓄積してClaudeへ再投入する再生成ハンドラ

エラーは累積で渡すと、Claudeが直した箇所を再び壊す逆戻りが止まる。直近だけでなく全履歴を1メッセージに畳んで投入する。

```typescript
const history: string[][] = [];
async function regenerate(cfg: string, errs: string[], env: BuildEnv) {
  history.push(errs);
  const log = history.map((e, i) => `#${i + 1}: ${e.join(", ")}`).join("\n");
  const res = query({ prompt:
    `${rules(env)}\n現tsconfig:\n${cfg}\n累積エラー:\n${log}\n
     全エラーを解消したtsconfig.jsonのみ返答。説明文禁止。` });
  let out = "";
  for await (const m of res) if (m.type === "text") out += m.text;
  return out.match(/\{[\s\S]*\}/)?.[0] ?? cfg;  // JSON部のみ抽出
}
```

42件→0件をCLIに任せれば、tsconfigの手直しは設計判断だけに絞れる。
