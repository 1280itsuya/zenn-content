---
title: "第4章 RSS2.0 itunesタグを手書き生成しGitHub Pagesで配信、Spotify審査落ち3回の原因と修正diff"
free: false
---

<!-- topics: ["python", "rss", "spotify", "automation", "githubpages"] -->

## Jinja2でitunes:image/enclosure lengthを含むRSS2.0を手書き生成する

Spotify審査を通すには `<itunes:image>` と `<enclosure length>` を正確に埋める必要がある。`feedgen` は `length` を自動計算しないため、Jinja2で手書きした方が早い。

```python
import os
from jinja2 import Template
from email.utils import formatdate

TPL = Template("""<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0" xmlns:itunes="http://www.itunes.com/dtds/podcast-1.0.dtd">
<channel>
  <title>毎朝はてなPodcast</title>
  <link>https://USER.github.io/podcast/</link>
  <language>ja</language>
  <itunes:category text="Technology"/>
  <itunes:image href="https://USER.github.io/podcast/cover.jpg"/>
  {% for ep in episodes %}
  <item>
    <title>{{ ep.title }}</title>
    <enclosure url="{{ ep.url }}" length="{{ ep.size }}" type="audio/mpeg"/>
    <guid isPermaLink="false">{{ ep.guid }}</guid>
    <pubDate>{{ ep.pub }}</pubDate>
  </item>
  {% endfor %}
</channel></rss>""")

eps = [{"title": "06/01朝のニュース", "url": "https://USER.github.io/podcast/0601.mp3",
        "size": os.path.getsize("0601.mp3"), "guid": "ep-0601",
        "pub": formatdate(localtime=False)}]
open("feed.xml", "w", encoding="utf-8").write(TPL.render(episodes=eps))
```

## GitHub Pagesにmp3とfeed.xmlを公開する3コマンド

`docs/` をPages公開ディレクトリに指定すると、mp3もfeed.xmlも同一HTTPSドメインで配信できる。100MB制限に注意（1エピソード40MB前後に収める）。

```bash
mkdir -p docs && cp 0601.mp3 feed.xml cover.jpg docs/
git add docs && git commit -m "ep 0601"
git push origin main   # Settings>Pages>Source=docs/ を事前設定
```

## Spotify審査落ち1回目: enclosure length不一致の修正diff

`length` をバイト数でなく秒数で書いていたのが原因。エラーは `Episode audio could not be processed`。`os.path.getsize` の戻り値（バイト）に修正する。

```diff
- <enclosure url="...0601.mp3" length="412" type="audio/mpeg"/>
+ <enclosure url="...0601.mp3" length="38502912" type="audio/mpeg"/>
```

## 審査落ち2回目: カバー画像1400px未満を3000pxへ

Spotifyは最小1400×1400px、推奨3000×3000px。`Image too small` で弾かれる。Pillowで強制リサイズしてから再公開する。

```python
from PIL import Image
im = Image.open("cover_src.png").convert("RGB")
im.resize((3000, 3000)).save("cover.jpg", quality=90)  # 1400未満は即落選
```

## 審査落ち3回目: HTTPSリダイレクトとRSSバリデータ検証

`http://` のguid内リンクで301が発生し `Invalid feed` 判定。全URLを `https://` 直書きに統一する。公開前に下記で構文を検証する。

```bash
curl -s https://USER.github.io/podcast/feed.xml \
  | xmllint --noout -   # エラー0行ならパス
# itunesタグ検証は podcastindex のバリデータにURL貼付
```

修正後の反映は即時ではなく、Spotify for Podcastersへの初回登録から実測で最長6時間かかった。RSS更新自体は5分で取り込まれるが、初回審査だけは半日見ておくと安全に運用できる。
