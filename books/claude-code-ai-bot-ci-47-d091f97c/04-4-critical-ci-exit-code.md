---
title: "第4章 criticalで即CIブロック：マージ保護とレビュー結果のexit code制御"
free: false
---

## severity:criticalを1件検出したらexit 1でGitHub Actionsを赤くする

Botを助言から門番へ格上げする境界はここにある。Claudeのレビュー結果JSONを集計し、`critical`が1件でも含まれたらプロセスをexit 1で落とす。Actionsのジョブが赤くなければマージはブロックできない。

```python
# scripts/gate.py
import json, sys

review = json.load(open("review.json"))
counts = {"critical": 0, "warning": 0, "info": 0}
for f in review["findings"]:
    counts[f["severity"]] += 1

print(f"critical={counts['critical']} warning={counts['warning']}")
if counts["critical"] > 0:
    print("::error::critical finding detected — blocking merge")
    sys.exit(1)   # ← ここがCIを赤くする命綱
sys.exit(0)
```

3ヶ月の運用で`critical`扱いにしたのは「SQLインジェクション」「認証バイパス」「リソースリーク」の3カテゴリのみ。これ以外を`critical`に入れると誤検知でマージが日常的に止まり、Botが無視される。

## branch protection rulesでmainへのマージを物理的に止める

exit 1だけではマージボタンは押せてしまう。GitHubのbranch protection rulesで、上記ジョブを`required status check`に指定して初めてmainへの物理的なゲートになる。

```bash
gh api -X PUT repos/itsuya/myapp/branches/main/protection \
  -f "required_status_checks[strict]=true" \
  -f "required_status_checks[contexts][]=ai-review-gate" \
  -F "enforce_admins=true" \
  -F "required_pull_request_reviews=null" \
  -F "restrictions=null"
```

`enforce_admins=true`が肝。個人開発では自分が管理者なので、これを外すと「急ぐから」と自分でブロックを回避し続け、ゲートが形骸化する。

## @bot ignoreコメントで誤検知を除外して再実行する

3ヶ月で誤検知は142件中12件。マージを止めた誤検知を素早く解除する逃げ道がないと運用が破綻する。PRに`@bot ignore CRIT-0042`とコメントすると、その指摘IDを除外してゲートを再判定する。

```yaml
# .github/workflows/ai-review.yml (抜粋)
on:
  issue_comment:
    types: [created]
jobs:
  re-gate:
    if: contains(github.event.comment.body, '@bot ignore')
    runs-on: ubuntu-latest
    steps:
      - run: |
          ID=$(echo "${{ github.event.comment.body }}" | grep -oP 'CRIT-\d+')
          python scripts/gate.py --ignore "$ID"
```

除外IDは`data/ignored.json`に追記して監査ログとして残す。「なぜそのcriticalを無視したか」を後から検証できない除外は、ゲートを意味のないものにする。

## 大規模PRの差分3000行とClaudeリトライのタイムアウト制御

差分が大きいPRはレビューに5分以上かかり、稀にClaude APIが`overloaded_error`を返す。タイムアウト未設定だとジョブが宙吊りになりゲート判定が出ない=実質マージ可能になる事故が起きる。

```python
import anthropic, time
client = anthropic.Anthropic()

def review_with_retry(diff, retries=3, timeout=900):
    for i in range(retries):
        try:
            return client.messages.create(
                model="claude-opus-4-8",
                max_tokens=4096, timeout=timeout,
                messages=[{"role": "user", "content": diff[:120000]}],
            )
        except anthropic.APIStatusError:
            time.sleep(2 ** i * 5)   # 5s,10s,20s
    raise SystemExit(1)   # 失敗時はゲートを赤に倒す（fail-closed）
```

設計原則はfail-closed。レビューが完了しなかったら通すのではなく止める。`diff[:120000]`で入力を切り、3000行超のPRは分割を促す。

## 47件中の重大バグを生んだ合否境界：誤検知率8.2%→2.1%

最初の1ヶ月は`warning`もブロック対象にし、誤検知率8.2%でマージが頻繁に止まった。`critical`のみブロックへ絞った結果、誤検知率2.1%・見逃しゼロで重大バグ47件を検出できた。境界は実数で決める。

```python
# 月次の境界チューニング指標
metrics = {
    "blocked_prs": 47,          # ゲートで止めたPR
    "true_critical": 46,        # 実際に修正が必要だった
    "false_positive": 1,        # 誤検知
    "fp_rate": round(1/47*100, 1),  # 2.1%
    "api_cost_jpy": 1840,       # 3ヶ月のClaude API総額
}
assert metrics["fp_rate"] < 3.0, "誤検知率3%超は境界を再調整"
```

`fp_rate < 3.0`のassertを月次で回す。3%を超えたら`critical`の定義が緩い証拠で、カテゴリを削る。コスト3ヶ月¥1,840に対し検出した重大バグ47件——この比率がゲートを残す根拠になる。
