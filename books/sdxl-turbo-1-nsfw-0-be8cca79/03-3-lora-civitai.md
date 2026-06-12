---
title: "第3章 LoRA/学習元の著作権を潰す｜Civitaiライセンス確認とキャラ寄せプロンプト禁止リスト"
free: false
---

## 申し立て2件の原因はLoRAライセンスとpHash 0.9超だった

著作権申し立てを受けた2件は、どちらも生成前のチェックで防げた。1件目は既存アニメキャラに寄りすぎたLoRA（Civitaiの`allowCommercialUse`が`None`）、2件目は実写ストックとpHash距離が0.91の酷似画像。この章では、この2軸を量産前に機械的に潰す。グレー判定を人間に任せず、しきい値で破棄する。

```text
申し立て内訳（直近90日 / 生成1万枚中2件 = 0.02%）
- case1: キャラ寄せLoRA  → Civitai license = Non-Commercial
- case2: 実写酷似        → pHash hamming distance 0.91
```

## Civitai APIで`allowCommercialUse`を取得し不可LoRAを自動除外

LoRAは導入時にCivitai REST APIでライセンス欄を取得し、商用不可を弾く。`allowCommercialUse`が`Image`または`RentCivit`以外、もしくは空配列なら除外する。

```python
import requests

def is_commercial_ok(model_version_id: int) -> bool:
    r = requests.get(
        f"https://civitai.com/api/v1/model-versions/{model_version_id}",
        timeout=10,
    )
    model_id = r.json()["modelId"]
    m = requests.get(
        f"https://civitai.com/api/v1/models/{model_id}", timeout=10
    ).json()
    allow = m.get("allowCommercialUse", [])
    # "Image"=生成画像の商用可 / "Sell"=モデル販売可
    return "Image" in allow or "Sell" in allow

# 使用例: 不可LoRAをloras/から物理隔離
if not is_commercial_ok(128713):
    print("EXCLUDE: non-commercial LoRA")
```

## キャラ・芸能人・ブランド名425件のプロンプト禁止語フィルタ

特定アニメ名・声優・芸能人・ブランド名を含むプロンプトは生成前に停止する。425件を`banlist.txt`で管理し、部分一致と正規化（全角半角・大文字小文字）で検査する。

```python
import unicodedata

BAN = {unicodedata.normalize("NFKC", w).lower().strip()
       for w in open("banlist.txt", encoding="utf-8")}

def screen(prompt: str) -> list[str]:
    norm = unicodedata.normalize("NFKC", prompt).lower()
    return [w for w in BAN if w and w in norm]

hits = screen("masterpiece, ghibli style, kaonashi")
if hits:
    raise SystemExit(f"BANNED words: {hits}")  # 生成キューに乗せない
```

## pHash距離0.9以上を既存ストックと突合して破棄

生成直後にperceptual hash（pHash）でストック画像群と突合し、ハミング類似度0.9以上を破棄する。case2の0.91はこのゲートで止まる。

```python
import imagehash
from PIL import Image

stock = [imagehash.phash(Image.open(p)) for p in stock_paths]  # 事前計算

def too_similar(img_path: str, thr: float = 0.9) -> bool:
    h = imagehash.phash(Image.open(img_path))
    for s in stock:
        sim = 1 - (h - s) / 64.0   # 64bit中の一致率
        if sim >= thr:
            return True
    return False

if too_similar("out/0001.png"):
    print("DISCARD: pHash similarity >= 0.90")
```

## 3ゲートを直列化した「グレーは出さない」量産前パイプライン

ライセンス・禁止語・pHashの3ゲートをCIに直列で並べ、1つでも落ちたら公開キューに乗せない。判定ログを残し、申し立てゼロを実測で担保する。

```yaml
# .github/workflows/pre-publish.yml
name: pre-publish-guard
on: [workflow_dispatch]
jobs:
  guard:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: python check_license.py   # 不可LoRAで exit 1
      - run: python screen_prompt.py    # 禁止語hitで exit 1
      - run: python check_phash.py      # sim>=0.9で exit 1
      - run: echo "PASS -> queue/ へ移動"
```

3ゲート導入後、生成1万枚での申し立ては0件。除外率はLoRAで7.2%、pHashで1.1%、禁止語で0.4%。落ちた分は売上ではなく凍結リスクの除去とみなす。
