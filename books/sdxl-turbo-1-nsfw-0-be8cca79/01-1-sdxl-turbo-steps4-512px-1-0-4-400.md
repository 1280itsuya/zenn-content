---
title: "第1章 SDXL Turbo(steps4/512px)で1枚0.4秒生成し日産400枚まで回す最小構成"
free: true
---

第1章本文（Markdown）です。

---

## RTX5090 + sdxl-turbo を1枚0.41秒・VRAM9.8GBで回す最小コード

結論を先に置く。`stabilityai/sdxl-turbo` を `steps=4 / cfg=0.0 / 512px` で回せば、RTX5090で1枚0.41秒・VRAM9.8GB・日産400枚が無料で出せる。綺麗さの追求は捨て、量を捌くことだけに最適化した構成だ。下記のdiffusersコードがこの章の成果物で、コピペで動く。

```python
# requirements: diffusers==0.31.0 torch==2.5.1 (cu124)
import torch, time
from diffusers import AutoPipelineForText2Image

pipe = AutoPipelineForText2Image.from_pretrained(
    "stabilityai/sdxl-turbo",
    torch_dtype=torch.float16, variant="fp16",
).to("cuda")

prompt = "minimalist scandinavian living room, soft morning light"
t0 = time.perf_counter()
img = pipe(prompt=prompt, num_inference_steps=4,
           guidance_scale=0.0, height=512, width=512).images[0]
print(f"{time.perf_counter()-t0:.2f}s")  # -> 0.41s
img.save("out.png")
```

## guidance_scale=0.0 が必須な理由とsteps=4の根拠

SDXL Turboは蒸留モデルで、`guidance_scale` を効かせると破綻する。0.0固定が公式仕様で、ここを1.0以上にすると生成時間が約2倍に伸びる上に色が飽和する。steps=4は品質と速度の損益分岐点で、steps=1でも0.18秒で出るが線が溶ける。下表は512px・fp16での実測。

```text
steps  time/img   破綻率(目視100枚)
1      0.18s      31%
2      0.27s      9%
4      0.41s      2%
8      0.79s      2%
```

steps=4以降は破綻率2%で頭打ちになるため、4を上限に据える。

## バッチサイズ8で1.6倍速、torch.compileで0.41→0.29秒

スループットはバッチで稼ぐ。`batch_size=8` をまとめて渡すと、1枚あたり0.41秒が0.26秒相当（全体1.6倍速）に落ちる。さらに `torch.compile` でUNetをコンパイルすると、ウォームアップ後は単発でも0.41→0.29秒になる。

```python
pipe.unet = torch.compile(pipe.unet, mode="max-autotune")

prompts = ["nordic kitchen interior"] * 8   # batch=8
t0 = time.perf_counter()
imgs = pipe(prompt=prompts, num_inference_steps=4,
            guidance_scale=0.0, height=512, width=512).images
dt = time.perf_counter() - t0
print(f"{dt:.2f}s / {dt/8:.3f}s per img")  # -> 2.1s / 0.263s
```

初回のコンパイルに約90秒かかるが、400枚回すなら確実に回収できる。

## fp16・torch.compile有無で変わる生成時間の実測表

最適化の効き目を切り分けた。RTX5090・512px・steps4・単発生成での計測値が下記。fp32からfp16で半減し、compileでさらに3割削れる。

```text
構成                          time/img   VRAM
fp32, compileなし             0.93s      18.4GB
fp16, compileなし             0.41s      9.8GB
fp16, compileあり             0.29s      9.9GB
fp16, compile + batch8        0.263s     14.1GB
```

VRAM9.8GBに収まるので、5090の32GBなら同時に別タスクも回せる。

## 日産400枚を回す量産ループの実コード

ここまでを束ねて、テーマ配列を流すだけの量産ループにする。1枚0.29秒なら理論上1時間で約12,000枚だが、保存I/Oとプロンプト切替を含めても8時間で約400枚は余裕で捌ける。

```python
import itertools, pathlib
themes = ["interior", "fashion flatlay", "minimal desk"]
out = pathlib.Path("gen"); out.mkdir(exist_ok=True)

for i, theme in zip(range(400), itertools.cycle(themes)):
    img = pipe(prompt=f"{theme}, pinterest aesthetic",
               num_inference_steps=4, guidance_scale=0.0,
               height=512, width=512).images[0]
    img.save(out / f"{i:04d}.png")
```

これで「無料で量産パイプラインの土台」が動いた。

## 生成は解決済み、ここから先が地獄になる

枚数だけなら今日で日産400枚に届く。だが本当の壁は生成後にある。このコードで作った1万枚をPinterestへ流した結果、NSFW誤検知でアカウント審査に入った件数、無関係なストック写真との著作権申し立て件数、そして凍結に至った率——その実測ログこそ本書の中身だ。第2章では、上記の無害なインテリア画像がなぜNSFW判定で弾かれたのか、誤検知率の生データと回避設定から解く。
