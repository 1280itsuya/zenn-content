---
title: "生成品質を担保する自動チェックと落とし穴への対処法"
free: false
---

## 生成品質を担保する自動チェックと落とし穴への対処法

自動生成パイプラインが動き始めると、次に直面するのが「量は出るが質が壊れる」問題だ。放置すると検索エンジンにスパム認定され、API料金が月数万円に跳ね上がり、最悪の場合アカウントを失う。本章では4大リスクとその回避コードを示す。

---

## リスク1：コンテンツ重複

同じキーワードで複数記事を生成すると、導入文や結論がほぼ同文になる。Googleはこれを重複コンテンツとして評価を下げる。

```python
import hashlib, json
from pathlib import Path

HASH_DB = Path("data/content_hashes.json")

def is_duplicate(text: str, threshold: float = 0.85) -> bool:
    hashes = json.loads(HASH_DB.read_text()) if HASH_DB.exists() else {}
    # 先頭200字のハッシュで高速チェック
    snippet_hash = hashlib.sha256(text[:200].encode()).hexdigest()
    if snippet_hash in hashes:
        return True
    hashes[snippet_hash] = True
    HASH_DB.write_text(json.dumps(hashes))
    return False
```

**落とし穴**：本文全体ではなく冒頭200字だけでハッシュを取るのがポイント。全文比較はI/Oが重く、同じテーマで書き出しが似るだけで弾かれすぎる。

---

## リスク2：プロンプトドリフト

「節約術」を書かせるつもりが、途中から「投資入門」になっている——これがプロンプトドリフトだ。長いシステムプロンプトを使うほど起きやすい。

```python
def validate_theme(title: str, body: str, keyword: str) -> bool:
    """タイトルのキーワードが本文に一定頻度登場するか検証"""
    freq = body.count(keyword) / max(len(body.split()), 1)
    if keyword not in title:
        print(f"[WARN] タイトルにキーワード欠落: {keyword}")
        return False
    if freq < 0.003:  # 1000字に3回未満は主題ズレ疑い
        print(f"[WARN] 本文頻度不足: {freq:.4f}")
        return False
    return True
```

チェックが失敗したら生成をスキップし、ログに残す。リトライは1回まで——無限再生成はコスト爆発の入口だ。

---

## リスク3：API料金爆発

最もやっかいなのがこれだ。エラーループや長大プロンプトの乱用で、一晩で数万円溶けることがある。

```python
import anthropic

client = anthropic.Anthropic()

DAILY_TOKEN_LIMIT = 500_000  # 1日上限（入力+出力合算）
daily_usage = 0

def safe_generate(prompt: str, max_tokens: int = 1200) -> str | None:
    global daily_usage
    estimated = len(prompt) // 4 + max_tokens  # 粗い推定
    if daily_usage + estimated > DAILY_TOKEN_LIMIT:
        print("[ABORT] 本日のトークン上限到達")
        return None

    resp = client.messages.create(
        model="claude-haiku-4-5-20251001",  # 量産はHaikuで十分
        max_tokens=max_tokens,
        messages=[{"role": "user", "content": prompt}],
    )
    used = resp.usage.input_tokens + resp.usage.output_tokens
    daily_usage += used
    return resp.content[0].text
```

**実例**：プロンプトに誤って全ログファイルを埋め込んでしまい、1リクエスト30,000トークンを1,000回実行——という事故が起きた。`estimated` チェックを入れるだけで止められる。

---

## リスク4：規約違反

Claude APIの利用規約は「他者を欺くコンテンツの大量生成」を禁じている。アフィリ記事でアウトになるのは主に次の3パターンだ。

| パターン | 具体例 | 対策 |
|---|---|---|
| 虚偽実績 | 「月収100万達成」の根拠なし主張 | 免責文を自動挿入 |
| ステマ | アフィリリンクを広告表示なしで掲載 | `AD_NOTICE` 定数を先頭に強制付与 |
| 機械生成の明記義務回避 | 「人間が執筆」と誤認させる | プロフィールに「AIアシスト」を明記 |

```python
AD_NOTICE = "※本記事にはアフィリエイトリンクが含まれます。\n\n"

def attach_compliance(body: str) -> str:
    return AD_NOTICE + body
```

---

## まとめ：チェックを「直列」に並べる

```python
def publish_pipeline(keyword: str, prompt: str) -> bool:
    body = safe_generate(prompt)
    if body is None:
        return False
    if is_duplicate(body):
        return False
    if not validate_theme(keyword, body, keyword):
        return False
    body = attach_compliance(body)
    # 投稿処理へ
    return True
```

4つのチェックを直列に置き、1つでも失敗したら即中断する。スキップされた件数を日次ログに残しておくと、プロンプト改善の手がかりになる。量より精度——この原則が長期運用でのアカウント生存率を決める。
