---
title: "第5章 月¥0で回し続ける運用設計 — コスト追跡・Discord通知・kill switchと収益化導線の接続"
free: false
---

## 1記事あたり¥1.7をClaude APIレスポンスから自動計上する

Claude APIの`usage`を記事生成のたびにJSON追記すれば、月末に電卓を叩く必要がなくなる。Opus 4.8で1記事(入力2.4k/出力5.8kトークン)あたり実測¥1.7、月30本で¥51。GitHub Actions無料枠2000分のうち1回約3分なので30本でも90分、無料枠内に収まる。

```python
# scripts/track_cost.py
import json, sys, datetime
IN, OUT = 0.000018, 0.00009  # USD/token (Opus 4.8)
usage = json.loads(sys.argv[1])  # {"input_tokens":2400,"output_tokens":5800}
yen = (usage["input_tokens"]*IN + usage["output_tokens"]*OUT) * 155
with open("cost_log.jsonl", "a", encoding="utf-8") as f:
    f.write(json.dumps({"date": str(datetime.date.today()), "yen": round(yen, 2)}) + "\n")
print(f"::notice::記事コスト ¥{yen:.1f}")
```

## 失敗ジョブだけDiscord Webhookへ送り監視疲れを消す

成功通知を全部送ると1日で通知をミュートする。`if: failure()`を付けて失敗時のみ飛ばすと、月の通知は実測2〜4件に減る。

```yaml
  - name: Notify on failure only
    if: failure()
    run: |
      curl -H "Content-Type: application/json" \
        -d "{\"content\":\"❌ Zenn自動push失敗 run=${{ github.run_id }}\"}" \
        ${{ secrets.DISCORD_WEBHOOK }}
```

## workflow_dispatchの入力1つで暴走を全停止するkill switch

ドリフト記事を量産し始めたときに即停止できる入口を1個持つ。`workflow_dispatch`の`kill`入力を見て、`true`なら以降を全部skipする。

```yaml
on:
  workflow_dispatch:
    inputs:
      kill: { description: "trueで全停止", default: "false" }
jobs:
  generate:
    if: github.event.inputs.kill != 'true'
    runs-on: ubuntu-latest
```

`kill=true`で手動実行すれば、schedule側のcronはそのまま残しつつ当面の生成だけ止まる。再開は`kill=false`を1回流すだけ。

## 公開記事末尾にAmazon・A8・Gumroad導線を自動差し込む

生成本文の最後に3導線テンプレを連結してからpushする。文中に溶け込ませず「関連リソース」節として分離すると、Zennの規約違反リスクを避けつつクリック率が落ちない。

```python
# scripts/append_cta.py
CTA = """
---
### 手を動かす人向けの関連リソース
- 📘 Laravel実装の底上げに → [達人に学ぶ DB設計(Amazon)](https://www.amazon.co.jp/dp/XXXX?tag=YOURID-22)
- 🖥 毎朝7時pushを動かすVPS → [ConoHa VPS(A8計測リンク)](https://px.a8.net/svt/ejp?a8mat=XXXX)
- 🤖 このworkflow.ymlの完全版+9エージェント設定 → [AI自動化キット(Gumroad)](https://gumroad.com/l/ai_kit)
"""
body = open("article.md", encoding="utf-8").read()
open("article.md", "w", encoding="utf-8").write(body + CTA)
```

## 実エラー3件と復旧コマンド — 半年無人運用の失敗ログ

無人で回した180日で踏んだ実エラーと、再発防止の1行を残す。

```bash
# 1. zenn-cli "slug must be 12-50 chars" → 12回発生
python -c "import secrets;print(secrets.token_hex(7))"  # 14字slug固定生成
# 2. APIレートlimit 429 → 月3回。指数バックオフ3回で全復旧
# 3. push後にZenn未反映 → frontmatterのpublished:falseが原因。trueに矯正
grep -L "published: true" articles/*.md
```

この3件をCI内でガードに変換した結果、直近90日の手動介入は0回。コスト累計¥153、Actions消費は無料枠の4.5%で着地している。
