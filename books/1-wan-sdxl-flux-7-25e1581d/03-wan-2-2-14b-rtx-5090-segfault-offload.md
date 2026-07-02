---
title: "Wan 2.2-14B ローカル実装：RTX 5090 segfault 回避と offload 必須設定の全手順"
free: false
---

## RTX 5090/32GB で offload 省略すると生成1ステップ目で SIGSEGV

環境は RTX 5090（32GB VRAM）、Python 3.11、diffusers 0.28.2、PyTorch 2.4.0+cu124。

```python
# ❌ これで SEGFAULT（1ステップ目でプロセスが kill される）
from diffusers import WanPipeline
import torch

pipe = WanPipeline.from_pretrained(
    "Wan-AI/Wan2.2-T2V-A14B-Diffusers",
    torch_dtype=torch.bfloat16,
)
pipe.to("cuda")  # ← VRAM 28.3GB 超過、アテンション計算中に OS レベル kill

video = pipe("a cat running", num_inference_steps=20).frames[0]
```

`pipe.to("cuda")` 時点で VRAM は 28.3GB に達し、残り 3.7GB でアテンション計算を走らせようとして SIGSEGV になる。`torch.cuda.OutOfMemoryError` ではなくプロセスごと落ちるため traceback が出ない。ここで診断に 2 日間費やした。

---

## enable_model_cpu_offload() は pipe.to("cuda") の前に置く

```python
# ✅ 正しい順序
from diffusers import WanPipeline
import torch

pipe = WanPipeline.from_pretrained(
    "Wan-AI/Wan2.2-T2V-A14B-Diffusers",
    torch_dtype=torch.bfloat16,
)
pipe.enable_model_cpu_offload()  # ← まずここ。to("cuda") は呼ばない

video = pipe(
    "a cat running",
    num_inference_steps=20,
    height=512,
    width=512,
).frames[0]
```

`enable_model_cpu_offload()` を先に呼ぶと、diffusers の hook が各サブモジュールの forward 前後に `to("cuda")` / `to("cpu")` を自動挿入する。ピーク VRAM は 22.1GB まで落ちる（offload なし比 −6.2GB）。

---

## VAE dtype を bfloat16 から float32 に固定しないと NaN フレームが散発する

```python
pipe.enable_model_cpu_offload()

# ← この位置。offload hook が dtype を上書きするため、前に置いても無効
pipe.vae = pipe.vae.to(dtype=torch.float32)
```

`bfloat16` のまま VAE デコードを走らせると一部フレームのピクセル値が `nan` になり真っ黒フレームが混入する（発生率 0.6%）。この1行だけで 0.6% の失敗が消える。挿入タイミングを `enable_model_cpu_offload()` の前にすると offload hook に上書きされて無効になる。この順序違いに気づくまで 3 時間溶かした。

---

## LoRA 適用は offload 有効化の後・VAE dtype 固定の前

```python
pipe.enable_model_cpu_offload()  # 1. offload

pipe.load_lora_weights("./lora/wan_anime_v2.safetensors")
pipe.fuse_lora(lora_scale=0.85)  # 2. LoRA マージ

pipe.vae = pipe.vae.to(dtype=torch.float32)  # 3. 最後に dtype 固定
```

`fuse_lora()` はウェイトをモデルにマージするため、VAE dtype 固定の前に実行しないとマージ後に `bfloat16` へ戻る。この 3 ステップの順序を壊すと NaN フレームが再発する。後章の統合クラスではこの順序を `__init__` に封じ込めてシーケンスミスを構造的に防ぐ。

---

## torch.compile は Wan 2.2 で無効化が正解（47.2秒 → 44.1秒）

```python
# ❌ compile ON → 初回 180秒、以降も遅い
pipe.unet = torch.compile(pipe.unet, mode="reduce-overhead")

# ✅ compile OFF（何も書かない）
```

実測（512px / 20steps / 同一プロンプト / 5回平均）：

| 設定 | 生成時間 |
|------|---------|
| torch.compile ON | 47.2 秒 |
| torch.compile OFF | 44.1 秒 |

compile ON のほうが遅い。offload の動的 device 移動が compile の静的グラフ最適化と干渉するためで、SDXL や Flux では compile が有効なのと対照的。モデルごとに計測が必須で、「compile は常に速い」は誤りだとここで確認した。

---

## 6ヶ月・2,847 回生成の失敗率 0.8% 内訳

| 原因 | 件数 | 割合 |
|------|------|------|
| NaN フレーム（VAE dtype 未固定期） | 17 | 0.60% |
| VRAM 競合（並列プロセス同時起動） | 3 | 0.11% |
| LoRA 順序ミス（リファクタ直後） | 3 | 0.11% |
| 原因不明（再実行で成功） | 0 | 0.00% |

VAE を `float32` に固定した 2 ヶ月目以降、NaN フレームはゼロになった。残る 0.21%（VRAM 競合＋LoRA 順序）はいずれもコードレベルの排他制御で防止できる。次章の統合クラスでは `threading.Lock` と上記の初期化順序をラップし、この 0.21% を構造的に潰す。
