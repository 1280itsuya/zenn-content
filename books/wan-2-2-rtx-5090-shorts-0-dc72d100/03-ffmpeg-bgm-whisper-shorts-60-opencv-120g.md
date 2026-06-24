---
title: "ffmpegでBGM+Whisper字幕入りShorts 60秒に自動編集 — OpenCVサムネ自動生成と120GB/月のディスク管理"
free: false
---

## Pixabay / YouTube Audio Library から BGM を取得して ffmpeg でループ合成する

著作権フリー音源の調達先は **Pixabay**（API不要・CDN URLが固定で `wget` が通る）と **YouTube Audio Library**（ブラウザログイン必須だが品質が高い）の2択に絞る。Pixabay の場合、CDN URL は `https://cdn.pixabay.com/audio/` 配下に集まっており、曲ページのソースから `contentUrl` を抽出できる。

取得とループ合成を1コマンドで完結させる:

```bash
# Pixabay から BGM 取得（headless wget でダウンロード可能）
wget -q "https://cdn.pixabay.com/audio/2024/05/lo-fi-chill.mp3" -O bgm_src.mp3

# 動画尺に合わせて BGM を無限ループ → 音量30%でミックス → 60秒で強制カット
ffmpeg -y \
  -stream_loop -1 -i bgm_src.mp3 \
  -i input_video.mp4 \
  -filter_complex "[0:a]volume=0.3[bgm];[bgm]apad[a]" \
  -map 1:v -map "[a]" \
  -t 60 -shortest \
  -c:v copy -c:a aac -b:a 128k \
  output_with_bgm.mp4
```

`-stream_loop -1` で BGM を無限ループさせ、`-shortest` で動画長に合わせて打ち切る。`-t 60` は YouTube Shorts 上限 60 秒の強制カットで、これを省くと Shorts 扱いされずロングフォームに落ちる。

## faster-whisper で日本語 SRT を生成して ffmpeg で字幕を焼き込む

```python
from faster_whisper import WhisperModel
import subprocess

def transcribe_to_srt(video_path: str, srt_path: str) -> None:
    # RTX 5090 + CUDA 12.x 環境では float16 固定で VRAM 3.2GB に収まる
    model = WhisperModel("medium", device="cuda", compute_type="float16")
    segments, _ = model.transcribe(video_path, language="ja", beam_size=5)

    with open(srt_path, "w", encoding="utf-8") as f:
        for i, seg in enumerate(segments, 1):
            start = _fmt_time(seg.start)
            end   = _fmt_time(seg.end)
            f.write(f"{i}\n{start} --> {end}\n{seg.text.strip()}\n\n")

def _fmt_time(sec: float) -> str:
    h, m, s = int(sec // 3600), int((sec % 3600) // 60), sec % 60
    return f"{h:02d}:{m:02d}:{s:06.3f}".replace(".", ",")

def burn_subtitles(video: str, srt: str, out: str) -> None:
    subprocess.run([
        "ffmpeg", "-y", "-i", video,
        "-vf", (
            f"subtitles={srt}:force_style="
            "'FontSize=22,PrimaryColour=&H00FFFFFF,"
            "OutlineColour=&H00000000,BorderStyle=3'"
        ),
        "-c:a", "copy", out
    ], check=True)
```

RTX 5090 での実測では `WhisperModel("medium")` が 60 秒動画を **約 7 秒**（実時間比 0.12 倍速）で書き起こす。`force_style` で白文字+黒縁を固定し、背景を問わず視認できるスタイルにする。

## OpenCV + Pillow で A/B テスト用サムネを 2 枚自動生成する

```python
import cv2
from PIL import Image, ImageDraw, ImageFont

def generate_thumbnails(video_path: str, title: str, out_dir: str) -> list[str]:
    cap = cv2.VideoCapture(video_path)
    total = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))
    results = []

    for variant, frame_idx in [("A", total // 4), ("B", total // 2)]:
        cap.set(cv2.CAP_PROP_POS_FRAMES, frame_idx)
        ret, frame = cap.read()
        if not ret:
            continue

        img = Image.fromarray(cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)).resize((1280, 720))

        # 下半分にグラデーションオーバーレイ
        overlay = Image.new("RGBA", img.size, (0, 0, 0, 0))
        draw = ImageDraw.Draw(overlay)
        for y in range(360, 720):
            alpha = int((y - 360) / 360 * 160)
            draw.line([(0, y), (1280, y)], fill=(0, 0, 0, alpha))
        img = Image.alpha_composite(img.convert("RGBA"), overlay).convert("RGB")

        font = ImageFont.truetype("/usr/share/fonts/noto/NotoSansJP-Bold.ttf", 64)
        ImageDraw.Draw(img).text((40, 580), title[:20], font=font, fill=(255, 255, 255))

        out_path = f"{out_dir}/thumb_{variant}.jpg"
        img.save(out_path, quality=95)
        results.append(out_path)

    cap.release()
    return results
```

フレーム取得位置を 1/4 点と 1/2 点に分けることで、シーンの異なるサムネを 1 ループで 2 枚生成する。YouTube Studio の A/B テストに直接アップロードできる形式で出力する。

## 月 100 本生成で発覚した 120GB 消費 —— pathlib 自動削除で 5GB に圧縮

実運用での素材ファイル内訳:

| 種別 | 1 本あたり | 月 100 本 |
|------|-----------|----------|
| Wan 2.2 生成動画（元素材）| 800MB〜1.2GB | **80〜120GB** |
| BGM 合成中間 MP4 | 200MB | 20GB |
| 完成 Shorts | 50MB | 5GB |

アップロード確認後に元素材と中間ファイルを即時削除する:

```python
import pathlib, json

def cleanup_after_upload(job_dir: str, upload_id: str) -> dict:
    """YouTube insert() 成功後に呼ぶ。完成品以外を全削除する"""
    base = pathlib.Path(job_dir)
    removed = []

    for pattern in ["*.mp4", "*.wav", "*.png"]:
        for f in base.glob(pattern):
            if upload_id not in f.stem:   # 完成品ファイル名には upload_id を含める命名規則
                size_mb = f.stat().st_size / 1_048_576
                f.unlink()
                removed.append({"file": f.name, "mb": round(size_mb, 1)})

    log = base / "cleanup_log.ndjson"
    with log.open("a") as fp:
        json.dump({"upload_id": upload_id, "removed": removed}, fp, ensure_ascii=False)
        fp.write("\n")

    return {"freed_mb": round(sum(r["mb"] for r in removed), 1), "count": len(removed)}
```

1 本あたり約 1GB を即時解放し、月の純消費を **5GB** まで圧縮できる。

## 全処理を pipeline.py 1 ファイルに統合して requirements.txt で固定する

```python
# pipeline.py ── ComfyUI 後段に接続するエントリポイント
import argparse, pathlib, subprocess
from bgm import mix_bgm
from subtitle import transcribe_to_srt, burn_subtitles
from thumbnail import generate_thumbnails

def run(video: str, title: str, out_dir: str) -> str:
    d = pathlib.Path(out_dir)
    d.mkdir(parents=True, exist_ok=True)

    bgm_out = str(d / "with_bgm.mp4")
    srt_out = str(d / "subtitle.srt")
    sub_out = str(d / "with_sub.mp4")
    final   = str(d / "final.mp4")

    mix_bgm(video, bgm_out)
    transcribe_to_srt(bgm_out, srt_out)
    burn_subtitles(bgm_out, srt_out, sub_out)
    generate_thumbnails(sub_out, title, out_dir)

    # YouTube 推奨: H.264 / 縦 9:16 / AAC 128k
    subprocess.run([
        "ffmpeg", "-y", "-i", sub_out,
        "-vf", "scale=1080:1920:force_original_aspect_ratio=decrease,pad=1080:1920:-1:-1",
        "-c:v", "libx264", "-preset", "fast", "-crf", "23",
        "-c:a", "aac", "-b:a", "128k",
        final
    ], check=True)
    return final

if __name__ == "__main__":
    ap = argparse.ArgumentParser()
    ap.add_argument("video")
    ap.add_argument("title")
    ap.add_argument("--out", default="out")
    args = ap.parse_args()
    print(run(args.video, args.title, args.out))
```

```text
# requirements.txt（バージョン固定で再現性を保証）
faster-whisper==1.1.0
opencv-python-headless==4.10.0.84
Pillow==10.4.0
```

`opencv-python-headless` はサーバー環境で X11 依存を排除するために必須。`faster-whisper==1.1.0` は CUDA 12.x + float16 での動作が確認済みのバージョンに固定する（1.2.x 以降は `WhisperModel` の API に破壊的変更がある）。

`pipeline.py video.mp4 "タイトル20文字以内" --out ./job_001` の 1 コマンドで BGM 合成→字幕焼き込み→サムネ 2 枚→縦 9:16 最終エンコードが完結する。次章では `youtube_upload.py` で YouTube Data API v3 に自動投稿し、`cleanup_after_upload` を成功コールバックに組み込む。
