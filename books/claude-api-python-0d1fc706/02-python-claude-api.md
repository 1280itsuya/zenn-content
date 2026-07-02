---
title: "環境構築と認証：PythonからClaude APIを叩く最短手順"
free: false
---

## 環境構築と認証：PythonからClaude APIを叩く最短手順

### 1. ライブラリのインストール

まず仮想環境を作り、依存ライブラリをインストールする。

```bash
python -m venv .venv
# Windows
.venv\Scripts\activate
# macOS / Linux
source .venv/bin/activate

pip install anthropic python-dotenv
```

`python-dotenv` は後述の `.env` 管理に使う。`anthropic` が公式SDKで、これ一本でAPI呼び出しからリトライ制御まで賄える。

---

### 2. APIキーの取得と安全な管理

[Anthropicコンソール](https://console.anthropic.com) でAPIキーを発行する。キーは `sk-ant-` で始まる文字列だ。

**絶対にやってはいけないこと：コードにハードコーディングする**

```python
# ❌ 悪い例 — Gitに上げると即漏洩
client = anthropic.Anthropic(api_key="sk-ant-xxxxxxxxxxxx")
```

Gitに上がった瞬間に第三者に悪用されるリスクがある。キーが漏れたらコンソールで即時無効化して再発行する。

---

### 3. .env ファイルで認証情報を管理

プロジェクトルートに `.env` を作成する：

```
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxx
```

`.gitignore` に必ず追加する：

```
.env
.venv/
```

コードでは `load_dotenv()` を使ってロードし、SDKが自動的に `ANTHROPIC_API_KEY` 環境変数を参照する：

```python
from dotenv import load_dotenv
import anthropic

load_dotenv()  # ← 必ずAnthropic()の前に呼ぶ

client = anthropic.Anthropic()
```

**落とし穴**：`load_dotenv()` より前に `Anthropic()` を初期化すると、環境変数が読まれずに `AuthenticationError` が発生する。順序を間違えると動かない。

---

### 4. 最初のAPI呼び出しで動作確認

```python
from dotenv import load_dotenv
import anthropic

load_dotenv()
client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=512,
    messages=[
        {"role": "user", "content": "「副業」について50字で要約してください。"}
    ]
)

print(response.content[0].text)
print(f"使用トークン: {response.usage.input_tokens} in / {response.usage.output_tokens} out")
```

`response.content` はリストなので、テキストは `[0].text` で取り出す。`response.usage` で消費トークン数を確認すると、コスト管理がしやすい。

---

### 5. レートリミット対策

大量にリクエストを送ると HTTP 429（`RateLimitError`）が返ってくる。**SDKはデフォルトで429と5xxを自動リトライ（指数バックオフ、最大2回）** するが、アフィリ記事を数十本まとめて生成するパイプラインではさらに対策が必要だ。

```python
import time
from dotenv import load_dotenv
import anthropic

load_dotenv()
client = anthropic.Anthropic(max_retries=5)  # リトライ上限を増やす

def generate_article(topic: str) -> str:
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        messages=[{"role": "user", "content": f"{topic}に関するアフィリ記事を書いてください。"}]
    )
    return response.content[0].text

topics = ["節約術", "副業アイデア", "投資入門"]

for topic in topics:
    article = generate_article(topic)
    print(article[:80])
    time.sleep(2)  # リクエスト間に2秒待機してレートリミットを回避
```

**注意点**：モデルは量産コスト優先で `claude-sonnet-4-6` を選ぶのが現実的だ。`claude-opus-4-8` は品質最優先の場面に絞ると費用対効果が上がる。

---

### 確認チェックリスト

| 症状 | 原因 | 対処 |
|---|---|---|
| `AuthenticationError` | `load_dotenv()` 呼び忘れ / APIキー未設定 | `.env` の記述確認 + 呼び出し順序を修正 |
| `RateLimitError` が多発 | リクエスト過多 | `time.sleep()` 追加・`max_retries` を増やす |
| `IndexError` on `content[0]` | `content` が空（拒否レスポンス） | `response.stop_reason` を確認してから参照する |
| APIキーがGitに漏洩 | ハードコーディング | コンソールで即時無効化 → 再発行 |

インストール・キー管理・`.env` 設定・レートリミット対策、この4点を押さえればローカル環境でClaude APIを安全に動かせる。次章から、この基盤の上にアフィリ記事の自動生成パイプラインを組み上げていく。
