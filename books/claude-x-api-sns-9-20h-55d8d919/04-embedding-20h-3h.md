---
title: "embeddingで重複ニュースを束ね収集を月20h→3hへ"
free: false
---

## embeddingの単価: ローカルe5で¥0、OpenAI text-embedding-3-smallで1万件¥48

intfloat/multilingual-e5-largeをローカルGPUで回せば追加コストは電気代のみ。クラウドに出すならOpenAI `text-embedding-3-small`が1万件で約¥48、`-3-large`で約¥312。日次500件なら月15万件、ローカル¥0に対しsmallで月¥720。

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("intfloat/multilingual-e5-large")

def embed(titles: list[str]):
    # e5は "query: " 接頭辞が必須。付け忘れると類似度が0.1ほど下振れする
    texts = [f"query: {t}" for t in titles]
    return model.encode(texts, normalize_embeddings=True)
```

## コサイン類似度0.86でニュースを束ねる

normalize済みベクトルなら内積がそのままコサイン類似度。`scipy`の階層クラスタリングで距離 `1 - sim` を使い、しきい値0.14（=類似度0.86）で切る。

```python
import numpy as np
from scipy.cluster.hierarchy import linkage, fcluster

def cluster(vecs: np.ndarray, sim_threshold=0.86):
    dist = 1.0 - (vecs @ vecs.T)
    np.fill_diagonal(dist, 0.0)
    condensed = dist[np.triu_indices(len(vecs), k=1)]
    Z = linkage(condensed, method="average")
    return fcluster(Z, t=1.0 - sim_threshold, criterion="distance")
```

## しきい値0.82 / 0.86 / 0.90の取りこぼし実測

3ヶ月の収集ログ4,812件を手ラベルと突き合わせた結果。0.82は別話題まで束ねる過剰マージが14%、0.90は同一ニュースを割る取りこぼしが11%。0.86が両者の交点だった。

```text
threshold  over_merge  miss_split  clusters
0.82       14.2%       1.1%        612
0.86        3.8%       3.0%        974   ← 採用
0.90        0.9%      11.4%       1,403
```

## クラスタ代表をClaudeで3行要約しダイジェスト化

各クラスタは最古の投稿を代表に選び、本文を結合してClaude `claude-haiku-4-5`へ。1クラスタ約700トークン、日次40クラスタで月の要約コストは約¥260。

```python
import anthropic
client = anthropic.Anthropic()

def summarize(cluster_texts: list[str]) -> str:
    joined = "\n---\n".join(cluster_texts[:5])
    msg = client.messages.create(
        model="claude-haiku-4-5",
        max_tokens=180,
        messages=[{"role": "user",
            "content": f"次の重複ニュース群を事実のみ3行で要約:\n{joined}"}],
    )
    return msg.content[0].text
```

## 月20h→3hの内訳と、取りこぼし率2%という限界

RescueTimeで計測した収集時間は、手作業20h（各SNS巡回13h＋重複の読み直し7h）から、ダイジェスト確認3hへ。差分17hはembeddingが重複読みを消した分。

```bash
# 毎朝6:40に収集→クラスタ→要約→1通配信
40 6 * * * cd /opt/news && .venv/bin/python digest.py >> logs/digest.log 2>&1
```

限界も数字で残す。0.86では速報の言い換え記事が別クラスタへ散り、重要ニュースの**2.0%（月およそ19件）を1通に載せ損ねた**。完全自動の代償として許容し、週次でmiss_splitログだけ目視する運用に落とした。
