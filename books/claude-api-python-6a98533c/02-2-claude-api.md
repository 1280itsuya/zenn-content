---
title: "第2章　Claude APIパイプラインの環境構築とコスト設計"
free: true
---

## 第2章　Claude APIパイプラインの環境構築とコスト設計

---

## 2-1　仮想環境の作成

プロジェクトごとにパッケージを隔離するため、`venv` を必ず使う。グローバル環境を汚すと依存関係が衝突し、後から原因追跡が困難になる。

```bash
# プロジェクトルートで実行
python -m venv .venv

# 有効化（Windows PowerShell）
.venv\Scripts\Activate.ps1

# 有効化（Mac / Linux）
source .venv/bin/activate

# 必要パッケージを一括インストール
pip install anthropic python-dotenv
pip freeze > requirements.txt
```

**落とし穴**：`.venv` ディレクトリを Git にコミットしないこと。`.gitignore` に `.venv/` を追加しておく。

---

## 2-2　APIキーの取得と安全な管理

1. [https://console.anthropic.com](https://console.anthropic.com) にアクセスし、**API Keys** → **Create Key**
2. 生成された `sk-ant-...` の文字列をコピーする（画面を閉じると二度と表示されない）
3. キーをコードに直書きしない。必ず `.env` ファイルで管理する

**絶対にやってはいけないこと**：

```python
# ❌ 絶対NG — GitHub に push すると即アカBANリスク
client = anthropic.Anthropic(api_key="sk-ant-api03-xxxxxxxxxxxx")
```

---

## 2-3　`.env` ファイルの設計

プロジェクトルートに `.env` を作成し、以下の形式で記述する。

```dotenv
# Anthropic
ANTHROPIC_API_KEY=sk-ant-api03-xxxxxxxxxxxx

# モデル設定（変更時に一箇所だけ直す）
CLAUDE_MODEL=claude-sonnet-4-6
CLAUDE_MAX_TOKENS=2048

# コスト上限（月次ガード用）
MONTHLY_BUDGET_USD=10.00
```

コード側での読み込みは以下の通り。`load_dotenv()` を必ず呼ぶのを忘れないこと（これを忘れると `ANTHROPIC_API_KEY` が `None` になり、実行時エラーが出る）。

```python
import os
from dotenv import load_dotenv
import anthropic

load_dotenv()  # ← これを忘れると動かない

client = anthropic.Anthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))
model  = os.getenv("CLAUDE_MODEL", "claude-sonnet-4-6")
```

`.env` は `.gitignore` に追加する。

```
# .gitignore
.env
.venv/
__pycache__/
```

---

## 2-4　月額コスト試算

Claude Sonnet 4.6 の料金（2025年8月時点）は入力 $3/MTok・出力 $15/MTok。ブログ記事1本を自動生成する場合の概算：

| 項目 | トークン数 | コスト |
|------|-----------|--------|
| システムプロンプト（入力） | 約 500 tok | $0.0015 |
| 記事本文生成（出力） | 約 1,500 tok | $0.0225 |
| **1記事合計** | — | **約 $0.024（≒¥3.6）** |

月30本生成しても **約¥110**。1日1本の自動投稿なら月額コストはコーヒー1杯以下に収まる。

コストを監視するユーティリティを用意しておくと安心だ。

```python
def estimate_cost(input_tokens: int, output_tokens: int) -> float:
    """Sonnet 4.6 の概算コスト（USD）を返す"""
    input_cost  = input_tokens  / 1_000_000 * 3.0
    output_cost = output_tokens / 1_000_000 * 15.0
    return input_cost + output_cost
```

Anthropic コンソールの **Usage** タブで当月の実績を週1回確認する習慣をつけると、意図しない大量呼び出しを早期に検知できる。

---

## 2-5　動作確認

環境が正しく整ったか、最小構成で確認する。

```python
# check_env.py
import os
from dotenv import load_dotenv
import anthropic

load_dotenv()

client = anthropic.Anthropic()
message = client.messages.create(
    model=os.getenv("CLAUDE_MODEL", "claude-sonnet-4-6"),
    max_tokens=64,
    messages=[{"role": "user", "content": "「環境構築完了」とだけ答えてください"}],
)
print(message.content[0].text)
```

`環境構築完了` と出力されれば、次章のパイプライン実装に進める準備が整っている。

---

> **本章のポイント**：`.env` によるキー隔離・`load_dotenv()` の呼び忘れ防止・月額コストの事前試算——この3点を押さえておけば、後続の自動生成スクリプトで「なぜか動かない」「気づいたら請求が膨らんでいた」という躓きを回避できる。
