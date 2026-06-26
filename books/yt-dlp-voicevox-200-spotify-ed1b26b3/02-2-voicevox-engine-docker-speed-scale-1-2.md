---
title: "第2章: VOICEVOX Engine Docker起動とspeed_scale 1.2・intonation_scale 0.9が最適解になった実測経緯"
free: false
---

## docker-compose.yml 1ファイルで VOICEVOX Engine 0.21 を3分起動

VOICEVOX は公式の Docker イメージを提供しており、GPU なしの CPU 版で合成コストゼロで動く。

```yaml
# docker-compose.yml
version: "3.9"
services:
  voicevox:
    image: voicevox/voicevox_engine:cpu-ubuntu20.04-0.21.0
    ports:
      - "50021:50021"
    restart: unless-stopped
```

```bash
docker compose up -d
curl -s http://localhost:50021/version
# => "0.21.0"
```

初回イメージ取得は約 2GB・3 分。以降はキャッシュが効くため `up -d` のみで済む。

---

## /audio_query → /synthesis の 2 ステップ POST で最初の WAV を得る

VOICEVOX API の基本シーケンスは 2 本のエンドポイントだ。`/audio_query` でアクセント付きクエリ JSON を生成し、そのオブジェクトのパラメータを上書きしてから `/synthesis` に投げる。クエリ生成時点で `speedScale` を指定しても無視されるため、**上書きはクエリ取得後** に行う点だけ押さえれば詰まらない。

```python
import requests, json

VOICEVOX_URL = "http://localhost:50021"
SPEAKER_ID   = 3  # ずんだもん（ノーマル）

def synthesize_wav(text: str, speed: float = 1.2, intonation: float = 0.9) -> bytes:
    query = requests.post(
        f"{VOICEVOX_URL}/audio_query",
        params={"text": text, "speaker": SPEAKER_ID},
    ).json()
    query["speedScale"]      = speed
    query["intonationScale"] = intonation
    query["pauseLengthScale"] = 1.1  # 句読点間隔をわずかに広げて息継ぎ感を出す

    return requests.post(
        f"{VOICEVOX_URL}/synthesis",
        params={"speaker": SPEAKER_ID},
        data=json.dumps(query),
        headers={"Content-Type": "application/json"},
    ).content
```

---

## 失敗1：句読点なし 3500 文字で segfault → 500 文字チャンク分割を実装

技術記事のスクレイピングテキストをそのまま渡したところ、コード引用が多く句読点密度が低い 3500 文字超の文章で VOICEVOX プロセスが毎回 segfault した（再現率 100%）。公式ドキュメントには上限記載がないが、実測で **500 文字** が安定上限だ。

```python
import re

def split_text(text: str, max_chars: int = 500) -> list[str]:
    chunks, buf = [], ""
    for s in re.split(r"(?<=[。！？\n])", text):
        if len(buf) + len(s) <= max_chars:
            buf += s
        else:
            if buf:
                chunks.append(buf.strip())
            while len(s) > max_chars:
                chunks.append(s[:max_chars])
                s = s[max_chars:]
            buf = s
    if buf:
        chunks.append(buf.strip())
    return [c for c in chunks if c]
```

---

## 失敗2：「yt-dlp」「GitHub」の誤読をユーザー辞書 JSON 登録で解決

デフォルト辞書は技術固有名詞に弱く、「自動化」が「じどうくわ」と旧仮名読みされるケースが発生した。ユーザー辞書 API で個別登録すると確実に上書きできる。

```python
def register_user_dict(surface: str, yomi: str) -> None:
    requests.post(
        "http://localhost:50021/user_dict_word",
        params={
            "surface": surface,
            "pronunciation": yomi,
            "accent_type": 0,
            "word_type": "PROPER_NOUN",
            "priority": 10,  # 最高優先度
        },
    )

CUSTOM_DICT = {
    "自動化":  "じどうか",
    "yt-dlp":  "わいてぃーだいえるぴー",
    "Spotify": "すぽてぃふぁい",
    "GitHub":  "ぎっとはぶ",
}
for surface, yomi in CUSTOM_DICT.items():
    register_user_dict(surface, yomi)
```

登録は Engine 起動中のみ有効。`GET /user_dict` でエクスポートした JSON を保存し、Docker 再起動後に一括再登録するスクリプトを CI に組み込んでおくと手間がない。

---

## 失敗3：intonation_scale 1.0 の棒読みが完聴率 23% を招いた実測比較表

Spotify for Podcasters の週次レポートから抽出したデータ（エピソード 20 本・平均 90 秒）を下表に示す。

| speed_scale | intonation_scale | 平均完聴率 | 体感 |
|:-----------:|:----------------:|:----------:|------|
| 1.0 | 1.0 | 23% | 棒読み・退屈 |
| 1.0 | 1.5 | 41% | 抑揚過多・不自然 |
| 1.2 | 0.9 | **71%** | **自然・情報密度ちょうど** |
| 1.3 | 0.9 | 68% | やや速い |
| 1.2 | 0.8 | 64% | 平坦気味 |

`speed_scale 1.2` は標準の 20% 増しで、技術系リスナーが情報を取りこぼさない速度の上限だった。`intonation_scale 0.9` は VOICEVOX デフォルトの強調感をわずかに抑え、長時間再生への適応を高める値だ。この 2 値の組み合わせが完聴率 23% → 71% の改善を生んだ。

---

## 最終形 voicevox_synth.py 60 行：90 秒・MP3 8MB に落ち着いた根拠

上記 3 つの失敗対応を統合した完成版が以下だ。

```python
#!/usr/bin/env python3
"""voicevox_synth.py — VOICEVOX HTTP APIでテキストをMP3変換"""
import json, re, io, time, sys, requests
from pathlib import Path
from pydub import AudioSegment

VOICEVOX_URL = "http://localhost:50021"
SPEAKER_ID   = 3
SPEED        = 1.2
INTONATION   = 0.9
CHUNK_SIZE   = 500

def split_text(text: str) -> list[str]:
    chunks, buf = [], ""
    for s in re.split(r"(?<=[。！？\n])", text):
        if len(buf) + len(s) <= CHUNK_SIZE:
            buf += s
        else:
            if buf: chunks.append(buf.strip())
            while len(s) > CHUNK_SIZE:
                chunks.append(s[:CHUNK_SIZE]); s = s[CHUNK_SIZE:]
            buf = s
    if buf: chunks.append(buf.strip())
    return [c for c in chunks if c]

def synth_chunk(text: str) -> bytes:
    q = requests.post(
        f"{VOICEVOX_URL}/audio_query",
        params={"text": text, "speaker": SPEAKER_ID},
    ).json()
    q["speedScale"], q["intonationScale"], q["pauseLengthScale"] = SPEED, INTONATION, 1.1
    return requests.post(
        f"{VOICEVOX_URL}/synthesis",
        params={"speaker": SPEAKER_ID},
        data=json.dumps(q),
        headers={"Content-Type": "application/json"},
    ).content

def text_to_mp3(text: str, out_path: Path) -> None:
    combined = AudioSegment.empty()
    for chunk in split_text(text):
        wav = synth_chunk(chunk)
        combined += AudioSegment.from_wav(io.BytesIO(wav))
    combined.export(out_path, format="mp3", bitrate="128k")
    print(f"{out_path}  ({out_path.stat().st_size / 1_048_576:.1f} MB)")

if __name__ == "__main__":
    src = Path(sys.argv[1]).read_text(encoding="utf-8")
    dst = Path(sys.argv[1]).with_suffix(".mp3")
    t0  = time.time()
    text_to_mp3(src, dst)
    print(f"生成時間: {time.time() - t0:.1f}s")
```

```bash
python voicevox_synth.py article.txt
# article.mp3  (7.9 MB)
# 生成時間: 88.3s
```

90 秒・8MB になる根拠は単純だ。技術記事 1 本が平均 1200 文字、`speed_scale 1.2` 適用後の実読時間は約 7 分、128kbps MP3 で 1 分あたり約 1MB のため 7MB 前後に収まる。GitHub Releases の 2GB/リポジトリ制限に対して余裕があり、Spotify のアップロード上限（128MB）にも引っかからない。第3章ではこの MP3 を GitHub Releases に自動プッシュし、Spotify への RSS フィードを構築する実装に進む。
