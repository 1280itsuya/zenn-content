---
title: "第2章 ProviderレジストリをPython Protocolで型安全に組む(SDXL/Flux/Wanの3アダプタ)"
free: false
---

> topics: `["python", "ai", "stablediffusion", "gpu", "machinelearning"]`（Zenn frontmatter の `topics` に5個すべて設定する）

結論から先に言うと、3モデルの差分は「pipeline クラス・`from_pretrained` 引数・`torch_dtype`」の3点に集約され、これを `Protocol` で縛った3アダプタに封じ込めれば、呼び出し側は `get_provider("flux")` の1行で固定できる。新モデル追加は30行のアダプタ1ファイルで完結する。

## `Protocol`で`load()`/`infer()`/`teardown()`の3メソッドを固定する

継承ではなく構造的部分型を使う。`ABC` と違いアダプタ側に基底クラス import が不要で、結合度が下がる。

```python
from typing import Protocol, runtime_checkable
from PIL.Image import Image

@runtime_checkable
class Provider(Protocol):
    name: str
    def load(self, device: str = "cuda") -> None: ...
    def infer(self, prompt: str, steps: int = 30) -> Image: ...
    def teardown(self) -> None: ...
```

## `@register("flux")`デコレータで文字列キー→クラスを宣言登録

レジストリは単なる `dict`。クラス定義の直上にデコレータを置くだけで登録が完了し、`if name == ...` の分岐が消える。

```python
_REGISTRY: dict[str, type[Provider]] = {}

def register(key: str):
    def deco(cls: type[Provider]) -> type[Provider]:
        _REGISTRY[key] = cls
        return cls
    return deco

def get_provider(key: str) -> Provider:
    return _REGISTRY[key]()
```

## SDXL/Flux/Wanで分岐する`torch_dtype`(fp16/bf16)をアダプタに閉じ込める

SDXL は `fp16`、Flux/Wan は `bf16` が安定。この差を3クラスのフィールドに固定し、呼び出し側には一切漏らさない。

```python
import torch
from diffusers import StableDiffusionXLPipeline, FluxPipeline

@register("sdxl")
class SDXLAdapter:
    name, _dtype = "sdxl", torch.float16
    _id = "stabilityai/stable-diffusion-xl-base-1.0"
    def load(self, device="cuda"):
        self.pipe = StableDiffusionXLPipeline.from_pretrained(
            self._id, torch_dtype=self._dtype).to(device)
    def infer(self, prompt, steps=30):
        return self.pipe(prompt, num_inference_steps=steps).images[0]
    def teardown(self):
        del self.pipe; torch.cuda.empty_cache()

@register("flux")
class FluxAdapter:
    name, _dtype = "flux", torch.bfloat16
    _id = "black-forest-labs/FLUX.1-dev"
    def load(self, device="cuda"):
        self.pipe = FluxPipeline.from_pretrained(
            self._id, torch_dtype=self._dtype).to(device)
    def infer(self, prompt, steps=30):
        return self.pipe(prompt, num_inference_steps=steps,
                         guidance_scale=3.5).images[0]
    def teardown(self):
        del self.pipe; torch.cuda.empty_cache()
```

WanAdapter も同型で `WanPipeline` と `bf16` を差すだけ。`guidance_scale` のような固有引数も `infer` 内に隠蔽され、3クラスの外形は完全に一致する。

## `get_provider()`でVRAM 24GBの呼び出し側を3行に保つ

利用側はキー文字列だけを知る。`teardown()` で `empty_cache()` を呼ぶため、24GB GPU でも SDXL→Flux の連続実行で OOM を踏まない。

```python
for key in ("sdxl", "flux"):
    p = get_provider(key)
    p.load()
    p.infer("a neon cat, cyberpunk").save(f"{key}.png")
    p.teardown()
```

## pytestで3アダプタの`Protocol`準拠を1関数で一括検証する

`runtime_checkable` を使えば `isinstance` でインターフェース欠落を検出できる。CI で3アダプタを横断テストし、メソッド名の typo を merge 前に弾く。

```python
import pytest
from registry import _REGISTRY, get_provider, Provider

@pytest.mark.parametrize("key", list(_REGISTRY))
def test_conforms_to_protocol(key):
    p = get_provider(key)
    assert isinstance(p, Provider)
    for m in ("load", "infer", "teardown"):
        assert callable(getattr(p, m)), f"{key}: {m} missing"
```

`_dtype` を `fp16` 固定でハードコードした旧実装では Flux が黒画像を返したが、この3クラス分離で `bf16` 差分を局所化した結果、3モデルすべてが同一の `infer(prompt)` シグネチャで動く。第3章ではこのレジストリに LoRA 適用フックを差し込み、同じ1行 API のまま重みを切り替える。
