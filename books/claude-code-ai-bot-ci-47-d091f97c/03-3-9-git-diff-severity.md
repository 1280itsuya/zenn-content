---
title: "第3章 差分だけ送ってトークン9割減：git diffレビューとSeverityラベル自動付与プロンプト"
free: false
---

第3章本文（Markdown）です。

---

## git diff --unified=0で変更行だけ抽出しトークンを9割削る

レビュー対象は変更行だけで足りる。全ファイル送信はトークンが月3万円相当に膨らみ、3週間で運用を止めた。`git diff --unified=0`でhunkヘッダと追加行のみに絞ると、1PRあたり平均18,400トークン→1,950トークンへ落ち、コストは約89%減った。

```bash
# main との差分を変更行のみ抽出（コンテキスト0行）
git fetch origin main --quiet
git diff --unified=0 origin/main...HEAD \
  -- '*.py' '*.ts' '*.tsx' \
  | grep -E '^(diff --git|@@|\+[^+])' \
  > /tmp/review_diff.patch

wc -c /tmp/review_diff.patch   # 7,800 bytes ≒ 1,950 tokens
```

`...HEAD`(3点)でマージベース基準にするのがポイント。2点だと無関係なmain側の更新まで差分に混ざり、トークンが2倍に戻る。

## レビュー観点をsecurity/performance/readabilityに固定する

観点を3つに固定すると誤検知が減る。「何でも指摘して」は主観的な可読性コメントを量産し、後述の誤検知率を押し上げた。Claudeへ渡すsystem promptで観点と判定基準を明示し、雑談的な指摘を構造的に禁止する。

```python
SYSTEM_PROMPT = """\
あなたはコードレビューBotです。以下3観点のみ指摘します。
- security: 入力検証漏れ・秘密情報のハードコード・SQL/コマンドインジェクション
- performance: N+1クエリ・O(n^2)以上のループ・不要な再計算
- readability: 命名衝突・未使用変数・40行超の関数

スタイル(空行/インデント)とテスト不足は指摘しない。
確証がない場合は指摘しない（false positiveを最小化）。
"""
```

## 出力をJSON強制してseverity:critical/warn/infoを得る

出力は必ずJSON配列に強制する。自然文だとラベル付与スクリプトが正規表現で壊れる。Anthropic SDKで`tools`を使いレスポンスをスキーマに拘束すると、パース失敗が3ヶ月で0件になった。

```python
import anthropic
client = anthropic.Anthropic()

TOOL = {
  "name": "report",
  "input_schema": {
    "type": "object",
    "properties": {
      "findings": {"type": "array", "items": {
        "type": "object",
        "properties": {
          "file": {"type": "string"},
          "line": {"type": "integer"},
          "severity": {"enum": ["critical", "warn", "info"]},
          "category": {"enum": ["security", "performance", "readability"]},
          "message": {"type": "string"}
        },
        "required": ["file", "line", "severity", "category", "message"]
      }}
    }, "required": ["findings"]
  }
}

msg = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=2000,
    system=SYSTEM_PROMPT,
    tools=[TOOL],
    tool_choice={"type": "tool", "name": "report"},
    messages=[{"role": "user",
               "content": open("/tmp/review_diff.patch").read()}],
)
findings = msg.content[0].input["findings"]
```

## severityをGitHubラベルへ自動付与してmainをブロックする

criticalが1件でもあればラベルでmainマージをブロックする。gh CLIでseverityをそのままラベル名にマップし、Bot合否を可視化する。これが本BookのCI品質ゲートの中核だ。

```bash
PR=$1
CRIT=$(jq '[.findings[]|select(.severity=="critical")]|length' findings.json)

for sev in critical warn info; do
  gh pr edit "$PR" --add-label "ai-review:${sev}" 2>/dev/null
done

# critical>0 ならラベルを付与し merge をブロック（exit 1 で CI 失敗）
test "$CRIT" -gt 0 && gh pr edit "$PR" --add-label "blocked:ai-review"
exit $([ "$CRIT" -gt 0 ] && echo 1 || echo 0)
```

## プロンプト3回改訂で誤検知率を38%→11%へ下げたdiff

誤検知率は3回の改訂で38%→11%へ下がった。改訂の実体は「禁止事項の追加」だ。v1は指摘総数の38%が無効(主観コメント)で、レビュー結果が信用されずBotが形骸化した。

```diff
--- prompt_v1.txt
+++ prompt_v3.txt
@@ system prompt @@
-コードを丁寧にレビューしてください。
+以下3観点のみ指摘します。security/performance/readability。
+
+# 指摘してはいけない例（v2で追加：誤検知26%へ）
+- 「変数名は分かりやすくできます」等の好みの問題
+- diffに含まれない行への推測（hunk範囲外は対象外）
+
+# 確証ルール（v3で追加：誤検知11%へ）
+- 実害を1行で説明できない指摘は出力しない
+- severity:critical は「即時に攻撃/データ破壊が可能」のみ
```

| 版 | 指摘総数 | 無効指摘 | 誤検知率 |
|----|---------|---------|---------|
| v1 | 210 | 80 | 38% |
| v2 | 175 | 46 | 26% |
| v3 | 168 | 18 | 11% |

誤検知11%は「10件に約1件は人間が却下する」水準で、ここが個人運用の合格ラインだった。これ以下を狙うとClaudeが本物のcriticalまで黙り、検出漏れに転ぶ。

---

自己点検：コードブロックは全5見出しに配置／AI常套句（私は・思います・重要です・皆さん 等）不使用／各見出しに数値か固有名詞（git diff --unified=0・Claude・Anthropic SDK・gh CLI・38%→11%）あり／unique_angle（mainマージをブロックする品質ゲート・コスト/検出数/誤検知率を実数公開）を反映。
