---
title: "第4章 ENV/YAMLでprovider切替を外部化しCIとバッチ生成に組み込む(seed固定込み)"
free: false
---

## DIFFUSERS_PROVIDER を pydantic-settings で読む（未設定はスキップ）

`.env` の1行でプロバイダを切り替える。`pydantic-settings` v2 で型検証し、未設定なら `None` を返して呼び出し側でスキップさせる。コードへ `if provider == "wan"` を一切書かないのがdispatch外部化の肝になる。

```python
# config.py
from pydantic_settings import BaseSettings
from typing import Literal, Optional

class RunConfig(BaseSettings):
    DIFFUSERS_PROVIDER: Optional[Literal["wan", "sdxl", "flux"]] = None
    model_config = {"env_file": ".env"}

cfg = RunConfig()
if cfg.DIFFUSERS_PROVIDER is None:
    raise SystemExit("DIFFUSERS_PROVIDER 未設定: 生成をスキップ (exit 0)")
```

## YAMLでsteps/guidance/解像度をprovider別に分離

steps・guidance・解像度はモデルごとに最適値が違う（Flux 28 / SDXL 30 / Wan 2.2 は50）。YAMLに外出しし、dispatchレイヤは値を知らないまま受け取る。

```yaml
# providers.yaml
flux:  { steps: 28, guidance: 3.5,  width: 1024, height: 1024 }
sdxl:  { steps: 30, guidance: 7.0,  width: 1024, height: 1024 }
wan:   { steps: 50, guidance: 5.0,  width: 832,  height: 480 }
```

```python
import yaml
params = yaml.safe_load(open("providers.yaml"))[cfg.DIFFUSERS_PROVIDER]
```

## seed固定で再現性を担保するバッチランナー

`torch.Generator` をプロンプトごとにseed付きで生成し、同一seed×同一YAMLならbitレベルで再現する。複数プロンプトをキューで回す。

```python
import torch
from dispatch import load_pipeline  # 第3章のdispatch

pipe = load_pipeline(cfg.DIFFUSERS_PROVIDER, **params)
prompts = ["a red fox in snow", "neon tokyo alley"]
for i, p in enumerate(prompts):
    g = torch.Generator("cuda").manual_seed(1234 + i)
    img = pipe(p, generator=g, **params).images[0]
    img.save(f"out/{cfg.DIFFUSERS_PROVIDER}_{i}_seed{1234+i}.png")
```

## GitHub Actions self-hosted GPU runnerで3provider各1枚を生成

PRごとに `flux/sdxl/wan` を1枚ずつ生成。VRAM 24GB の self-hosted runner を `runs-on` で指定し、CIへ組み込む。

```yaml
# .github/workflows/regression.yml
jobs:
  generate:
    runs-on: [self-hosted, gpu]
    strategy:
      matrix: { provider: [flux, sdxl, wan] }
    steps:
      - uses: actions/checkout@v4
      - run: DIFFUSERS_PROVIDER=${{ matrix.provider }} python batch.py
      - uses: actions/upload-artifact@v4
        with: { name: img-${{ matrix.provider }}, path: out/ }
```

## 出力ハッシュ比較で品質デグレを自動検知

モデル更新時の意図しない出力変化を、seed固定画像のSHA-256で検知する。差分があればCIを赤くして人間にレビューを促す。

```bash
# verify.sh — 期待ハッシュと照合 (差分0件で exit 0)
sha256sum out/*.png > current.sha
diff <(sort baseline.sha) <(sort current.sha) \
  && echo "回帰なし" \
  || { echo "::error::出力デグレ検知"; exit 1; }
```

ハッシュ不一致時はPRに `current.sha` をコメント添付すれば、3provider × seed1234 の差分が一目で追える。これで「diffusers `0.31→0.32` 更新でSDXLのVAEが変わった」級の事故を、マージ前に止められる。

---

**topics（Zenn公開タグ・本文整合）**: `python` / `ai` / `stablediffusion` / `gpu` / `machinelearning`
