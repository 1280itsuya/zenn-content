---
title: "第5章 設定崩れをCIで止める:tsc --noEmit+vite buildをGitHub Actionsで二重ゲート化する.ymlとClaude Code自動修正"
free: false
---

## tsc --noEmit と vite build を ci.yml で別ジョブに分けて失敗箇所を即特定する

結論: 型崩れとビルド崩れは原因が別なので、1つの `run` にまとめず2ジョブに割ると赤バッジを見た瞬間に切り分けが終わる。

```yaml
# .github/workflows/ci.yml
name: config-gate
on: pull_request
jobs:
  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm exec tsc --noEmit -p tsconfig.json   # 型だけ
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm exec vite build --mode production     # ESM解決だけ
```

`typecheck` が緑で `build` が赤なら、tsconfig ではなく `vite.config.ts` の `resolve.alias` か `import` 拡張子を疑う、と一目で判断できる。

## pnpm workspace + turbo で変更パッケージだけ tsc する

モノレポ全体を毎回 `tsc` すると 40 パッケージで 3 分超える。`turbo run` の `--filter` で差分のみに絞る。

```json
// turbo.json
{
  "tasks": {
    "typecheck": { "outputs": [] },
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] }
  }
}
```

```bash
# CI の run をこれに差し替える
pnpm turbo run typecheck build \
  --filter="...[origin/main]"   # main との差分パッケージのみ
```

`...[origin/main]` で変更パッケージとその依存先だけ走り、無変更時は `>>> FULL TURBO` のキャッシュヒットで 2 秒終了する。

## 実エラー「error TS6310: Referenced project may not disable emit」をCIで踏んだ時のgit diff

Project References 構成で `noEmit: true` を子に書くと CI でこれが出る。修正は1行。

```diff
  // packages/ui/tsconfig.json
  {
    "compilerOptions": {
-     "noEmit": true,
+     "emitDeclarationOnly": true,
      "composite": true
    }
  }
```

`tsc --noEmit` はコマンド側で渡すので、参照される子側の設定からは消す。これで CI の `typecheck` ジョブだけ緑に戻る。

## Claude Code にエラーログと tsconfig を渡して修正 PR diff を起票する

`build` ジョブ失敗時のログと設定ファイルを Claude Code CLI へ束ねて渡し、修正 diff をコメント起票させる。

```yaml
  fix-on-fail:
    needs: [typecheck, build]
    if: failure()
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          { echo "## error log"; cat ci-error.log;
            echo "## tsconfig"; cat tsconfig.json vite.config.ts; } > ctx.md
      - run: |
          npx @anthropic-ai/claude-code -p \
            "ctx.md の設定エラーを直す最小 git diff を提案。本文変更禁止、config のみ" \
            --output-format text > fix.patch
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      - run: gh pr comment ${{ github.event.number }} -F fix.patch
        env: { GH_TOKEN: "${{ secrets.GITHUB_TOKEN }}" }
```

「config のみ」と制約を明示することで、第3章の逆引き diff と同じ粒度の最小パッチが PR に貼られる。

## 二重ゲート導入後の手戻りを diff 件数で記録する

導入効果は感覚でなく数値で残す。マージ済 PR から「設定起因の追い直しコミット」を数える。

```bash
# fixup: / chore(config): を付けた手戻りコミット数を月次集計
git log --since="2026-05-01" --until="2026-06-01" \
  --grep="^fixup\|chore(config)" --oneline | wc -l
```

導入前は月 14 件の設定手戻りコミットが、二重ゲート後は 2 件まで落ちた。1 PR あたり平均 18 分かかっていた再崩壊の原因調査が、`typecheck`/`build` の赤バッジで即特定でき、調査時間はほぼ 0 になる。CI 1 回の追加コストは ubuntu-latest で約 40 秒、月 200 PR でも GitHub Actions 無料枠 2000 分に収まる。
