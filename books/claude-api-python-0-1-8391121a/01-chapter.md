---
title: "パイプライン全体像と副業収益の仕組みを把握する"
free: true
---

## パイプライン全体像と副業収益の仕組みを把握する

### 全体アーキテクチャ

記事量産パイプラインは大きく3層に分かれる。

```
[キーワード収集] → [Claude APIで記事生成] → [プラットフォーム投稿]
                                                    ↓
                                           [アフィリリンク経由クリック]
                                                    ↓
                                               [収益発生]
```

**入力層**：Googleトレンド・Qiita人気タグ・競合記事タイトルなどからキーワードを収集する。人手でやると1日30分かかる作業だが、スクレイピングで自動化できる。

**生成層**：収集したキーワードをClaude APIに渡し、記事本文・タイトル・タグを一括生成する。ここが本書の核心で、1記事あたりの生成コストは約¥2〜5（claude-sonnet-4-6使用時）。

**投稿層**：生成した記事をZenn・Qiita・はてなブログなど各プラットフォームのAPIへ自動投稿する。投稿先ごとにタグ規則や文字数制限が違うため、ポスター関数を分離して管理するのがポイント。

---

### 収益が発生するまでの流れ

副業収益の仕組みはシンプルだ。記事内に**アフィリエイトリンク**を埋め込み、読者がそこからサービスに登録・購入すると報酬が入る。

```
記事のview数 → クリック率(CTR) → コンバージョン率(CVR) → 収益
例: 1000view × 3%CTR × 2%CVR × 単価¥3,000 = ¥1,800/月
```

重要なのは「view数を稼ぐテーマ選定」と「CVRが高いアフィリ案件を選ぶこと」の両立だ。技術系なら**A8.net**でエンジニア向けサービス（クラウドサービス・学習サービス）の案件を探すと単価が高い。

---

### Pythonコードでパイプラインを1本通す

最小構成のパイプラインは以下の4関数で成立する。

```python
import anthropic
import os

client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

def generate_article(keyword: str) -> dict:
    """キーワードから記事タイトル・本文・タグを生成"""
    prompt = f"""
技術記事を書いてください。
キーワード: {keyword}
出力形式(JSON):
  title: str
  body: str (Markdown, 800字以上)
  tags: list[str] (最大5個)
"""
    resp = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        messages=[{"role": "user", "content": prompt}]
    )
    import json
    return json.loads(resp.content[0].text)

def post_to_qiita(article: dict) -> str:
    """Qiita APIへ投稿し、URLを返す"""
    import requests
    headers = {"Authorization": f"Bearer {os.environ['QIITA_TOKEN']}"}
    payload = {
        "title": article["title"],
        "body": article["body"],
        "tags": [{"name": t} for t in article["tags"]],
        "private": False,
    }
    r = requests.post("https://qiita.com/api/v2/items", json=payload, headers=headers)
    r.raise_for_status()
    return r.json()["url"]

def run_pipeline(keywords: list[str]):
    for kw in keywords:
        article = generate_article(kw)
        url = post_to_qiita(article)
        print(f"投稿完了: {url}")
```

`run_pipeline(["Python 仮想環境 エラー", "AWS Lambda コールドスタート 対策"])` を実行すると、2記事がQiitaに自動投稿される。

---

### 落とし穴：タイトルと本文がズレる

最もよくあるバグが**タイトルと本文の乖離**だ。Claude APIに「JSON形式で出力」と指示しても、タイトルだけ「Pythonエラー対策」なのに本文がAWSの話になっていることがある。

対策は生成後に一致チェックを挟むこと。

```python
def validate_coherence(article: dict, keyword: str) -> bool:
    """タイトルと本文の主題がkeywordと一致するか検証"""
    check = client.messages.create(
        model="claude-haiku-4-5-20251001",  # 安価なモデルで十分
        max_tokens=64,
        messages=[{"role": "user", "content":
            f"以下の記事タイトルと本文冒頭はキーワード「{keyword}」と一致しますか？"
            f"YES/NOだけ答えてください。\nタイトル: {article['title']}\n"
            f"本文冒頭: {article['body'][:200]}"
        }]
    )
    return "YES" in check.content[0].text

# 生成後に必ず通す
if not validate_coherence(article, kw):
    print(f"テーマ不一致のためスキップ: {kw}")
    continue
```

---

### 収益化の現実的な目線

パイプラインを動かしても最初の1ヶ月は収益¥0が普通だ。Zenn・Qiitaは投稿してもview数が付くまでに数週間かかり、アフィリ報酬が口座に入るまでは2〜3ヶ月のタイムラグがある。

**最初にやるべきこと**は量より質：まず10記事を人の目で確認してから自動化を広げる。テンプレっぽいタイトル（「〇〇とは？メリット・デメリット」）はアルゴリズムに嫌われるため、実在するエラーメッセージや具体的なユースケースをキーワードに選ぶほうがview数が伸びやすい。

次章では、このパイプラインに**キーワード自動収集モジュール**を接続し、毎朝7時に完全無人で動く仕組みを実装する。
