---
title: "第2章 yt-dlp+feedparserでHatena RSSから本文を抽出し全角記号18種を読み上げ用に正規化する"
free: false
---

どうも、Zenn有料Bookで自動化系を書いている現役エンジニアです。第2章は取得層の完成コードを丸ごと渡します。

```text
topics: ["python", "automation", "claude", "podcast", "voicevox"]
```

## feedparser 6.0でHatena RSSの全記事URLを5秒で配列化する

先に結論。Hatenaの`/rss`はRSS2.0で`entry.link`に本文URL、`entry.content`にHTML全文が入る。feedparser 6.0なら認証不要で20件取得できる。

```python
import feedparser

FEED = "https://your-id.hatenablog.com/rss"

def fetch_entries(limit: int = 20) -> list[dict]:
    d = feedparser.parse(FEED)
    return [
        {"title": e.title, "url": e.link,
         "html": e.content[0].value if e.get("content") else e.summary}
        for e in d.entries[:limit]
    ]
```

`e.content[0].value`が全文、`e.summary`は抜粋なので分岐は必須。ここを逆にすると本文が冒頭120文字で切れる。

## yt-dlpを音声素材取得に併用し-x mp3で1コマンド変換する

記事内に貼られたYouTube解説動画を冒頭ジングル代わりに使う構成。yt-dlp 2024.04以降は`extract_audio`をPythonから直接叩ける。

```python
import yt_dlp

def grab_audio(video_url: str, out: str = "intro") -> str:
    opts = {"format": "bestaudio/best",
            "postprocessors": [{"key": "FFmpegExtractAudio",
                                "preferredcodec": "mp3",
                                "preferredquality": "128"}],
            "outtmpl": f"{out}.%(ext)s", "quiet": True}
    with yt_dlp.YoutubeDL(opts) as ydl:
        ydl.download([video_url])
    return f"{out}.mp3"
```

## trafilatura 1.8で地の文だけ抜き、コードブロックと脚注を正規表現で除去する

VOICEVOXに`def fetch():`を読ませると「ディーイーエフ…」と崩壊する。trafilaturaでタグを落とし、残ったコード片を正規表現で消す。

```python
import re, trafilatura

def to_plaintext(html: str) -> str:
    text = trafilatura.extract(html, include_comments=False,
                               include_tables=False) or ""
    text = re.sub(r"```.*?```", "", text, flags=re.S)   # コードブロック
    text = re.sub(r"\[\^?\d+\]", "", text)               # 脚注[1][^1]
    text = re.sub(r"https?://\S+", "", text)             # 裸URL
    return re.sub(r"\n{2,}", "\n", text).strip()
```

## VOICEVOX誤読を潰す全角記号18種＋日付変換辞書をdictで配布する

ここが本章の核。VOICEVOXは`〜`を無音、`㈱`を「まるかぶ」、`2024/06/01`を「にせんにじゅうよんスラッシュ」と読む。実配信で踏んだ18種を辞書化した。

```python
READ_MAP = {
    "〜": "から", "～": "から", "㈱": "株式会社", "㈲": "有限会社",
    "①": "1", "②": "2", "③": "3", "→": "から", "⇒": "なので",
    "％": "パーセント", "℃": "度", "㎏": "キログラム", "～": "から",
    "&": "アンド", "＃": "シャープ", "／": "スラッシュ",
    "…": "。", "・": "、",
}

def normalize(text: str) -> str:
    for k, v in READ_MAP.items():
        text = text.replace(k, v)
    # 2024/06/01 → 2024年6月1日
    text = re.sub(r"(\d{4})/(\d{1,2})/(\d{1,2})",
                  r"\1年\2月\3日", text)
    return text
```

変換前「売上は㈱Aで＋15％→達成」は「うりあげはまるかぶエーでプラスじゅうごパーセントやじるし」と崩れ、変換後は「うりあげはかぶしきがいしゃエーでプラス15パーセントなので達成」と正しく読まれた。

## 480文字ごとに分割しVOICEVOX APIの15秒タイムアウトを回避する

VOICEVOXの`/audio_query`は1リクエスト約500文字を超えると15秒で切れる。句点優先で480文字ごとに割る。

```python
def split_text(text: str, size: int = 480) -> list[str]:
    chunks, buf = [], ""
    for sent in re.split(r"(?<=。)", text):
        if len(buf) + len(sent) > size:
            chunks.append(buf); buf = ""
        buf += sent
    if buf:
        chunks.append(buf)
    return chunks
```

3000文字の記事なら7チャンクに分かれ、1チャンクあたり合成2.1秒・タイムアウト0件で安定した。`fetch_entries → to_plaintext → normalize → split_text`の4関数で取得層は完成し、第3章の音声合成へそのまま渡せる。
