---
title: "第2章 Zennのslug・キャッシュ・絵文字仕様でOGPが崩れる7箇所と回避コード"
free: false
---

# 第2章 Zennのslug・キャッシュ・絵文字仕様でOGPが崩れる7箇所と回避コード

20記事を貼り直した運用ログから、OGPが崩れる原因は「slug」「キャッシュ」「絵文字frontmatter」の3系統に集約された。以下、実際に旧画像が貼られた7箇所を再現コードで潰す。

## slugのハイフン有無でリンクカードが消える正規表現検証

Zennのslugは `^[a-z0-9-]{12,50}$` を満たさないとリンクカード（og:image付き）が生成されず、`https://zenn.dev/...` がただの青リンクに落ちる。CIで弾く。

```python
import re, sys, glob, frontmatter

SLUG = re.compile(r"^[a-z0-9](?:[a-z0-9-]{10,48})[a-z0-9]$")

for path in glob.glob("articles/*.md"):
    slug = path.split("/")[-1].removesuffix(".md")
    if not SLUG.match(slug):
        sys.exit(f"NG slug: {slug} (12-50字/英小文字数字ハイフン/端ハイフン禁止)")
```

末尾ハイフンや12字未満で200を返すのにカードだけ出ない事故が3記事で起きた。

## published_atとslug変更でOGPが旧画像のまま残るキャッシュ破棄

Zenn・X・Slackは og:image を URL 単位で長期キャッシュする。slugを据え置いて画像を差し替えると最大1週間旧画像が出る。クエリでキャッシュキーを変える。

```python
def og_url(slug: str, rev: str) -> str:
    # rev に git の短縮SHA を入れて画像更新ごとにキーを変える
    return f"https://img.example.dev/og/{slug}.png?v={rev}"
```

`rev` は `git rev-parse --short HEAD` をCIで注入。X側は手動で「カードを再取得」が必要なので、PRコメントに [Card Validator](https://cards-dev.twitter.com/validator) リンクを自動投下した。

## 絵文字frontmatterと自前og:imageの優先順位を固定する

`emoji: 🦁` を残したまま自前画像を指定すると、Zenn内部カードは絵文字、外部クローラは自前画像という二重表示になる。生成側で排他にする。

```python
def build_meta(post) -> dict:
    if post.get("og_image"):
        post.pop("emoji", None)          # 自前画像優先なら絵文字を捨てる
    return post
```

## og:imageの絶対URL必須要件とクローラ差分表

`og:image` は絶対URL必須。相対パスはZennでは通るがDiscord/Slackで無視される。貼る先で読むmetaが違うため設計段階で全部出す。

```html
<meta property="og:image" content="https://img.example.dev/og/abc.png">
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:image" content="https://img.example.dev/og/abc.png">
```

| クローラ | 読むmeta | 絶対URL | 再取得 |
|---|---|---|---|
| X | twitter:image→og:image | 必須 | Validator手動 |
| Slack | og:image | 必須 | 自動(数分) |
| Discord | og:image | 必須 | 即時 |

`twitter:card` 未設定だとXだけ小サムネに落ちる。3metaを必ずセットで吐くのが崩れゼロの条件だ。
