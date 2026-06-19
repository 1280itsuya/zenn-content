---
title: "SNS運用パイプライン：Claude APIでXノイズをフィルタしZenn/Qiitaへ自動投稿する"
free: false
---

## SNS運用パイプライン：Claude APIでXノイズをフィルタしZenn/Qiitaへ自動投稿する

技術発信の最大の敵は「書く時間よりSNSに溶かす時間のほうが長い」問題だ。本章では三つの自動化を一本のGitHub Actionsワークフローで束ねる。①Xのノイズソースを自動ミュート、②Claude APIで記事メタデータを生成、③Zennへ非対話でgit push—これだけで週3時間以上が返ってくる。

---

## Step 1：Xノイズをミュートリストに自動追加する

Twitter API v2の`/2/users/:id/muting`を使う。ミュート判定はClaude APIに任せる。

```python
import anthropic, tweepy, json

client = anthropic.Anthropic()
tw = tweepy.Client(bearer_token=BEARER, access_token=AT, access_token_secret=ATS,
                   consumer_key=CK, consumer_secret=CS)

def should_mute(tweet_text: str) -> bool:
    resp = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=10,
        messages=[{"role": "user", "content":
            f"次のツイートは技術情報として価値がないノイズか？ yes/no のみ答えよ。\n\n{tweet_text}"}]
    )
    return resp.content[0].text.strip().lower() == "yes"

mentions = tw.get_home_timeline(max_results=100)
for tweet in mentions.data or []:
    if should_mute(tweet.text):
        tw.mute(tweet.author_id)
        print(f"muted: {tweet.author_id}")
```

**落とし穴①**：`claude-haiku-4-5-20251001`を使う。`sonnet`では1,000ツイート処理で約¥200かかるがhaikuなら1/10以下。判定精度より速度とコストを優先する場面だ。

---

## Step 2：Claude APIで記事メタデータを一括生成する

Zenn/Qiita投稿に必要なfrontmatter（タイトル・emoji・tags・slug）をClaude APIで自動生成する。

```python
def generate_metadata(body: str) -> dict:
    resp = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=256,
        messages=[{"role": "user", "content": f"""
以下の記事本文からZenn投稿用メタデータをJSONで返せ。
キー: title, emoji(1文字), topics(配列・5個以内・Zenn正規タグのみ), slug(kebab-case)

本文:
{body[:2000]}
"""}]
    )
    raw = resp.content[0].text
    return json.loads(raw[raw.index("{"):raw.rindex("}")+1])
```

**落とし穴②**：Zennの`topics`は`["javascript","python","ai"]`のような**公式タグ**のみ有効。LLMが造語タグ（`"claude-api"`など）を返すと記事が検索に乗らない。生成後に`VALID_TAGS`セットと差分チェックを必ず挟むこと。

---

## Step 3：GitHub ActionsでZenn git pushを非対話化する

最大の落とし穴はここだ。ローカルでは通るが、CI上で`git push`すると**GitHub認証ダイアログがTTY待ちしてジョブが2時間後にタイムアウト**する。

解決策はPATをURLに直接埋め込み、`GIT_TERMINAL_PROMPT=0`で非対話を強制することだ。

```yaml
# .github/workflows/zenn_publish.yml
name: Zenn Auto Publish
on:
  schedule:
    - cron: "0 22 * * *"   # 毎日07:00 JST
  workflow_dispatch:

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Generate article
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: python src/generate_article.py

      - name: Push to Zenn repo
        env:
          GIT_TERMINAL_PROMPT: "0"
          PAT: ${{ secrets.ZENN_PAT }}
        run: |
          git config user.email "bot@example.com"
          git config user.name "zenn-bot"
          git remote set-url origin https://x-access-token:${PAT}@github.com/yourname/zenn-content.git
          git add articles/
          git diff --cached --quiet || git commit -m "auto: $(date '+%Y-%m-%d')"
          git push origin main
```

**落とし穴③**：`git diff --cached --quiet || git commit`で**差分がない場合のpushをスキップ**している。これがないと「nothing to commit」でActions全体がエラー終了し、後続のQiita投稿ステップも巻き込まれる。

---

## 三つをつなぐ実行順序

```
07:00 JST  ┌─ mute_noisy_accounts.py  （Xノイズ除去）
           ├─ generate_article.py      （Claude APIでメタデータ＋本文生成）
           ├─ zenn_push.yml            （git push → Zenn公開）
           └─ qiita_post.py            （Qiita API POST、5時間インターバル必須）
```

Qiitaは429レート制限が厳しく、**最低5時間のインターバル**なしに連投するとBANに近い制限がかかる。`QIITA_MIN_INTERVAL_HOURS`環境変数でガードを実装し、前回投稿時刻をファイルに記録して比較すること。

この三段構成を動かせば、SNSに引きずられることなく技術記事が毎朝自動で世に出る。
