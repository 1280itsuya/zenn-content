---
title: "第5章：Claude API利用コストとアカウントBANを避けるための落とし穴10選"
free: false
---

## 第5章：Claude API利用コストとアカウントBANを避けるための落とし穴10選

実運用で筆者が実際に踏んだ失敗を、コスト実測値と回避策とともに列挙する。

---

### 落とし穴① 巨大システムプロンプトを毎回送信してトークン爆死

SEOルール・文体指定・禁止表現を1つのシステムプロンプトに詰め込み、全記事生成に使い回した結果、**1リクエストあたり入力トークンが8,000超**になっていたケース。

```python
# NG: 3,000トークンのシステムプロンプトを毎回送る
system_prompt = open("rules_full.txt").read()  # 約12,000字

# OK: Prompt Cachingを有効化（claude-sonnet-4-5以降）
response = client.messages.create(
    model="claude-sonnet-4-5",
    system=[{"type": "text", "text": system_prompt,
             "cache_control": {"type": "ephemeral"}}],
    messages=[{"role": "user", "content": article_prompt}]
)
```

Prompt Cachingを使うと、キャッシュヒット時の入力トークン料金が**約90%オフ**になる。100本/月の生成なら月数百円の差が出る。

---

### 落とし穴② max_tokensを設定せず出力爆発

`max_tokens`未指定のまま「詳しく書いて」と指示した結果、1記事で4,000トークン出力されコストが3倍になった。

```python
# NG
response = client.messages.create(model="...", messages=messages)

# OK: 記事用途なら1,500〜2,000で十分
response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=1800,
    messages=messages
)
```

---

### 落とし穴③ エラー時の無限リトライによる二重課金

ネットワーク瞬断でレスポンスが途切れ、コード側がリトライを無制限に繰り返した。APIはリクエストを受理しており、**両方のトークンが課金済み**だった。

```python
import anthropic
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3),
       wait=wait_exponential(multiplier=1, min=4, max=10))
def generate_article(prompt: str) -> str:
    # 3回まで・指数バックオフ
    return client.messages.create(...).content[0].text
```

リトライは**最大3回・指数バックオフ**を基本とする。

---

### 落とし穴④ 重複記事の大量投稿でアフィリASPアカウント停止

同一テーマの記事を日付だけ変えて量産しA8.netに大量誘導した結果、プログラム利用規約違反でアカウント停止。**アフィリ報酬がゼロになった。**

回避策：

- 記事ごとに異なるキーワードをシードに使う
- 投稿間隔を最低6時間空ける（スパム判定回避）
- A8規約第9条「不正なクリック誘導の禁止」を必ず確認

---

### 落とし穴⑤ プロンプトにアフィリリンクを直接含めてフィルタリング

「この記事にhttps://a8.net/...のリンクを自然に埋め込んで」と指示したところ、Claudeがリンクを拒否するかランダムに書き換えた。

```python
# NG: リンクをプロンプトに含める
# OK: 生成後にプレースホルダを置換する
article = generate_article(prompt)
article = article.replace("{{AFFILIATE_LINK}}", real_affiliate_url)
```

リンクはプロンプト外でポスト処理として挿入する。

---

### 落とし穴⑥ 1分あたりリクエスト数（RPM）上限超過

夜間バッチで100本を並列生成しようとし、`529 Overloaded`が頻発。処理が止まり朝までに0本しか完成しなかった。

```python
import asyncio

async def generate_batch(prompts, rpm_limit=40):
    semaphore = asyncio.Semaphore(rpm_limit // 6)  # 10秒窓
    async with semaphore:
        return await async_generate(prompt)
```

Tier 1では**RPM=50が上限**。セマフォで同時実行数を絞る。

---

### 落とし穴⑦ ログなしでコスト把握ゼロ・月末に請求額ショック

```python
# 必ずusageをログする
usage = response.usage
print(f"in={usage.input_tokens} out={usage.output_tokens} "
      f"cost≈¥{(usage.input_tokens*0.0003+usage.output_tokens*0.0015)/1000*150:.2f}")
```

1記事あたりの実コストを毎回記録し、月次レポートで累積を確認する習慣をつける。筆者実測では**Sonnet利用・1,500字記事で約¥3〜5/本**。

---

### 落とし穴⑧ 本番APIキーをGitHubにプッシュして即失効

`.env`をコミットし、GitHub Secretスキャンにひっかかってキーが自動revoke。APIが全停止した。

```bash
# .gitignore に必ず追加
echo ".env" >> .gitignore
echo "*.key" >> .gitignore
```

環境変数は`python-dotenv`で読み込み、キーはAnthropicダッシュボードで用途別に分けて管理する。

---

### 落とし穴⑨ 生成コンテンツの検証なしで投稿してペナルティ

Claudeが「実際の価格は¥9,800です」と架空の数値を生成し、そのままアフィリ記事として公開。景品表示法上の問題になりうる。

```python
def validate_article(text: str) -> bool:
    # 数字を含む断定表現を検出
    import re
    if re.search(r'¥[\d,]+', text):
        return False  # 人間レビューキューへ
    return True
```

価格・実績・比較データは**必ず人間が確認するキューに回す**。

---

### 落とし穴⑩ claude-opus系をデフォルト使用してコスト10倍

記事量産にOpusを使い続け、Sonnetの**約10倍のコスト**が発生。品質差はほぼなかった。

| モデル | 入力(1M tok) | 出力(1M tok) | 適用場面 |
|---|---|---|---|
| claude-opus-4-8 | $15 | $75 | 企画・設計のみ |
| claude-sonnet-4-6 | $3 | $15 | 記事生成・主力 |
| claude-haiku-4-5 | $0.80 | $4 | タグ付け・要約 |

記事本文はSonnet、メタデータ生成はHaikuと**用途別に使い分ける**のが最適解。
