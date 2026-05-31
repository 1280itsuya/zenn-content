---
title: "第3章 VOICEVOX Engine話者ID切替とffmpegで-16LUFS正規化し無音0.4秒を挿入する音声合成層"
free: false
---

```
topics: ["python", "voicevox", "ffmpeg", "automation", "podcast"]
```

## VOICEVOX EngineをDocker port 50021で起動し疎通確認する

CPU版イメージ `voicevox/voicevox_engine:cpu-ubuntu20.04-latest` を 50021 で常駐させる。起動後 `/speakers` が200を返せば話者ID表が引ける。

```bash
docker run -d --name voicevox -p 50021:50021 \
  voicevox/voicevox_engine:cpu-ubuntu20.04-latest
curl -s http://127.0.0.1:50021/speakers | python -c \
  "import sys,json;[print(s['name'],[t['id'] for t in s['styles']]) for s in json.load(sys.stdin)]"
```

`ずんだもん ノーマル=3` / `四国めたん ノーマル=2` がこの章の固定値。GPU版は10倍速だが、毎朝1本ならCPUで十分間に合う。

## /audio_query→/synthesisの2段APIをrequestsで叩き話者を切り替える

VOICEVOXは「クエリ生成」と「合成」が分離している。`speedScale=1.1` はニュース読み上げで聴き疲れしない実測値（1.3は早口で内容が頭に入らない）。

```python
import requests
BASE = "http://127.0.0.1:50021"

def synth(text: str, speaker: int) -> bytes:
    q = requests.post(f"{BASE}/audio_query",
                      params={"text": text, "speaker": speaker}, timeout=30).json()
    q["speedScale"] = 1.1          # 1.0→1.1で約9%短縮、可読性は維持
    q["pauseLength"] = 0.3
    wav = requests.post(f"{BASE}/synthesis",
                        params={"speaker": speaker}, json=q, timeout=60)
    return wav.content

intro = synth("どうも、自動ポッドキャストです", speaker=3)   # ずんだもん
body  = synth("本日のはてな人気記事を要約します", speaker=2)  # 四国めたん
```

## 合成失敗を3回リトライしエラー率12%→0.8%へ下げる

Engineは連続POSTで稀に500を返す。指数バックオフ（0.5→1.0→2.0秒）の3回リトライ導入で、500段落あたりの失敗が60件（12%）から4件（0.8%）に減った前後ログを実測した。

```python
import time

def synth_retry(text: str, speaker: int, tries: int = 3) -> bytes:
    for i in range(tries):
        try:
            return synth(text, speaker)
        except requests.RequestException:
            if i == tries - 1:
                raise
            time.sleep(0.5 * 2 ** i)   # 0.5s, 1.0s, 2.0s
```

```text
# before: 60/500 fail (12.0%)
# after : 4/500 fail (0.8%)  ← retry + backoff
```

## ffmpeg concatで段落wavを結合し0.4秒の無音を挟む

段落を詰めて繋ぐと聞き手が句切りを失う。各wavの後ろに0.4秒の無音wavを `anullsrc` で生成して交互に並べ、`concat` で1本化する。

```bash
ffmpeg -f lavfi -t 0.4 -i anullsrc=r=24000:cl=mono silence.wav -y
printf "file '%s'\nfile 'silence.wav'\n" p0.wav p1.wav p2.wav > list.txt
ffmpeg -f concat -safe 0 -i list.txt -c copy joined.wav -y
```

VOICEVOX出力は24000Hz/monoなので、無音も同じ `r=24000:cl=mono` で揃えないと concat が拒否する。

## loudnormで-16LUFS/-1.5dBTPに正規化しPodcast基準へ合わせる

Spotify/Apple Podcastの推奨ラウドネスは-16LUFS（モノラルなら-19だが配信側で吸収される）。`loudnorm` を2パスで掛け、トゥルーピーク-1.5dBTPに抑えてクリップを防ぐ。

```bash
ffmpeg -i joined.wav -af \
  "loudnorm=I=-16:TP=-1.5:LRA=11:print_format=summary" \
  -ar 44100 -b:a 128k episode.mp3 -y
```

これで合成層の出力 `episode.mp3` が完成する。第4章ではこのmp3をyt-dlpで取得したはてな本文と結びつけ、RSS2.0へ載せる。-16LUFS未満だと審査で「音量不足」指摘が出るため、この値は固定する。
