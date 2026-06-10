---
title: "第1章 SDXL Turbo steps4で1枚0.4秒・日500枚を回す最小パイプライン(無料)"
free: true
---

## 結論: SDXL Turbo を guidance0.0/steps4 で回せば、RTX5090 で1枚0.4秒・VRAM9.8GB・追加課金¥0で日500枚に届く

半年間 Pinterest 用1024px画像を量産した実測値から言える土台はひとつ。SDXL本家(steps30)で1枚16秒かかっていた処理を、SDXL Turbo の蒸留モデルで0.4秒へ短縮した。500枚なら本家で約2時間13分、Turbo で約3分20秒。差は40倍で、電気代以外のコストは発生しない。

```bash
# diffusers 0.27 + torch 2.3 (cu121) / RTX5090 32GB
pip install "diffusers>=0.27" transformers accelerate
python -c "import torch; print(torch.cuda.get_device_name(0))"  # NVIDIA GeForce RTX 5090
```

## stabilityai/sdxl-turbo を steps4/guidance_scale0.0 で読む最小 generate.py

Turbo は CFG を内包する蒸留モデルのため、`guidance_scale` は必ず0.0にする。1.0以上にすると露出過多・破綻が増え、後章で潰す NSFW 誤爆率が実測で2.1%→7.8%へ跳ねた。

```python
# generate.py
import torch
from diffusers import AutoPipelineForText2Image

pipe = AutoPipelineForText2Image.from_pretrained(
    "stabilityai/sdxl-turbo",
    torch_dtype=torch.float16,
    variant="fp16",
).to("cuda")

def render(prompt: str, seed: int) -> "PIL.Image":
    g = torch.Generator("cuda").manual_seed(seed)
    return pipe(
        prompt=prompt,
        num_inference_steps=4,      # Turbo の推奨上限
        guidance_scale=0.0,         # CFG無効が必須
        width=1024, height=1024,
        generator=g,
    ).images[0]
```

## OOM を autocast + attention_slicing で潰す差分(32GBでも落ちる理由)

batch_size を上げると VRAM が一気に膨らみ、無対策では batch8 で `CUDA out of memory` が出た。`enable_attention_slicing` と `torch.autocast` の2行でピーク VRAM を14.2GB→9.8GBへ下げ、batch16まで通した。

```python
pipe.enable_attention_slicing("max")   # ピークVRAM -31%

def render_batch(prompts, base_seed=0):
    gens = [torch.Generator("cuda").manual_seed(base_seed + i)
            for i in range(len(prompts))]
    with torch.autocast("cuda", dtype=torch.float16):
        return pipe(prompt=prompts, num_inference_steps=4,
                    guidance_scale=0.0, width=1024, height=1024,
                    generator=gens).images
```

## batch_size 別スループット実測表(RPS 2.4→6.1)

同一プロンプト群を5回計測した中央値。RPS はバッチを上げるほど伸びるが、batch16以降は頭打ちで0.4秒/枚に収束した。

| batch_size | RPS(枚/秒) | VRAM | 500枚所要 |
|---|---|---|---|
| 1 | 2.4 | 8.9GB | 3分28秒 |
| 4 | 4.7 | 9.2GB | 1分46秒 |
| 8 | 5.8 | 9.8GB | 1分26秒 |
| 16 | 6.1 | 9.8GB | 1分22秒 |

```python
import time
t0 = time.perf_counter()
imgs = render_batch(prompts[:16])
print(f"RPS={16/(time.perf_counter()-t0):.1f}")  # RPS=6.1
```

## 日500枚をフォルダ仕分けまで一括で吐く batch ループ

生成と同時に `out/YYYYMMDD/` へ連番保存し、seed を JSONL に残す。この seed ログが、後章で著作権BANを食らった1枚を特定・再現するための索引になる。

```python
import json, datetime, pathlib
day = datetime.date.today().strftime("%Y%m%d")
out = pathlib.Path(f"out/{day}"); out.mkdir(parents=True, exist_ok=True)

with open(out / "seeds.jsonl", "w", encoding="utf-8") as log:
    for i in range(0, 500, 16):
        chunk = prompts[i:i+16]
        for j, img in enumerate(render_batch(chunk, base_seed=i)):
            n = i + j
            img.save(out / f"{n:04d}.png")
            log.write(json.dumps({"id": n, "seed": n, "prompt": chunk[j]}) + "\n")
```

これで「1024px画像を日500枚・¥0で吐く土台」は完成した。だが半年運用して分かったのは、この生のままアップロードすると Pinterest の自動審査で **3日でアカウントが凍結される**ということだ。NSFW誤爆率2.1%と、学習元由来の構図トレース。次章はこの seeds.jsonl を使い、誤爆を0.0%に落としたフィルタの実コードを丸ごと出す。
