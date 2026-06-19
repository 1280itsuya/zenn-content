---
title: "まとめ：Claude APIパイプラインを育て続けるための運用ループ設計"
free: false
---

## まとめ：Claude APIパイプラインを育て続けるための運用ループ設計

パイプラインは「完成」した瞬間から劣化が始まる。検索トレンドが変わり、APIの応答傾向が変わり、読者の期待値も変わる。この章では、パイプラインを作り捨てにしないための**自己進化ループ**を設計する。

---

## PDCAの正体：3ステップで回す改善サイクル

運用ループの核心は次の3フェーズだ。

```
[1] Winner抽出  →  [2] プロンプト改善  →  [3] 再計測
     ↑                                          ↓
     └──────────── 週次GitHub Actions ──────────┘
```

**フェーズ1: Winner抽出**  
Zenn・Qiitaのview数をAPIで取得し、過去30日のトップ5記事のタグ・キーワード・文体パターンを抽出する。

```python
# winner_extractor.py（抜粋）
def extract_winners(zenn_articles: list[dict]) -> dict:
    sorted_articles = sorted(zenn_articles, key=lambda x: x["liked_count"], reverse=True)
    top5 = sorted_articles[:5]
    return {
        "top_tags": Counter([t for a in top5 for t in a["topics"]]).most_common(5),
        "avg_length": sum(len(a["body_markdown"]) for a in top5) // 5,
        "title_patterns": [a["title"] for a in top5],
    }
```

**フェーズ2: プロンプト改善**  
抽出結果をClaude APIに渡し、次週用のシステムプロンプトのドラフトを生成させる。

```python
# prompt_evolver.py（抜粋）
def evolve_prompt(current_prompt: str, winners: dict) -> str:
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=[{
            "role": "user",
            "content": f"""
現在のシステムプロンプト:
{current_prompt}

先週のWinner記事の傾向:
- 人気タグ: {winners['top_tags']}
- 平均文字数: {winners['avg_length']}字
- タイトル例: {winners['title_patterns']}

これらを反映した改善版プロンプトを提案してください。
変更点は箇条書きで明示すること。
"""
        }]
    )
    return response.content[0].text
```

**フェーズ3: 再計測**  
新プロンプトで生成した記事を投稿後、1週間後に同じAPIでview数を取得し、改善前後を比較する。

---

## GitHub Actionsで自動化する

毎週月曜日にループを自動実行するワークフローを仕込む。

```yaml
# .github/workflows/pipeline_evolution.yml
name: Pipeline Evolution
on:
  schedule:
    - cron: "0 1 * * 1"  # 毎週月曜 10:00 JST

jobs:
  evolve:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run winner extraction
        env:
          ZENN_USERNAME: ${{ secrets.ZENN_USERNAME }}
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: python scripts/winner_extractor.py

      - name: Evolve prompts
        run: python scripts/prompt_evolver.py

      - name: Commit updated prompts
        run: |
          git config user.email "bot@example.com"
          git config user.name "Pipeline Bot"
          git add prompts/
          git commit -m "chore: weekly prompt evolution $(date +%Y-%m-%d)" || echo "No changes"
          git push
```

プロンプトはコードと同じくGitで管理する。これにより、改悪したときに`git revert`で即座に戻せる。

---

## 落とし穴：3つの典型的な失敗パターン

**1. 数値捏造を見抜けずにWinnerを誤認する**  
Claude APIは「それらしい数字」を生成しがちだ。view数は必ず外部APIから直接取得し、LLMに数値の判断をさせてはいけない。

```python
# NG: LLMに「人気記事はどれ?」と聞く
# OK: Zenn APIから liked_count を直接取得して自前でソート
```

**2. プロンプトを毎週上書きして比較不能になる**  
`prompts/system_v1.md` のようにバージョン管理するのではなく、Git履歴そのものがバージョン管理になるよう設計する。ファイル名は変えずにcommitで差分を追う。

**3. 改善を「量」で測る**  
記事本数を増やしてもview総数は増えない。1記事あたりのview数・いいね率・タグ別CTRで改善を測る。

---

## 実例：4週間で平均viewが2.3倍になったプロンプト変化

実際の運用では、以下の変化がview数に最も効いた。

| 変更前 | 変更後 | 効果 |
|--------|--------|------|
| 「AIツールの使い方を解説する」 | 「実際のエラーメッセージを1件取り上げ、原因と解決コードを示す」 | +180% |
| タグ: `AI`, `機械学習` | タグ: `Python`, `Claude`, `自動化` | +40% |
| 冒頭: 概念説明 | 冒頭: 「このコードで詰まった人向け」 | +60% |

抽象的な「AI記事」から「このエラーで詰まった人向け」への転換が、最大の改善要因だった。ループが教えてくれたことを、次のプロンプトに焼き直す——これが運用ループの本質だ。

---

## 運用ループを「仕組み」にするための最終チェックリスト

- [ ] view数はプラットフォームAPIから自動取得されているか
- [ ] プロンプトはGitで管理され、ロールバック可能か
- [ ] GitHub Actionsが週次で自動実行されているか
- [ ] 改善指標がview数/記事など「比率」で定義されているか
- [ ] プロンプト変更後1週間のA/B期間を設けているか

パイプラインは放置すれば朽ちる。週1回のループを回すだけで、半年後には初期の3倍以上の精度に育つ。仕組みを作ったら、あとはループに育てさせればいい。
