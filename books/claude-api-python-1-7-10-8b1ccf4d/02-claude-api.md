---
title: "パイプライン全体設計——Claude APIでテーマ選定から投稿まで繋ぐアーキテクチャ図"
free: false
---

## パイプライン全体設計——Claude APIでテーマ選定から投稿まで繋ぐアーキテクチャ図

コンテンツ自動化の核心は「繋ぎ方」にある。Claude APIで記事を生成できても、それが品質チェックを経ずに投稿され、収益と結びついていなければ、量産は単なるスパムになる。この章では、4ブロックのアーキテクチャを設計し、各ブロックをPythonコードと環境変数でどう接続するかを示す。

---

## 4ブロック構成の全体像

```
[1. 生成]        [2. 品質ゲート]     [3. 自動投稿]     [4. 収益計測]
テーマ選定    →  LLM-as-judge    →  Zenn/Qiita    →  view/click
本文生成      →  スコア閾値       →  はてな/Dev.to →  A8アフィリ
タグ付け      →  タイトル一致検証  →  Gumroad       →  ROIトラッカー
```

各ブロックは **独立したPythonモジュール** として実装し、`.env` の環境変数で有無を切り替える。これにより、Qiitaトークンが失効しても他チャネルへの投稿は止まらない。

---

## ブロック1——生成エンジン

テーマ選定は乱数ではなく **勝ちテーマの再投入** で行う。過去記事のviewトップ5からキーワードを抽出し、派生テーマとして渡す。

```python
# src/generators/base_generator.py
import anthropic, os

client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

def generate_article(topic: str, exemplar: str = "") -> dict:
    prompt = f"""
テーマ: {topic}
参考記事の構成: {exemplar}

## 要件
- 見出し3〜5個、コード例1つ以上
- タイトルと本文のテーマを必ず一致させる
- 1500〜2500字

JSON形式で返す: {{"title": "...", "body": "...", "tags": [...]}}
"""
    msg = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=4096,
        messages=[{"role": "user", "content": prompt}]
    )
    return json.loads(msg.content[0].text)
```

**落とし穴①**: `json.loads` は必ず `try/except` で囲む。Claude APIは稀にJSON前後に説明文を付けることがあり、パースエラーで生成物が全滅する。

---

## ブロック2——品質ゲート

生成した記事をそのまま投稿すると「タイトルはPythonの話なのに本文はJavaScriptになっている」ドリフトが発生する。LLM-as-judgeで **タイトルと本文の一致スコア** を算出し、閾値未満は廃棄する。

```python
# src/quality/judge.py
def judge_article(title: str, body: str, threshold: float = 0.75) -> bool:
    prompt = f"""
タイトル: {title}
本文（先頭300字）: {body[:300]}

タイトルと本文のテーマ一致度を0.0〜1.0で返せ。数値のみ。
"""
    score = float(client.messages.create(
        model="claude-haiku-4-5-20251001",  # 判定はHaikuで十分
        max_tokens=10,
        messages=[{"role": "user", "content": prompt}]
    ).content[0].text.strip())
    return score >= threshold
```

Haikuを使うことでAPI費用を1/10に抑えられる。判定が甘い場合は `threshold` を上げるだけで品質を調整できる。

---

## ブロック3——自動投稿

投稿先は `.env` で制御する。トークン未設定のチャネルは自動スキップされるため、新チャネルの追加はトークンを追記するだけでよい。

```ini
# .env
ZENN_GITHUB_TOKEN=ghp_xxx       # 未設定→Zennスキップ
QIITA_TOKEN=xxx                 # 未設定→Qiitaスキップ
QIITA_MIN_INTERVAL_HOURS=5      # 429対策: 最低5時間間隔
HATENA_API_KEY=xxx
```

```python
# orchestrator/post_dispatcher.py
from dotenv import load_dotenv
load_dotenv()  # ← これを忘れると os.environ が空のまま全スキップ

def dispatch(article: dict):
    if os.environ.get("ZENN_GITHUB_TOKEN"):
        zenn_poster.post(article)
    if os.environ.get("QIITA_TOKEN"):
        qiita_poster.post(article)  # レートゲート内蔵
```

**落とし穴②**: `load_dotenv()` の呼び出し漏れが最も多いバグ。各モジュールではなく **オーケストレーター** で一度だけ呼ぶことを徹底する。

---

## ブロック4——収益計測

投稿して終わりでは改善ループが回らない。view数を日次で取得し、勝ちテーマをブロック1へフィードバックする。

```python
# src/analytics/winner_extractor.py
def extract_top_topics(platform="zenn", top_n=5) -> list[str]:
    """view上位N件のタグ・キーワードを返す"""
    articles = fetch_my_articles(platform)
    sorted_articles = sorted(articles, key=lambda a: a["views"], reverse=True)
    return [extract_keywords(a["title"]) for a in sorted_articles[:top_n]]
```

この関数の出力を翌朝のテーマ生成に渡すことで、「よく読まれるテーマをさらに深掘りする」増幅ループが成立する。

---

## 環境変数設計の原則

| 変数名 | 役割 | 未設定時の挙動 |
|---|---|---|
| `ANTHROPIC_API_KEY` | 生成エンジン必須 | 起動時エラーで即停止 |
| `QIITA_TOKEN` | Qiita投稿 | チャネルスキップ |
| `QUALITY_THRESHOLD` | 品質ゲート閾値 | デフォルト0.75 |
| `DRY_RUN` | 実投稿しない | `1`でデバッグ用 |

`ANTHROPIC_API_KEY` だけは起動時に必須チェックを入れ、他は全てオプションにする。これにより「何も設定しなくても生成だけは動く」状態を保ち、段階的なセットアップが可能になる。

---

## まとめ

4ブロックの責務を分離し、`.env` を制御盤にすることでパイプラインは **チャネルを増やすほど複雑さが増さない** 設計になる。次章ではブロック1の生成エンジンを掘り下げ、テーマ選定アルゴリズムとプロンプト設計の詳細を解説する。
