---
title: "第2章 GitHub ActionsにClaude Codeを常駐：.env秘匿とpull_request発火の最小yml"
free: false
---

## Secrets経由でANTHROPIC_API_KEYを渡す3行設定

APIキーをymlに直書きすると、PRログとリポジトリ履歴に平文で残る。`Settings > Secrets and variables > Actions` に `ANTHROPIC_API_KEY` を登録し、ジョブの `env` に注入する。これが3ヶ月でキー流出ゼロを維持した唯一の経路だ。

```yaml
jobs:
  review:
    runs-on: ubuntu-latest
    env:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # diff取得のため全履歴を取る
```

`fetch-depth: 0` を省くと `git diff origin/main` が空になり、Botが「変更なし」と誤判定して47件中12件を取りこぼした。

## pull_request と pull_request_target のフォークPRキー漏洩

`pull_request_target` はベースリポジトリの権限でSecretsを露出するため、フォークPRが悪意あるコードでキーを盗める。外部PRをレビュー対象にするなら `pull_request` に固定し、Secretsを渡さない読み取り専用ジョブと分離する。

```yaml
on:
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]
```

`synchronize` を入れないと、追加コミットでBotが再走しない。これに気づくまで2週間、初回コミットだけ審査される穴があった。

## permissions を contents:read に絞る最小権限

GITHUB_TOKEN はデフォルトで書き込み権限を持つ。レビューコメント投稿に必要なのは2つだけ。残りを `none` にすればトークン漏洩時の被害を限定できる。

```yaml
permissions:
  contents: read
  pull-requests: write
  # それ以外は暗黙でnone
```

## Claude Code CLIをnpxでCI実行する完成yml

`npx` でCLIを取得し、diffを標準入力で渡す。`--max-turns` を絞ると1PRあたり平均$0.04に収まった(3ヶ月で総額$11、検出47件)。

```yaml
      - name: Run Claude review
        run: |
          git diff origin/main...HEAD > /tmp/diff.patch
          npx @anthropic-ai/claude-code@latest \
            -p "以下のdiffを重大バグのみ指摘。なければOK:$(cat /tmp/diff.patch)" \
            --max-turns 3 > /tmp/review.txt
          gh pr comment ${{ github.event.number }} --body-file /tmp/review.txt
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## rate limit 429 と exit code 非ゼロの2つの罠

初週、連続PRで `429 Too Many Requests` が発生しCIが赤くなった。リトライをBash側で挟み、CLIの非ゼロ終了をマージブロック判定に正しく接続する。

```bash
for i in 1 2 3; do
  npx @anthropic-ai/claude-code@latest -p "$PROMPT" --max-turns 3 && break
  echo "429 retry $i"; sleep $((i * 20))
done
# 重大バグ検出時はexit 1を返し、required checkでmainマージを物理ブロック
grep -q "CRITICAL" /tmp/review.txt && exit 1 || exit 0
```

この `exit 1` を `required status check` に登録した瞬間から、Botの合否が本物の品質ゲートになり、3ヶ月の誤検知率は8.5%(47件中4件)に収まった。
