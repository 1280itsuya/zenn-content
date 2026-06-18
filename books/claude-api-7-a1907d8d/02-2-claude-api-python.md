---
title: "第2章：Claude APIセットアップとPython環境構築——ゼロからパイプラインの土台を作る"
free: true
---

## 第2章：Claude APIセットアップとPython環境構築——ゼロからパイプラインの土台を作る

---

## APIキーの取得と保護

Anthropic Consoleにアクセスし、アカウント登録後に「API Keys」メニューからキーを発行する。キーは`sk-ant-api03-...`形式で始まる文字列だ。

**絶対にやってはいけないこと：** キーをコードに直書きすること。GitHubに誤ってpushすると、スキャンボットが数秒以内に検出し悪用される。

---

## .envファイルによるキー管理

プロジェクトルートに`.env`を作成し、以下を記述する：

```bash
ANTHROPIC_API_KEY=sk-ant-api03-xxxxxxxxxxxx
```

`.gitignore`に必ず追加する：

```bash
echo ".env" >> .gitignore
```

Pythonからは`python-dotenv`で読み込む：

```python
from dotenv import load_dotenv
import os

load_dotenv()
api_key = os.getenv("ANTHROPIC_API_KEY")
```

**落とし穴：** `load_dotenv()`を呼ぶ前に`os.getenv()`を呼ぶと`None`が返る。ファイル先頭で必ず`load_dotenv()`を実行すること。

---

## Python環境構築

```bash
python -m venv .venv
.venv\Scripts\activate        # Windows
# source .venv/bin/activate   # Mac/Linux

pip install anthropic python-dotenv
```

バージョン固定はトラブル予防の基本だ：

```bash
pip freeze > requirements.txt
```

---

## 基本的なAPI呼び出し

```python
import anthropic
from dotenv import load_dotenv

load_dotenv()

client = anthropic.Anthropic()  # ANTHROPIC_API_KEYを自動参照

message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "AI副業の記事タイトルを5つ考えて"}
    ]
)

print(message.content[0].text)
```

`client`生成時にキーを明示する必要はない——環境変数を自動で参照する。

---

## レートリミット対策とリトライ設計

Claude APIには**1分あたりのリクエスト数制限（RPM）とトークン制限（TPM）**がある。無料・低Tierでは特に厳しく、バッチ処理で連続呼び出しすると`RateLimitError`が発生する。

本番で使えるリトライラッパーを実装する：

```python
import time
import anthropic
from dotenv import load_dotenv

load_dotenv()
client = anthropic.Anthropic()

def call_claude(prompt: str, retries: int = 3) -> str:
    for attempt in range(retries):
        try:
            msg = client.messages.create(
                model="claude-sonnet-4-6",
                max_tokens=1024,
                messages=[{"role": "user", "content": prompt}]
            )
            return msg.content[0].text
        except anthropic.RateLimitError:
            wait = 60 * (attempt + 1)   # 60秒→120秒→180秒
            print(f"レートリミット。{wait}秒後にリトライ")
            time.sleep(wait)
        except anthropic.APIStatusError as e:
            print(f"APIエラー: {e.status_code}")
            raise
    raise RuntimeError("リトライ上限に達しました")
```

**ポイント：** 指数バックオフ（待機時間を徐々に伸ばす）を使うことで、API側の過負荷を悪化させずに回復を待てる。`RateLimitError`と`APIStatusError`は別々にハンドルし、前者はリトライ、後者（認証エラーなど）は即時停止が正しい設計だ。

---

## コスト管理の基本

APIはトークン消費量に応じて課金される。パイプラインを回す前に、1呼び出しあたりのトークン数を把握しておく：

```python
# レスポンスからトークン数を確認
msg = client.messages.create(...)
print(f"入力: {msg.usage.input_tokens} / 出力: {msg.usage.output_tokens}")
```

`claude-haiku-4-5`はSonnetの約10分の1のコストで動作するため、大量バッチ処理の下書き生成にはHaikuを使い、最終仕上げだけSonnetに渡すという**2段階設計**がコスト削減に効く。

---

これで土台は完成だ。次章からこの`call_claude()`関数を核に、記事生成・投稿・収益化の各パイプラインを積み上げていく。
