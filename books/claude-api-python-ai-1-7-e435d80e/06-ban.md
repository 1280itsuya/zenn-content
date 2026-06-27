---
title: "落とし穴集──アカウントBAN・著作権・テンプレスパム判定を回避する実践知"
free: false
---

## 落とし穴集──アカウントBAN・著作権・テンプレスパム判定を回避する実践知

自動投稿パイプラインを動かすと、遅かれ早かれ「無言停止」に遭遇する。以下は実際に踏んだ地雷とその回避策だ。

---

## Qiita 403 停止──レートリミットの罠

Qiita APIはIPとトークン単位でレート制限を持つ。1日に複数記事を連続POSTすると、無言で403が返り始める。実際に3日間 403 を受け続け、復旧にはトークン再発行と数日の投稿休止が必要だった。

```python
# 悪い例：間隔なしで連続POST
for article in articles:
    post_to_qiita(article)

# 良い例：最低5時間インターバルをゲートとして設ける
import time

QIITA_MIN_INTERVAL_HOURS = int(os.getenv("QIITA_MIN_INTERVAL_HOURS", "5"))

def post_with_gate(article, last_posted_at: float) -> bool:
    elapsed = time.time() - last_posted_at
    if elapsed < QIITA_MIN_INTERVAL_HOURS * 3600:
        return False  # スキップして次のサイクルへ
    return post_to_qiita(article)
```

**原則は1日1本。**インターバルを環境変数で調整できる設計にしておくと、プラットフォームの動向に合わせて即対応できる。

---

## X（旧Twitter）凍結──同一文体の連投が引き金

同一文体・同一ハッシュタグを短時間に連投すると、スパムフィルタが反応する。2026年6月に実際にアカウントが凍結され、X投稿を全停止した。

```python
import random

HASHTAG_POOL = ["#AI副業", "#Python自動化", "#副業ブログ", "#LLM活用", "#Claude"]

def build_post(summary: str) -> str:
    tags = random.sample(HASHTAG_POOL, 2)  # 毎回ランダムに2つ選択
    return f"{summary}\n{' '.join(tags)}"
```

凍結後の教訓は「プラットフォームが一つ死んでも詰まらない設計」だ。X・Bluesky・Zenn・Qiitaを並列配線し、1チャネルが落ちたら残りが継続するようにする。X停止は環境変数1つで切れる構造にしておく。

```python
# .env
X_POSTING_DISABLED=1  # 凍結時はこれだけで全X投稿を止める
```

---

## テンプレスパム判定──タイトルと本文のドリフト

量産したのにビュー数がほぼゼロ、という状況の真因の一つが「タイトルと本文の乖離」だ。LLMにテンプレを渡すと、タイトルで約束したテーマが本文で消える「ドリフト」が頻発する。プラットフォームのアルゴリズムはタグと内容の一致度を見ており、テンプレ構造が透けるとサーチにも弾かれる。

```python
def validate_article_consistency(title: str, body: str) -> None:
    """タイトルのキーワードが本文に含まれるか検証。不一致なら投稿中止。"""
    keywords = [w for w in title.split() if len(w) > 2]
    missing = [kw for kw in keywords if kw not in body]
    if missing:
        raise ValueError(f"タイトルキーワード {missing} が本文に未登場。投稿中止。")
```

エージェントへのプロンプトにも「タイトルのテーマを冒頭・中盤・結論で明示すること」と指示を固定する。これだけで、Qiitaのビュー数は平均的に3倍以上改善した。

---

## 著作権──ニュース要約の残留リスク

「最新ニュースを要約して記事にして」と指示すると、元記事の構成や文体が残留しやすい。これは著作権侵害リスクに直結する。

安全な生成パターンは次の3ステップだ。

1. ニュースは「トピック抽出」のみに使い、本文はゼロから書かせる
2. プロンプトに「元記事の文章を引用・翻案しないこと」と明記する
3. 生成後に類似度チェックを通す

```python
import difflib

def similarity_check(source: str, generated: str, threshold: float = 0.35) -> None:
    ratio = difflib.SequenceMatcher(None, source, generated).ratio()
    if ratio > threshold:
        raise ValueError(f"類似度 {ratio:.2f} が閾値超過。再生成してください。")
```

---

## PAT平文漏洩──GitHubに絶対にPushしない

Zenn連携でGitHub PATをコードに直書きしてpushすると、GitHubのSecretスキャンが即座に検知してPATを無効化する。実際にこれを踏み、パイプラインが止まった。

```bash
# .gitignore に必ず追加
.env
.env.local
*.secret
```

`.env`を`.gitignore`に追加し忘れたまま`git add .`すると、PATが公開リポジトリに残る。発覚後はトークンを即座にrevokeし再発行すること。CI/CDで使う場合はGitHub Actions Secretsを経由し、コードには一切書かない。

---

## リスク早見表

| リスク | 検知サイン | 対策 |
|---|---|---|
| Qiita 403 | レスポンスコード監視 | 5h以上インターバル |
| X凍結 | API認証エラー | 多チャネル冗長化 |
| テンプレスパム | ビュー数ゼロ継続 | タイトル-本文一致検証 |
| 著作権 | 類似度スコア超過 | ニュースはトピック抽出のみ |
| PAT漏洩 | GitHubアラートメール | `.env` を `.gitignore` 必須 |

量産自動化の落とし穴は「設計の欠陥」より「運用の見落とし」から来るものが多い。各リスクを検知するコードを最初から組み込んでおくことで、無人運用でも事故を最小限に抑えられる。
