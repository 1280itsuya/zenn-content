---
title: "Civitai投稿の初動72時間でBuzzが決まる: 18本の当たりLoRA分析"
free: false
---

## topics指定とBuzz収支の前提: 62本で¥31,400

公開設定はZennと同じく流入導線が命だ。本書サンプルリポジトリの章メタデータには以下のtopicsを必ず指定する(Civitai側タグ戦略は後述のチェックリストと共通)。

```yaml
title: "Civitai投稿の初動72時間でBuzzが決まる: 18本の当たりLoRA分析"
emoji: "📈"
type: "tech"
topics: ["stablediffusion", "comfyui", "python", "lora", "ai"]
published: true
price: 1200
```

4ヶ月・62本の入金実績はリアクション由来58%、Early Access 31%、生成数連動11%。合計31,400 Buzz(換金レート$1=1,000 Buzzで約$31.4)。外した44本の学習コストは電気代込みで実質▲¥9,800だった。

## 初動72時間リアクションと累計Buzzの相関係数0.81

62本の投稿ログをpandasで突合すると、投稿後72時間のリアクション数と90日累計Buzzのピアソン相関は0.81。72時間で20リアクション未満の34本は、その後どれも累計500 Buzzを超えなかった。

```python
import pandas as pd
df = pd.read_csv("civitai_posts.csv")
corr = df["reactions_72h"].corr(df["buzz_90d"])
print(f"r={corr:.2f}")  # r=0.81
losers = df[df.reactions_72h < 20]
print((losers.buzz_90d > 500).sum())  # 0 / 34本
```

つまり72時間で挽回不能。初動に全変数を寄せる。

## Playwright半自動アップロード: 手作業22分→4分

CivitaiのアップロードUIはAPI未公開部分が多く、Playwrightでギャラリー画像投入だけ自動化する。1本あたり22分の手作業が4分に縮んだ。

```typescript
import { chromium } from "playwright";
const browser = await chromium.launchPersistentContext("./civitai-profile");
const page = await browser.newPage();
await page.goto("https://civitai.com/models/create");
await page.setInputFiles('input[type="file"]', galleryImages); // 12枚一括
await page.fill('[name="triggerWords"]', "myloraTrigger, masterpiece");
```

## 当たり18本 vs 外れ44本: 投稿はUTC 22時・ギャラリー12枚

メタデータ差分で有意だった4変数をチェックリスト化した。投稿時刻は米西海岸の朝にあたるUTC 22時台が18本中14本を占めた。

```yaml
checklist:
  post_time_utc: "22:00-23:00"   # JST 7時。タスクスケジューラと相性◎
  gallery_images: 12              # 8枚以下の当たりは1本のみ
  trigger_words: 2                # 3語以上は外れ率82%
  title_pattern: "<style> + <被写体> + XL"  # 固有名詞型が勝つ
```

## Windowsタスクスケジューラで毎朝7時に全パイプライン実行

選定→学習→投稿準備までをJST 7時(=UTC 22時)に一括起動する。

```powershell
Register-ScheduledTask -TaskName "LoraPipeline" `
  -Trigger (New-ScheduledTaskTrigger -Daily -At 7:00) `
  -Action (New-ScheduledTaskAction -Execute "python" -Argument "C:\lora\pipeline.py --upload-draft")
```

規約変更への備えとして、Buzz換金条件(現行$50最低出金)とAPI利用規約のdiffを週1でWebFetchして差分通知する1行をpipeline.pyに足してある。2025年11月のEarly Access料率改定をこの仕組みが3日前に検知し、18本中6本の価格を先回りで改定できた。
