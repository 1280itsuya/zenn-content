---
title: "Claude API×Python環境構築：パイプラインの土台を作る"
free: false
---

## 1. Anthropic ConsoleでAPIキーを取得する

まず [console.anthropic.com](https://console.anthropic.com) にアクセスしてアカウントを作成する。左サイドバーの **API Keys** から「Create Key」をクリックし、生成されたキーをコピーしておく。

> **落とし穴**: キーはこの画面を閉じると二度と表示されない。必ずすぐにどこかに控えること。

---

## 2. プロジェクトディレクトリとvenvを作る

```bash
mkdir article-pipeline && cd article-pipeline
python -m venv .venv

# Windows (PowerShell)
.\.venv\Scripts\Activate.ps1

# Mac / Linux
source .venv/bin/activate
```

仮想環境を使う理由は単純で、`anthropic` ライブラリのバージョンをプロジェクト単位で固定し、他のプロジェクトと干渉させないためだ。`pip install` の前に必ず `(.venv)` がプロンプトに表示されていることを確認する。

---

## 3. 必要パッケージをインストールする

```bash
pip install anthropic python-dotenv
pip freeze > requirements.txt
```

`anthropic` が Claude API のクライアント、`python-dotenv` が `.env` ファイルを読み込むためのライブラリだ。`requirements.txt` に書き出しておくことで、別マシンへの移行時に `pip install -r requirements.txt` 一発で環境が再現できる。

---

## 4. `.env` でAPIキーを管理する

プロジェクトルートに `.env` ファイルを作り、キーを書く。

```
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxxxxxxxxxxxxx
```

次に `.gitignore` を作って `.env` を除外する。これを忘れると GitHub にキーが公開される。

```
# .gitignore
.env
.venv/
__pycache__/
```

> **実際にやらかしたパターン**: `.env` をコミットしてしまい、Anthropic から「キーが公開されています」というメールが届く。GitHub はリポジトリをクロールしてキー漏洩を検知するため、push してから数分で無効化される。

---

## 5. 動作確認：最小限のAPI呼び出し

```python
# test_api.py
from dotenv import load_dotenv
import anthropic
import os

load_dotenv()  # .env を読み込む

client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

message = client.messages.create(
    model="claude-haiku-4-5-20251001",
    max_tokens=256,
    messages=[{"role": "user", "content": "テスト。一言で返して。"}]
)

print(message.content[0].text)
```

```bash
python test_api.py
# → "了解しました。"
```

`claude-haiku-4-5-20251001` を使うのは、テスト段階でコストを抑えるためだ。記事生成パイプライン本番では `claude-sonnet-4-6` に切り替える。

---

## 6. この章のチェックリスト

| 項目 | 確認 |
|------|------|
| `.venv` が有効化されている | `(.venv)` がプロンプトに表示される |
| `.env` にキーが記載されている | `ANTHROPIC_API_KEY=sk-ant-...` |
| `.gitignore` で `.env` を除外済み | `git status` に `.env` が出ない |
| `test_api.py` が返答を返す | エラーなく文字列が表示される |

この4点が揃えば、次章から書く記事生成エージェントの土台は完成だ。
