---
title: "第1章 配布リポジトリで動かす: Claudeノイズ判定が15分で動くまで"
free: true
---

## 15分の内訳: clone から初回判定まで

このリポジトリは Python 3.11 と `anthropic` / `tweepy` の2依存だけで動く。所要時間は clone 1分・依存導入3分・`.env` 入力5分・初回実行6分の計15分。

```bash
git clone https://github.com/example/x-noise-filter.git
cd x-noise-filter
python -m venv .venv && .venv\Scripts\activate
pip install -r requirements.txt   # anthropic==0.40 tweepy==4.14
```

## .env に2キーだけ入れる

必要なのは X API の Bearer Token と Claude の API キーの2つ。両方とも無料枠で初回実行まで到達できる。

```bash
# .env
X_BEARER_TOKEN=AAAAAAAAAAAA...        # X Developer Portal の Free tier
ANTHROPIC_API_KEY=sk-ant-...
CLAUDE_MODEL=claude-haiku-4-5-20251001
```

## 直近200件をTLから取得する

`fetch_timeline.py` が認証ユーザーのホームTL直近200件を取得し `data/raw.json` に落とす。Free tier の取得上限に合わせ100件×2ページに分割している。

```python
import os, json, tweepy
client = tweepy.Client(os.environ["X_BEARER_TOKEN"])
me = client.get_me().data.id
tweets, token = [], None
for _ in range(2):  # 100件×2 = 200件
    r = client.get_home_timeline(max_results=100, pagination_token=token)
    tweets += [{"id": t.id, "text": t.text} for t in r.data]
    token = r.meta.get("next_token")
    if not token: break
json.dump(tweets, open("data/raw.json", "w"), ensure_ascii=False)
```

## Claudeに7ラベルで分類させる(プロンプト全文)

半年の運用で誤ミュート率を11%→1.8%まで下げた現行プロンプトをそのまま載せる。ラベルは `spam / 宣伝 / 愚痴 / 連投 / 有益 / 交流 / 不明` の7種。

```python
import anthropic, json
PROMPT = """あなたはXのTL選別器。各投稿を次の7ラベルから1つで分類し、
理由を15字以内で添えてJSON配列のみ返す: spam,宣伝,愚痴,連投,有益,交流,不明。
判定基準: 学び・一次情報・具体的数値があれば有益。煽り/定型営業はspam,宣伝。
迷えば不明(ミュート対象外)。出力例: [{"id":1,"label":"spam","why":"定型DM誘導"}]"""
cli = anthropic.Anthropic()
def classify(tweets):
    msg = cli.messages.create(model=os.environ["CLAUDE_MODEL"], max_tokens=2000,
        messages=[{"role":"user","content":PROMPT+"\n"+json.dumps(tweets, ensure_ascii=False)}])
    return json.loads(msg.content[0].text)
```

## 初回実行の生ログ: 48件中19件がノイズ

検証アカウントでの初回出力。200件のうち下書き含む有効48件で、`spam/宣伝/愚痴/連投` 合計19件がミュート候補に挙がった。

```json
[
  {"id": 1772, "label": "spam",  "why": "副業DM誘導"},
  {"id": 1781, "label": "有益",  "why": "API障害の一次報告"},
  {"id": 1795, "label": "宣伝",  "why": "自著の連続告知"},
  {"id": 1802, "label": "不明",  "why": "文脈不足"}
]
```

```bash
python run.py   # data/mute_candidates.csv に19件を書き出し
# => 48 classified / 19 noise (spam:7 宣伝:5 愚痴:4 連投:3)
```

`mute_candidates.csv` がこの時点で手元に残る資産になる。ただし初回の19件には誤判定も混じる——この生ログをそのまま `/mutes` に流すと巻き込みが起きる。2章では誤ミュート率を1.8%へ詰める検証手順と、月¥2,400に収めたコスト設計を扱う。
