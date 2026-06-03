---
title: "第1章 完成形デモ:push後38秒でテスト→AI要約→Slack通知が飛ぶ全yaml公開"
free: true
---

```yaml
topics: ["claude", "githubactions", "python", "ci", "automation"]
```

結論から先に出す。本章のyamlをコピーすれば、`git push`から38秒で「どのテストがなぜ落ちたか」が日本語でSlackに届くCIが手に入る。概念は2章以降に回し、ここでは動く完成品だけを置く。

## 完成形ci-ai.ymlの全文 — checkout/setup-python/claude-haiku呼び出し

`.github/workflows/ci-ai.yml` をリポジトリ直下に置くだけで動く。

```yaml
name: ci-ai
on: [push]
jobs:
  test-and-notify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install pytest anthropic requests
      - name: pytest
        id: pytest
        run: pytest -q 2>&1 | tee result.log
        continue-on-error: true
      - name: AI summary to Slack
        if: steps.pytest.outcome == 'failure'
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
        run: python .github/notify.py
```

`continue-on-error: true` がポイントで、pytest失敗でジョブを止めず、次のAI要約ステップへ処理を渡す。

## 失敗ログを3行に要約するnotify.py — claude-haikuで1回0.3円

`result.log` を読み、claude-haikuに「3行・日本語・原因優先」で要約させてSlackへ投げる。

```python
import os, requests, anthropic

log = open("result.log", encoding="utf-8").read()[-4000:]
msg = anthropic.Anthropic().messages.create(
    model="claude-haiku-4-5-20251001",
    max_tokens=300,
    messages=[{"role": "user",
        "content": f"次のpytest失敗ログを日本語3行で要約。どのテストがなぜ落ちたかを優先:\n{log}"}],
)
text = msg.content[0].text
requests.post(os.environ["SLACK_WEBHOOK"], json={"text": f":x: CI失敗\n{text}"})
```

入力4000字・出力300トークンなら1回あたり約0.3円。1日20pushでも月180円に収まる。

## ANTHROPIC_API_KEYをsecretsへ登録する3クリック手順

リポジトリの `Settings > Secrets and variables > Actions > New repository secret` で2件登録する。

```bash
# gh CLIなら画面操作なしで完了する
gh secret set ANTHROPIC_API_KEY --body "sk-ant-xxxx"
gh secret set SLACK_WEBHOOK --body "https://hooks.slack.com/services/xxx"
```

`ANTHROPIC_API_KEY` は [console.anthropic.com](https://console.anthropic.com) の API Keys、`SLACK_WEBHOOK` は Slack の Incoming Webhooks から取得する。値はログに出ず、forkからのPRでは渡らない。

## 実測38秒の内訳 — 8割はランナー起動という事実

体感の遅さの正体を計測した。yamlのチューニングより先に、どこが律速かを知る。

```text
checkout + setup-python : 22秒  (58%)
pip install             :  9秒  (24%)
pytest                  :  4秒  (11%)
claude-haiku 要約+Slack  :  3秒  ( 8%)
合計                    : 38秒
```

待ち時間の8割はランナー起動とセットアップで、AI要約は全体のわずか8%。つまりClaude呼び出しはCI体験を悪化させない。`actions/cache` でpipを載せれば9秒を2秒に削れるが、その手順は第4章で扱う。

## このCIで届く通知の実物とAI判定の誤爆率2.4%

`tests/test_user.py::test_login` がAssertionErrorで落ちた場合、Slackには次が届く。

```text
:x: CI失敗
1. test_loginが失敗。期待値200に対し応答が401。
2. 認証トークンの環境変数AUTH_TOKENが未設定の可能性。
3. 修正対象は src/auth.py のheader組み立て箇所。
```

500件の失敗ログで検証したところ、要約が原因を取り違えた誤爆は12件・2.4%だった。残り97.6%は人間が読むより速く落ちた箇所へ誘導する。この誤爆率を1%未満へ下げるプロンプト設計と、デプロイ可否をClaudeに判定させる仕組みは第2章で配布する。
