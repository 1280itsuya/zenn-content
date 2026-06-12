---
title: "第2章 NudeNetとsafety_checkerの誤検知率を1000枚で計測しfalse positiveを3.1%→0.4%に下げる"
free: false
---

## CompVis safety_checkerが風景画1000枚中71枚を誤検知した記録

CompVis/stable-diffusion-safety-checkerは肌色や曲線に過敏で、海・夕焼け・木目の風景画1000枚で71枚(7.1%)をNSFW判定し投稿キューが停止した。原因は内部のCLIP埋め込みと17個の「concept」コサイン類似度がデフォルト閾値で発火するため。まず生の判定値を全件吐かせて分布を見た。

```python
from diffusers.pipelines.stable_diffusion.safety_checker import StableDiffusionSafetyChecker
import torch, numpy as np

def raw_scores(clip_input, checker):
    emb = checker.vision_model(clip_input)[1]
    emb = checker.visual_projection(emb)
    cos = torch.nn.functional.cosine_similarity(
        emb[:,None,:], checker.concept_embeds[None,:,:], dim=-1)
    return (cos - checker.concept_embeds_weights).max(dim=1).values  # >0 で発火
```

## NudeNet score閾値0.45→0.62で1000枚をラベル付け比較する

NudeNet(classifierではなくdetector)に切り替え、`FEMALE_BREAST_EXPOSED`等の露出クラスのscore最大値で判定した。手作業で1000枚にsafe/nsfwラベルを付け、閾値を0.45から0.62へ上げた効果を混同行列で確認する。

```python
from nudenet import NudeDetector
det = NudeDetector()
EXPOSE = {"FEMALE_BREAST_EXPOSED","FEMALE_GENITALIA_EXPOSED","BUTTOCKS_EXPOSED","MALE_GENITALIA_EXPOSED"}

def nsfw_score(path):
    ds = det.detect(path)
    s = [d["score"] for d in ds if d["class"] in EXPOSE]
    return max(s) if s else 0.0

def predict(path, th=0.62):
    return nsfw_score(path) >= th
```

## 肌色面積フィルタを足してfalse positiveを3.1%→0.4%に下げる

閾値0.62単体ではFP3.1%(31枚)。HSVで肌色画素の面積比が25%未満なら強制safeにする後段フィルタを足すと、誤発火した木目・人物イラストの背景が落ちFPは0.4%(4枚)まで下がった。

```python
import cv2, numpy as np
def skin_ratio(path):
    img = cv2.cvtColor(cv2.imread(path), cv2.COLOR_BGR2HSV)
    m = cv2.inRange(img, (0,30,60), (25,180,255))
    return m.mean()/255

def judge(path):
    if nsfw_score(path) < 0.62: return "safe"
    return "nsfw" if skin_ratio(path) >= 0.25 else "safe"
```

混同行列(1000枚)は次の通り。FN(すり抜け)は2枚に増えたがFP優先で許容した。

| | 予測safe | 予測nsfw |
|---|---|---|
| 実safe(940) | 936 | 4 |
| 実nsfw(60) | 2 | 58 |

## 判定ログをSQLiteに残し週次でNudeNet閾値を再調整する

毎回のscoreと最終判定、人手の事後ラベルをSQLiteへ蓄積し、週次でFP/FNを集計して閾値を動かす。下のクエリで先週のFP率が0.4%を超えた日だけ閾値を+0.02する運用にした。

```python
import sqlite3
db = sqlite3.connect("nsfw_log.db")
db.execute("""CREATE TABLE IF NOT EXISTS log(
  ts TEXT, path TEXT, score REAL, verdict TEXT, human TEXT)""")

def weekly_fp_rate():
    cur = db.execute("""SELECT
      AVG(CASE WHEN verdict='nsfw' AND human='safe' THEN 1.0 ELSE 0 END)
      FROM log WHERE ts > datetime('now','-7 day')""")
    return cur.fetchone()[0] or 0.0
```

## すり抜けた1枚でPinterest警告を受けた事例と対策

FN2枚のうち1枚(透け描写のイラスト、score0.58で閾値未満)がPinterestのadult content警告に触れ、該当ピンが非表示になった。score0.5〜0.62の「グレー帯」38枚を別キューに回し人手確認する三段目を追加。これで投稿1万枚あたりの警告は1件→0件になった。

```python
def route(path):
    s = nsfw_score(path)
    if s >= 0.62 and skin_ratio(path) >= 0.25: return "block"
    if 0.50 <= s < 0.62: return "manual_review"   # グレー帯38枚はここ
    return "publish"
```
