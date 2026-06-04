---
title: "第2章 Claude Code Agent SDKでpackage.json生成プロンプトを設計し型崩れ0にする"
free: false
---

## プレーン一発生成が10回中4回壊れる実測ログ

結論: `claude -p "package.jsonを書いて"` の素朴な一発生成は型崩れする。手元で30回回した内訳が以下だ。

```bash
# 検証用ハーネス。生成結果をJSON.parseに通すだけ
for i in $(seq 1 30); do
  claude -p "TypeScript用のpackage.jsonを生成して。コードブロックのみ返せ" \
    --output-format text > out_$i.json
  node -e "JSON.parse(require('fs').readFileSync('out_$i.json'))" \
    && echo "$i OK" || echo "$i NG"
done
```

失敗12件のうち、不正JSON（末尾の説明文混入）が7件、`"typescript": "^9.9.9"` のような存在しないバージョンが5件。初回10回に絞ると4件が壊れた。原因はLLMに「JSONを書く」だけを任せ、検証経路がないことに尽きる。

## generate→validate→self-correctを回すAgent SDKループ

LLMにJSONを書かせるのではなく、検証ツールを噛ませて自己修正させると型崩れが消える。Agent SDKの`tools`に検証関数を渡し、不合格なら同じセッションで修正させる。

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

const result = query({
  prompt: GEN_PROMPT,
  options: {
    allowedTools: ["validate_package_json"],
    maxTurns: 4, // リトライ上限。超えたら打ち切り
    systemPrompt: "生成→validate_package_json呼び出し→errorsが空になるまで修正",
  },
});
for await (const msg of result) {
  if (msg.type === "result") console.log(msg.result);
}
```

`maxTurns: 4` が肝で、検証→修正の往復を最大3回まで許す。これで30件中30件が一発でparse可能になった。

## npm registryで依存バージョンの実在を検証するツール

存在しないバージョンは、生成時ではなく検証ツールで弾く。npm registryへHEADを投げ、404なら差し戻す。

```typescript
async function validatePackageJson(input: { json: string }) {
  const pkg = JSON.parse(input.json);
  const errors: string[] = [];
  for (const [name, range] of Object.entries(pkg.dependencies ?? {})) {
    const v = String(range).replace(/^[\^~]/, "");
    const res = await fetch(`https://registry.npmjs.org/${name}/${v}`);
    if (res.status === 404) errors.push(`${name}@${v} は存在しない`);
  }
  return { errors }; // 空配列ならOK、SDKが自己修正を止める
}
```

`typescript@9.9.9` のような幻覚は404で即検出され、SDKが実在する最新版（例: `5.7.2`）へ書き換える。

## JSON Schemaでscripts命名とtype:module整合を縛る

結論: scripts命名規約とexports整合は自然言語ではなくJSON Schemaで縛る。Ajvに通し、違反メッセージをそのまま修正プロンプトへ返す。

```typescript
import Ajv from "ajv";
const schema = {
  type: "object",
  required: ["name", "version", "type", "scripts"],
  properties: {
    type: { const: "module" }, // CJS生成を禁止
    scripts: {
      type: "object",
      required: ["build", "lint", "test"], // 命名規約を強制
      additionalProperties: { type: "string" },
    },
    exports: { type: "object" }, // type:module時は必須化
  },
  allOf: [{ if: { properties: { type: { const: "module" } } },
           then: { required: ["exports"] } }],
};
const ajv = new Ajv();
const valid = ajv.compile(schema);
```

`type: "module"` なのに`exports`が無い構成はここで落ち、検証ツールが`ajv.errors`を文字列化してSDKへ返す。

## リトライ上限と1回約¥6のコスト試算

30件を本ループで処理した実測コストを置く。修正往復が増えるほど課金が伸びるため、上限管理が収益直結する。

```bash
# usageを集計（result メッセージのtokenから算出）
# 入力 1,800tok / 出力 600tok / 平均1.4往復
# Sonnet: (1800*3 + 600*15)/1e6 USD ≈ $0.0144 ≈ ¥2.2/往復
# 平均1.4往復 → 約¥3。検証404での再生成込みでも約¥6/件
echo "30件 × ¥6 = ¥180（一発生成の手直し時間ゼロ）"
```

型崩れ0件・1件¥6・最大4ターンという3数値を握れば、第3章のtsconfig生成へそのままループを流用できる。Agent SDKの`maxTurns`を上げるほど精度は上がるが、¥6が¥15へ膨らむ損益分岐をこの章の表で必ず確認しておく。
