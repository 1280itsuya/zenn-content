---
title: "環境構築：Claude APIパイプラインの土台を5分で作る"
free: false
---

## 必要なものを揃える

Python 3.9以上とpipが入っていれば準備完了だ。まず作業ディレクトリを作り、仮想環境を立ち上げる。

```bash
mkdir claude-pipeline && cd claude-pipeline
python -m venv .venv
# Windows
.venv\Scripts\activate
# Mac/Linux
source .venv/bin/activate

pip install anthropic python-dotenv
```

仮想環境を忘れてグローバルにインストールすると、後でプロジェクトをまたいだバージョン衝突に悩まされる。必ずここで分離しておく。

## APIキーを安全に管理する

[Anthropic Console](https://console.anthropic.com/) でAPIキーを発行したら、絶対にソースコードに直書きしない。代わりに`.env`ファイルに書く。

```
# .env
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxxxxxxxxx
```

そして**.gitignoreに必ず追加する**。これを忘れてGitHubにプッシュした瞬間、キーは即無効化される（Anthropicの自動検知システムが動く）。

```
# .gitignore
.env
.venv/
__pycache__/
```

落とし穴：`.env`を後から`.gitignore`に追加しても、すでにコミット済みなら履歴に残る。`git rm --cached .env`で追跡を外してから再コミットすること。

## 最初のリクエストを送る

環境変数を読み込んでAPIを呼び出す最小構成がこれだ。

```python
# main.py
from anthropic import Anthropic
from dotenv import load_dotenv

load_dotenv()  # .envを読み込む

client = Anthropic()  # ANTHROPIC_API_KEYを自動参照

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Pythonで「Hello, Pipeline!」と表示するコードを書いて"}
    ]
)

print(response.content[0].text)
```

`load_dotenv()`を呼ぶ前に`Anthropic()`を初期化すると環境変数が読めずエラーになる。順序に注意。

## レスポンス構造を把握する

`response`オブジェクトには使用トークン数も含まれている。コスト管理のために最初から記録する習慣をつけると後で楽になる。

```python
print(f"入力: {response.usage.input_tokens} tokens")
print(f"出力: {response.usage.output_tokens} tokens")
print(f"停止理由: {response.stop_reason}")  # end_turn / max_tokens
```

`stop_reason`が`max_tokens`なら出力が途中で切れている。長文生成では`max_tokens`を余裕を持って設定しておく。

## 動作確認

```bash
python main.py
```

`print(hello)` と返ってくれば環境構築は完了だ。次章からはこのクライアントをラップして、ブログ記事・SNS投稿・Kindle原稿を自動生成するパイプラインに育てていく。
