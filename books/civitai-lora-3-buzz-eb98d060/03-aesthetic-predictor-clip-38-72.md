---
title: "aesthetic-predictor×CLIPで採用率38%→72%に上げた選定器の実装"
free: false
---

topics: `["stablediffusion", "lora", "python", "clip", "ai"]`（Zenn公開設定にこの5スラッグをそのまま使う。`lora` / `clip` は既存スラッグで流入があるため省略しない）

## aesthetic-predictor v2.5 で1,400枚を0〜10点に一括スコア化する

Civitaiから収集した画像をそのまま学習に回すと、構図崩れ画像が混入してLoRAの当たり率が3割を切る。最初の関門は美的スコアによる足切りで、RTX 4070なら1,400枚を約90秒で処理できる。

```python
import torch, glob
from PIL import Image
from aesthetic_predictor_v2_5 import convert_v2_5_from_siglip

model, preprocessor = convert_v2_5_from_siglip(
    low_cpu_mem_usage=True, trust_remote_code=True)
model = model.to(torch.bfloat16).cuda()

scores = {}
for path in glob.glob("raw/*.png"):
    img = Image.open(path).convert("RGB")
    px = preprocessor(images=img, return_tensors="pt").pixel_values
    with torch.inference_mode():
        scores[path] = model(px.to(torch.bfloat16).cuda()).logits.item()
```

## CLIP ViT-L/14 でコンセプト一致度0.28未満を捨てる

美的スコアが高くても「学習したい概念と無関係な美麗画像」は毒になる。コンセプト文とのCLIP類似度で第2フィルタをかける。閾値0.28は失敗LoRA 21本の学習セットを遡って逆算した値で、0.25まで緩めると概念ドリフトが目視で増えた。

```python
import open_clip

clip_model, _, prep = open_clip.create_model_and_transforms(
    "ViT-L-14", pretrained="openai")
tok = open_clip.get_tokenizer("ViT-L-14")
text = tok(["a flat-color anime girl with twin tails"]).cuda()

def concept_sim(path: str) -> float:
    img = prep(Image.open(path)).unsqueeze(0).cuda()
    with torch.inference_mode():
        i = clip_model.encode_image(img)
        t = clip_model.encode_text(text)
    return torch.cosine_similarity(i, t).item()
```

## insightface で顔面積比12%未満の引き画を除外する

キャラ系LoRAでは顔が小さすぎる引き画が顔崩れの主因になる。insightfaceのRetinaFaceで顔bboxを取り、画像面積に対する顔面積比12%を下限にする。風景LoRAではこの段を丸ごとスキップする設計にしておく。

```python
from insightface.app import FaceAnalysis
import cv2

app = FaceAnalysis(name="buffalo_l")
app.prepare(ctx_id=0, det_size=(640, 640))

def face_ratio(path: str) -> float:
    img = cv2.imread(path)
    faces = app.get(img)
    if not faces:
        return 0.0
    x1, y1, x2, y2 = faces[0].bbox
    return ((x2 - x1) * (y2 - y1)) / (img.shape[0] * img.shape[1])
```

## 固定閾値 aesthetic 5.5 が採用率38%で止まった実験ログ

運用初期4本のLoRAは「aesthetic 5.5以上を採用」で固定運用し、学習後の自己評価(生成100枚中の合格枚数)は38%で頭打ちだった。原因は収集元によるスコア分布の差で、写実系収集セットは平均6.1、フラット塗りセットは平均4.8。固定値5.5はフラット塗りでは厳しすぎて学習枚数が80枚を割り、過学習を起こしていた。分布の上位40パーセンタイル方式に変えた5本目以降、採用率は72%まで回復した。

```python
import numpy as np

vals = np.array(list(scores.values()))
threshold = np.percentile(vals, 60)  # 上位40%を採用
selected = [p for p, s in scores.items()
            if s >= threshold
            and concept_sim(p) >= 0.28
            and face_ratio(p) >= 0.12]
print(f"採用 {len(selected)}/{len(scores)} 枚 (閾値={threshold:.2f})")
```

## 閾値を0.5動かすと何が変わるかを sd-scripts で目視比較する

パーセンタイル方式でも最後は目視確認が要る。閾値を±0.5振った3つの学習セットを同一seed・同一stepでkohya-ss/sd-scriptsに通し、生成結果を横に並べるスクリプトを付ける。手元の検証では閾値+0.5で線の安定性が上がる一方、ポーズ多様性が体感2割落ちた。このトレードオフを自分のジャンルで一度測ってから本番運用に入ると、失敗LoRA分の電気代とCivitai掲載枠を無駄にしない。

```bash
for delta in -0.5 0.0 +0.5; do
  python select.py --percentile-shift $delta --out "set_${delta}"
  accelerate launch sd-scripts/train_network.py \
    --train_data_dir "set_${delta}" --seed 42 \
    --max_train_steps 1600 --output_name "lora_${delta}"
done
python compare_grid.py --models "lora_-0.5,lora_0.0,lora_+0.5" --seed 42
```
