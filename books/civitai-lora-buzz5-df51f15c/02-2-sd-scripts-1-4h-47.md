---
title: "第2章: sd-scripts自動化スクリプトで学習1本あたりの工数を4h→47分に短縮"
free: false
---

学習1本あたりの工数が4時間→47分になる理由を先に示す。ボトルネックは「コマンド打ち直し・パラメータ確認・ファイル整理」という工程間の切り替えコストだ。全ステップをDAGとして直列に繋ぎ、1コマンドで完走させるだけで3/4の時間が消える。

## 5ステップをDAGに変換するlora_pipeline.pyの全体像

パイプラインは以下の5フェーズで構成する:

1. **Fetch** — Civitai/Danbooru から画像DL
2. **Tag** — WD14タガーで自動キャプション生成
3. **Config** — sd-scripts用 toml を動的生成
4. **Train** — RunPod or ローカルGPU で学習起動
5. **Save** — チェックポイントをB2/S3に保存

エントリポイントは `python lora_pipeline.py lora.toml` の1行のみ。

```python
# lora_pipeline.py
import subprocess
import sys
import tomllib

def main(config_path: str) -> None:
    with open(config_path, "rb") as f:
        cfg = tomllib.load(f)

    steps = [
        ("fetch",  f"python steps/fetch.py {config_path}"),
        ("tag",    f"python steps/tag.py {config_path}"),
        ("config", f"python steps/gen_config.py {config_path}"),
        ("train",  f"python steps/train.py {config_path}"),
        ("save",   f"python steps/save.py {config_path}"),
    ]

    for name, cmd in steps:
        print(f"[{name}] start")
        subprocess.run(cmd, shell=True, check=True)
        print(f"[{name}] done")

if __name__ == "__main__":
    main(sys.argv[1] if len(sys.argv) > 1 else "lora.toml")
```

## CivitaiとDanbooru自動クロール+WD14タガーで1,000枚を18分処理

Civitai の `GET /api/v1/images` はレート制限1req/secなので `asyncio.Semaphore(1)` で制御する。

```python
# steps/fetch.py
import asyncio, aiohttp, tomllib, sys
from pathlib import Path

async def fetch_civitai(tag: str, limit: int, out_dir: Path) -> int:
    out_dir.mkdir(parents=True, exist_ok=True)
    sem = asyncio.Semaphore(1)
    saved = 0

    async with aiohttp.ClientSession() as session:
        resp = await session.get(
            "https://civitai.com/api/v1/images",
            params={"tags": tag, "limit": min(limit, 200), "sort": "Most Reactions"},
        )
        items = (await resp.json()).get("items", [])

        async def dl(item):
            nonlocal saved
            async with sem:
                dst = out_dir / f"{item['id']}.jpg"
                if dst.exists():
                    return
                r = await session.get(item["url"])
                dst.write_bytes(await r.read())
                saved += 1
                await asyncio.sleep(1.0)

        await asyncio.gather(*[dl(i) for i in items])
    return saved

if __name__ == "__main__":
    with open(sys.argv[1], "rb") as f:
        cfg = tomllib.load(f)
    n = asyncio.run(fetch_civitai(
        cfg["fetch"]["tag"], cfg["fetch"]["limit"], Path(cfg["fetch"]["out_dir"])
    ))
    print(f"Downloaded {n} images")
```

WD14タガーはバッチサイズ32で1,000枚を18分処理（RTX 4090実測）。subprocess経由で叩く。

```bash
# steps/tag.py から呼ぶシェルコマンド
python wd14_tagger.py \
  --model wd-v1-4-convnext-tagger-v2 \
  --batch_size 32 \
  --train_data_dir ./data/raw \
  --caption_extension .txt \
  --thresh 0.35
```

## sd-scripts tomlをJinja2で動的生成：手打ちゼロで設定ミスを排除

```python
# steps/gen_config.py
from jinja2 import Environment
import tomllib, sys

TMPL = """\
[general]
enable_bucket = true
caption_extension = ".txt"
shuffle_caption = true
keep_tokens = 1
resolution = {{ resolution }}

[dataset.{{ tag }}]
image_dir = "{{ image_dir }}"
num_repeats = {{ num_repeats }}
"""

def main(cfg: dict) -> None:
    out = Environment().from_string(TMPL).render(
        resolution=cfg["train"]["resolution"],
        tag=cfg["fetch"]["tag"],
        image_dir=cfg["fetch"]["out_dir"],
        num_repeats=cfg["train"]["num_repeats"],
    )
    with open("dataset.toml", "w") as f:
        f.write(out)
    print("dataset.toml generated")

if __name__ == "__main__":
    with open(sys.argv[1], "rb") as f:
        cfg = tomllib.load(f)
    main(cfg)
```

管理するパラメータは `network_dim`・`network_alpha`・`learning_rate` の3つに絞る。残りはデフォルト固定にすることで後述する失敗の原因特定を容易にした。

```toml
# lora.toml (管理対象パラメータのみ)
[fetch]
tag = "anime_style"
limit = 200
out_dir = "./data/raw"

[train]
backend = "local"          # "local" or "runpod"
runpod_pod_id = ""
resolution = 512
num_repeats = 8
network_dim = 32
network_alpha = 16
learning_rate = "1e-4"
```

## RTX 4090ローカル vs RunPod A100実測比較：¥380 vs ¥1,200の損益分岐点

3ヶ月・42本のLoRAで取ったコストデータ。

| 環境 | 学習時間/本 | 電力・GPU代/本 | 固定費割り* | 合計/本 |
|---|---|---|---|---|
| RTX 4090ローカル | 38分 | ¥25 | ¥355 | **¥380** |
| RunPod A100 80G | 22分 | $8.0 ≈ ¥1,200 | ¥0 | **¥1,200** |

*電気代¥30/kWh・350W・PC購入費を月50本で償却した場合。

損益分岐点は月**17本**。それ以下ではRunPodの方がコスト効率が高い（初期投資ゼロ・スケールアウト容易）。17本を超えた時点でローカルに切り替える判断基準として機能する。

`lora.toml` の `backend` 1行でRunPod/ローカルを切り替えられるようにしてあるので、月途中での切り替えもコマンド変更不要だ。

## 過学習でBuzz数が逆転した失敗3件とパラメータ修正手順

**失敗①：num_repeats=20でBuzz数-40%**

学習画像200枚にnum_repeats=20を設定した結果、実質ステップ数が4,000を超えメインキャラ顔が固定化。CLIP aesthetic scorerが7.2→4.8に下落し、Buzz数は前LoRA比-40%。修正値: `num_repeats=8`。

**失敗②：network_dim=128でA100 80GBがOOM→チェックポイント欠損**

`network_dim=128, network_alpha=64, batch_size=4` の組み合わせでOOM。途中保存のチェックポイントのまま終了し再現不可。network_dim=32との品質差は実測で5%未満だったため、128を使う理由はない。修正値: `network_dim=32, network_alpha=16`。

**失敗③：learning_rate=5e-4でスタイル崩壊（10ステップ目）**

「学習が遅い」と感じてlrを5倍にしたところ、WD14キャプション精度が低い画像が混入していた場合に10ステップで崩壊。先にキャプション品質フィルタを通す手順を追加した。

```python
# steps/filter_captions.py — confidence 0.5未満の行を削除
from pathlib import Path

def filter_low_confidence(caption_dir: str, threshold: float = 0.5) -> int:
    removed = 0
    for txt in Path(caption_dir).glob("*.txt"):
        lines = txt.read_text(encoding="utf-8").splitlines()
        # WD14 出力形式: "tag:0.92, tag2:0.31, ..."
        filtered = [
            l for l in lines
            if ":" not in l or float(l.rsplit(":", 1)[-1].strip()) >= threshold
        ]
        if len(filtered) < len(lines):
            txt.write_text("\n".join(filtered), encoding="utf-8")
            removed += len(lines) - len(filtered)
    return removed

if __name__ == "__main__":
    import sys
    n = filter_low_confidence(sys.argv[1])
    print(f"Removed {n} low-confidence lines")
```

3件に共通する教訓: パラメータ変更は**1変数ずつ**行い、Buzz数の計測には**最低72時間**を確保する。複数変数を同時に変えると原因特定ができなくなり、失敗コストが2倍以上になる。次章ではこの計測を自動化するCLIP aesthetic scorer統合と、Buzz数をKPIとしたA/Bテスト設計に入る。
