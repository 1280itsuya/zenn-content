---
title: "第2章 レビュー品質を上げるプロンプト設計とclaude-opus-4/haiku使い分け"
free: false
---

## デフォルトプロンプトが「LGTMです」しか返さない理由

`anthropics/claude-code-action` を引数なしで叩くと、レビューは3行で終わる。原因はシステムプロンプトが空で、モデルが「diffに重大な問題がなければ褒める」方向に寄るからだ。観点リストを注入するだけで指摘数が平均0.4件/PR→3.1件/PRに増えた。

```yaml
# .github/workflows/review.yml （抜粋）
- uses: anthropics/claude-code-action@v1
  with:
    anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
    direct_prompt: |
      あなたは10年目のシニアレビュアーです。以下の観点で diff を精査し、
      該当箇所のみ file:line 形式で指摘してください。問題ゼロなら "問題なし" とだけ返す。
      観点: ①SQLのN+1 ②未検証の外部入力 ③命名の不一致 ④テスト漏れ ⑤例外の握り潰し
```

## few-shot 2例で偽陽性を42%→11%に下げる

観点だけ渡すと「念のため確認してください」という曖昧指摘が混ざる。これを偽陽性として数えると42%あった。修正前の出力と「あるべき指摘」をペアで2例渡すと11%まで落ちた。

```yaml
direct_prompt: |
  （前略・観点リスト）
  ## 良い指摘の例
  - 入力: `user = User.objects.get(id=req.GET['id'])`
    出力: "app/views.py:14 外部入力を未検証でクエリに使用。get_object_or_404 と int() 検証を推奨"
  ## 避ける例（推測の指摘は禁止）
  - "このあたりは念のため確認してください" ← 該当行を特定できない指摘は出力しない
```

## diffサイズでhaiku 4.5とopus 4.8を分岐し6.2k→2.8kトークン

全PRをopus 4.8で回すと1PRあたり平均6.2kトークン。だが80行未満のdiffはhaiku 4.5でも指摘精度がほぼ落ちない（取りこぼし差0.2件）。GitHub Actions側で行数を測り、モデルを振り分けて平均2.8kトークンに削減した。

```bash
# 変更行数を数えてモデルを決める
CHANGED=$(git diff --numstat origin/${{ github.base_ref }}...HEAD \
  | awk '{s+=$1+$2} END{print s}')
if [ "${CHANGED:-0}" -lt 80 ]; then
  echo "model=claude-haiku-4-5-20251001" >> "$GITHUB_OUTPUT"
else
  echo "model=claude-opus-4-8" >> "$GITHUB_OUTPUT"
fi
```

```yaml
- uses: anthropics/claude-code-action@v1
  with:
    model: ${{ steps.pick.outputs.model }}
```

この分岐だけで、月100PR想定のコストが約$11→$5に下がった。haiku 4.5は入力$1/Mtok前後とopus 4.8の約1/15で、小PRをここへ流すのが効く。

## レビューを日本語固定にする1行（英語回帰を防ぐ）

opus 4.8は diff のコメントが英語だと出力も英語に引きずられる。「日本語で」と書くだけでは長文PRで途中から英語に戻る事故があった。出力フォーマットを明示固定すると再発しなくなった。

```yaml
direct_prompt: |
  （前略）
  ## 出力ルール（厳守）
  - すべて日本語。コード識別子以外に英文を混ぜない
  - 1指摘 = `path:line` + 1行理由 + 修正例（```diff）
  - 末尾に "重大度: 高/中/低" を必ず付ける
```

## 巨大PRをファイル単位に分割しコンテキスト溢れで$3溶かした失敗

1,200行のリファクタPRを丸ごと投げ、入力が18kトークンを超えてレビューが途中で打ち切られた。それでも課金は走り、再実行を3回繰り返して$3を無駄にした。ファイル単位に分割し、各ファイルを独立リクエストにして解決した。

```bash
# 変更ファイルごとにレビューを分割実行
git diff --name-only origin/${{ github.base_ref }}...HEAD | while read -r f; do
  git diff origin/${{ github.base_ref }}...HEAD -- "$f" > "/tmp/${f//\//_}.diff"
  # 1ファイル = 1リクエスト。1ファイルが400行超なら hunk 単位で更に分割
done
```

分割後はopus 4.8でも1ファイル平均1.9kトークンに収まり、打ち切りはゼロになった。「1リクエスト=1ファイル」を守るだけで、再実行による二重課金が消える。
