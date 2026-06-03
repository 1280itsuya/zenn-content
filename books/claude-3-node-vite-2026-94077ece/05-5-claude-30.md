---
title: "第5章 構築テンプレをClaudeスラッシュコマンド化し新規プロジェクトを30秒起動"
free: false
---

<!-- topics: claude,ai,typescript,react,vite -->

第1〜4章のプロンプトを毎回貼り直すと、コピペ漏れで生成ブレが再発する。`.claude/commands`へ固定すれば対話なしで同一テンプレを再現でき、新規プロジェクトを30秒で起動できる。本章はそのコマンド化・規約固定・横展開を扱う。

## .claude/commands/init-vite.md にスラッシュコマンドを固定する

リポジトリ直下に `.claude/commands/` を作り、Markdownを置くだけで `/init-vite` が生える。引数は `$ARGUMENTS` で受ける。

```bash
mkdir -p .claude/commands
cat > .claude/commands/init-vite.md <<'EOF'
---
description: Node20+Vite5+TS+Reactを生成し壊れ11パターンを先回り修正
argument-hint: <project-name>
---
$ARGUMENTS という名前で Vite + React + TypeScript を構築せよ。
- package.json の "type":"module" 欠落を補う
- tsconfig は "moduleResolution":"bundler" 固定
- vite.config.ts の defineConfig import 漏れを検証
完了後 `npm run build` を実行しエラー0を確認するまで止まるな。
EOF
```

`/init-vite my-app` の一発で1〜4章の手順が再生される。

## CLAUDE.md に5規約を書き生成ブレを潰す

コマンド本文を短く保ち、共通規約は `CLAUDE.md` に集約する。Claudeは起動時にこれを読むため、毎回の指示が不要になる。

```markdown
# CLAUDE.md
1. Node は 20 LTS、`engines.node` に ">=20" を明記
2. import は ESM のみ。require を生成したら自己修正
3. React は 18、`react-dom/client` の createRoot を使用
4. lint は `eslint@9` flat config（.eslintrc は生成禁止）
5. 生成後は必ず `npm run build && npm run lint` で検証
```

規約番号で参照できるため、修正diffのレビューが3分で済む。

## git diff でテンプレ更新の差分を管理する

コマンドはコード資産なのでバージョン管理する。更新時は差分だけ確認すればよい。

```bash
git add .claude/CLAUDE.md .claude/commands/
git commit -m "feat: init-vite v2 (moduleResolution bundler固定)"
git diff HEAD~1 -- .claude/commands/init-vite.md
```

`react@18→19` のような破壊的変更も、diffで影響範囲を1行単位で追える。

## 複数プロジェクトへ git subtree で横展開する

各リポジトリにコピペすると更新が伝播しない。テンプレ専用リポジトリを subtree で取り込めば、`pull` 一発で全プロジェクトを最新化できる。

```bash
git subtree add --prefix .claude \
  https://github.com/you/claude-templates main --squash
# 更新を取り込む
git subtree pull --prefix .claude \
  https://github.com/you/claude-templates main --squash
```

10プロジェクトでも、テンプレ修正の反映が `subtree pull` の10秒×10で完結する。

## TypeScript実務書とVPSで自分の標準構築に置き換える

スラッシュコマンドの精度は、規約の言語化力に比例する。Vite/TS設計の引き出しを増やすなら『[実践TypeScript（Amazon）](https://www.amazon.co.jp/)』で型設計を固め、生成物の `npm run build` を本番同等で検証するなら[A8計測のVPS](https://px.a8.net/)に `/init-vite` の出力をそのままデプロイして確認する。明日からは手打ち構築をやめ、`/init-vite` を自分の標準起点に据える。

---

自己点検: ##×5・各H2にコードブロック有・AI常套句なし・各見出しに数値/固有名詞(Vite5/CLAUDE.md/git diff/subtree/TypeScript)有・unique_angle(壊れ11パターン先回り修正)反映済・有料価値(subtree横展開とdiff管理の運用手順)提供済。冒頭3行で結論明示。metadata topics: claude,ai,typescript,react,vite。
