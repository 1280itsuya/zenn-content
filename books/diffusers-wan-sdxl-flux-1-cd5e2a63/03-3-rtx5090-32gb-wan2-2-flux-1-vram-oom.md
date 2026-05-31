---
title: "第3章 RTX5090/32GBでWan2.2とFlux.1を共存させるVRAM管理とOOM回避ログ"
free: false
---

## Flux.1-devが24.1GB・Wan2.2が要offloadというVRAM実測値

RTX5090/32GBでも、Flux.1-devとWan2.2を同一プロセスに同時常駐させると落ちる。実測ピークは下記。

| モデル | ロード直後 | 生成ピーク | offload要否 |
|---|---|---|---|
| Flux.1-dev (bf16) | 24.1GB | 27.8GB | 任意 |
| SDXL (fp16) | 6.9GB | 9.2GB | 不要 |
| Wan2.2 14B | 31.4GB | OOM | 必須 |

```python
import torch
def vram_mib() -> int:
    return torch.cuda.max_memory_allocated() // (1024**2)
```

Flux常駐中にWanをloadした瞬間、合計55GB超を要求し確実にOOMになる。

## enable_model_cpu_offloadとsequential_cpu_offloadのMiB差

Wan2.2単体での3手法のピークVRAMと生成時間（832x480, 25step）を比較する。

```python
pipe.enable_model_cpu_offload()        # ピーク14,820MiB / 312秒
pipe.enable_sequential_cpu_offload()   # ピーク6,140MiB  / 781秒
# offloadなし                           # 31,400MiB → OOM
```

`model_cpu_offload`はモジュール単位で退避し2.5倍速、`sequential`はレイヤ単位で半分のVRAMだが2.5倍遅い。32GBなら前者を既定にする。

## teardown()でアダプタ破棄してから次をload

切替時のOOMはdispatchレイヤのteardownで解決した。`del`だけでは参照が残りキャッシュが解放されない。

```python
def teardown(self):
    if self.pipe is not None:
        self.pipe.to("cpu")
        del self.pipe
        self.pipe = None
    torch.cuda.empty_cache()
    torch.cuda.synchronize()
```

`empty_cache()`の後に`synchronize()`を入れないと、次のloadが解放完了前に走り約3.2GBの残留でOOMする。

## OOMで7回落ちた切替ログと遅延ロード

Flux→Wan→SDXLを連続実行した初期版は7回中6回OOM。原因はload先行・破棄後回し。遅延ロードで0回に。

```python
class Dispatcher:
    def switch(self, name: str):
        if self.current == name:
            return self.pipe
        self.teardown()          # 先に破棄
        self.pipe = self._load(name)  # 空いてからload
        self.current = name
        return self.pipe
```

teardown→load順で残留VRAMが0.4GB以下に収まり、切替成功率が14%→100%になった。

## VAE tiling/slicingの生成時間とピークのトレードオフ

Flux.1-devで1024x1024生成時、VAE設定でピークが変わる。

```python
pipe.vae.enable_tiling()   # ピーク27.8→23.1GB / +1.4秒
pipe.vae.enable_slicing()  # ピーク27.8→25.6GB / +0.3秒
```

`tiling`は4.7GB削減だが+1.4秒。32GBでFlux単体なら無効、Wanと交互運用するなら有効化して余裕を作る。

---

topics: `python` / `ai` / `stablediffusion` / `gpu` / `machinelearning`
