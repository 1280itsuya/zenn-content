---
title: "Claude API環境構築──キー取得・コスト設計・レート制限の罠"
free: true
---

## Claude API環境構築──キー取得・コスト設計・レート制限の罠

### APIキーの発行手順

[console.anthropic.com](https://console.anthropic.com) にアクセスし、Googleアカウントでサインアップする。登録直後は **Tier 1**（月$100上限）が適用される。

1. 左メニュー「API Keys」→「Create Key」
2. キー名を `auto-blog-prod` のように用途別に命名
3. 表示されるキーを即コピー（再表示不可）

`.env` ファイルに保存し、コードには直書きしない。

```bash
# .env
ANTHROPIC_API_KEY=sk-ant-api03-xxxxxx
```

```python
import anthropic
from dotenv import load_dotenv

load_dotenv()
client = anthropic.Anthropic()  # 環境変数から自動読み込み
```

---

### モデル選択とコスト設計

記事量産に使うモデルは目的で使い分ける。

| モデル | 用途 | 入力/出力 |
|--------|------|-----------|
| claude-haiku-4-5 | 下書き・タグ生成 | $0.80/$4.00 /MTok |
| claude-sonnet-4-6 | 品質ゲート・校正 | $3.00/$15.00 /MTok |

1記事を500トークン入力・800トークン出力と仮定する。Haikuで月300記事を生成した場合：

```
入力: 500 × 300 = 150,000 tok = 0.15 MTok → $0.12
出力: 800 × 300 = 240,000 tok = 0.24 MTok → $0.96
合計: 約 $1.08/月
```

コストより先に詰まるのがレート制限だ。

---

### プロンプトキャッシュで入力コストを75%削減

毎回同じシステムプロンプト（構成指示・ペルソナ定義）を送っている場合、**Prompt Caching** を使うと2回目以降はキャッシュヒット扱いになり入力コストが75%減る。

```python
response = client.messages.create(
    model="claude-haiku-4-5-20251001",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": "あなたはSEOに強い技術ライターです。...(長い定型指示)...",
            "cache_control": {"type": "ephemeral"},  # ←ここがキモ
        }
    ],
    messages=[{"role": "user", "content": f"以下のテーマで記事を書いて: {topic}"}],
)
```

キャッシュが有効になるのはブロックが **1,024トークン以上** のとき。短すぎると効かない。

---

### 429エラーの罠と回避策

Tier 1ではリクエスト制限（RPM）とトークン制限（TPM）の2軸が同時に存在する。片方だけ守っても詰まる。

**よくある失敗パターン：**

```python
# NG：ループをそのまま回すと数十秒でブロック
for topic in topics:
    result = call_claude(topic)  # 429で落ちる
```

**正しい対処：指数バックオフ付きリトライ**

```python
import time
import anthropic

def call_with_retry(client, **kwargs):
    for attempt in range(5):
        try:
            return client.messages.create(**kwargs)
        except anthropic.RateLimitError:
            wait = 2 ** attempt  # 1, 2, 4, 8, 16秒
            print(f"429 hit. wait {wait}s...")
            time.sleep(wait)
    raise RuntimeError("Rate limit retry exhausted")
```

さらに記事を**バースト送信せず**、1リクエストごとに最低1秒スリープを挟むだけで安定度が大きく変わる。

```python
time.sleep(1.2)  # 50 RPM制限に対してバッファを持たせる
```

---

### 落とし穴まとめ

- **キーを `.gitignore` に入れ忘れる** → GitHub pushで即漏洩。`git secret` か `.env` の除外を最初に設定する
- **`load_dotenv()` を書き忘れる** → `ANTHROPIC_API_KEY` が読まれず認証エラーになる（静かにスキップされることも）
- **キャッシュブロックが1,024トークン未満** → キャッシュが効かないのに `cache_control` を書いたと思い込む
- **Tier 1のまま本番負荷をかける** → 月$100上限に達すると当月中は全APIが停止。コンソールでSpend Limitのアラートを必ず設定する

環境が正しく動くことを確認してから、次章の記事生成ロジックへ進む。
