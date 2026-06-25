---
title: "第2章：個人開発環境を整える——Claude API・Python・GitHub Actionsの最小セットアップ"
free: true
---

## 第2章：個人開発環境を整える——Claude API・Python・GitHub Actionsの最小セットアップ

この章では、パイプラインを動かすための土台を作る。必要なのは3つだけ——**APIキー・Python仮想環境・GitHub Actionsのcronトリガー**。余分なものは一切入れない。

---

## 2-1. Claude APIキーを取得する

[console.anthropic.com](https://console.anthropic.com) にアクセスし、アカウントを作成する。左メニューの「API Keys」→「Create Key」でキーを発行する。表示されるのは一度きりなのでコピーしておく。

**落とし穴：** 無料枠は存在しない。最低$5のクレジットチャージが必要。ただし本書のパイプラインは月$1〜3程度で動く。まず$5入れて試すのが現実的な判断だ。

---

## 2-2. Python仮想環境を作る

プロジェクトルートで以下を実行する。

```bash
python -m venv .venv
source .venv/bin/activate      # Mac/Linux
.venv\Scripts\activate         # Windows PowerShell
```

必要なライブラリをインストールする。

```bash
pip install anthropic python-dotenv
pip freeze > requirements.txt
```

`requirements.txt` は必ず生成しておく。GitHub Actionsで同じ環境を再現するために使う。

---

## 2-3. APIキーを`.env`で管理する

キーをコードに直書きすると、GitHubに上げた瞬間に漏洩する。`.env`ファイルに分離する。

```
# .env
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxxxxxxx
```

Pythonコードからはこう読む。

```python
from dotenv import load_dotenv
import os

load_dotenv()
api_key = os.getenv("ANTHROPIC_API_KEY")
```

そして **`.gitignore`に必ず追加する**。

```
# .gitignore
.env
.venv/
```

**落とし穴：** `load_dotenv()` を書き忘れると `api_key` が `None` になり、APIコールが認証エラーで静かに落ちる。スクリプトの先頭で必ず呼ぶこと。

---

## 2-4. Claude APIの最小呼び出しを確認する

環境が正しく整ったかを確認するテストスクリプトを作る。

```python
# test_claude.py
from dotenv import load_dotenv
import os
import anthropic

load_dotenv()

client = anthropic.Anthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))

message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=256,
    messages=[{"role": "user", "content": "こんにちは。一言で自己紹介してください。"}]
)

print(message.content[0].text)
```

```bash
python test_claude.py
# → "Anthropicが開発したAIアシスタント、Claudeです。"
```

このレスポンスが返れば、APIキーと環境は正常だ。

---

## 2-5. GitHub ActionsでAPIキーを安全に渡す

自動実行するには、GitHubのリポジトリにAPIキーを登録する必要がある。コードには書かず、**Secretsに登録**する。

1. GitHubリポジトリの「Settings」→「Secrets and variables」→「Actions」
2. 「New repository secret」をクリック
3. 名前：`ANTHROPIC_API_KEY`、値：コンソールで取得したキーを貼る

---

## 2-6. GitHub Actionsのcronトリガーを設定する

`.github/workflows/daily.yml` を作成する。

```yaml
name: Daily Pipeline

on:
  schedule:
    - cron: '0 22 * * *'   # 毎日UTC 22:00 = JST 7:00
  workflow_dispatch:         # 手動実行ボタンも有効にする

jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run pipeline
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: python main.py
```

**落とし穴：** GitHubのcronはUTC基準。JST 7:00に動かしたければ `'0 22 * * *'`（前日UTC 22:00）と書く。`workflow_dispatch` を追加しておくと、手動でボタン1つでテスト実行できて開発中に便利だ。

**もう一つの落とし穴：** GitHub Actionsの無料枠はpublicリポジトリなら無制限、privateは月2,000分。毎日5分以内のジョブなら月150分で収まるので問題ない。

---

## チェックリスト

この章を終えた時点で、以下がすべて満たされているか確認する。

- [ ] `.env` にAPIキーが入っており、`.gitignore` に除外設定済み
- [ ] `python test_claude.py` でレスポンスが返る
- [ ] GitHubのSecretsに `ANTHROPIC_API_KEY` が登録済み
- [ ] `.github/workflows/daily.yml` がリポジトリにpush済み
- [ ] ActionsタブでWorkflowが表示され、手動実行が成功する

ここまで揃えば、次章から書くパイプライン本体を「毎朝自動実行」する器が完成している。
