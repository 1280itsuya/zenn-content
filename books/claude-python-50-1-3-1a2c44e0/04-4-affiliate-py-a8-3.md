---
title: "第4章 affiliate.pyでA8/楽天リンクを自動差込：プレースホルダ事故と収益導線の最適配置3箇所"
free: false
---

## 結論：差込は「冒頭・比較表・末尾」の3箇所に固定し、本文KWと商材YAMLをマッチングする

affiliate.py の役割は1つ。記事本文を読み、商材マスタから文脈一致するリンクを選び、決め打ちした3箇所だけに挿入する。乱挿入はスパム判定の温床になるため、1記事あたり最大3リンク・同一ASPは2回までに制限した。

```python
# affiliate.py
import yaml, re
from collections import Counter

MASTER = yaml.safe_load(open("data/affiliates.yml", encoding="utf-8"))
MAX_LINKS = 3

def pick(body: str, top_n: int = MAX_LINKS) -> list[dict]:
    words = Counter(re.findall(r"[一-龥ぁ-んァ-ヶA-Za-z0-9]+", body))
    scored = []
    for item in MASTER:                         # 格安SIM/ふるさと納税/ネット証券
        hit = sum(words[kw] for kw in item["keywords"])
        if hit >= item.get("min_hits", 2):
            scored.append((hit, item))
    scored.sort(key=lambda x: -x[0])
    return [it for _, it in scored[:top_n]]
```

## 商材マスタは48件をYAMLで一元管理し、min_hitsで誤爆を抑える

格安SIM・ふるさと納税・ネット証券の3カテゴリで計48件を `affiliates.yml` に登録。`min_hits` を2に設定し、本文に該当KWが2回未満なら差し込まない。これで「証券」が1回出ただけの記事に投資リンクが混入する事故を防ぐ。

```yaml
- name: "楽天モバイル"
  asp: "a8"
  url: "https://px.a8.net/svt/ejp?a8mat=REALCODE_RAKUTEN"
  keywords: ["格安SIM", "データSIM", "月額", "通信速度"]
  min_hits: 2
  category: "sim"
- name: "さとふる"
  asp: "rakuten"
  url: "https://hb.afl.rakuten.co.jp/REALCODE_SATOFURU"
  keywords: ["ふるさと納税", "返礼品", "ポイント還元", "ワンストップ"]
  min_hits: 2
```

## ダミーURL公開で47本が機会損失：本番ガードでREALCODE未差替を停止

テスト段階で `REALCODE_xxx` のまま50本中47本を公開し、その期間のクリックは全て収益ゼロに流れた。原因はプレースホルダ検知の欠落。以降、公開前に未差替コードを検出したら例外で停止させる品質ゲートを入れた。

```python
def assert_no_placeholder(html: str):
    leaked = re.findall(r"REALCODE_\w+", html)
    if leaked:
        raise SystemExit(f"[BLOCK] 未差替リンク {len(leaked)}件: {set(leaked)}")
# publish前に必ず呼ぶ。47本の事故後、CIで100%通過を必須化
```

## rel=sponsored・ディスクロージャ文・UTMを自動付与する

ステマ規制対応として、各リンクに `rel="sponsored nofollow"` とディスクロージャ文を自動挿入。クリック計測は UTM を付け、配置（top/table/bottom）を `utm_content` で区別する。

```python
DISCLOSURE = "> 本記事はアフィリエイト広告を含みます。\n\n"

def render(item: dict, slot: str) -> str:
    sep = "&" if "?" in item["url"] else "?"
    url = f'{item["url"]}{sep}utm_source=zenn&utm_medium=book&utm_content={slot}'
    return f'<a href="{url}" rel="sponsored nofollow">{item["name"]}を見る</a>'

def inject(body: str, picks: list[dict]) -> str:
    top, table, bottom = picks + [None]*(3-len(picks))
    out = DISCLOSURE
    if top:    out += render(top, "top") + "\n\n"
    out += body
    if bottom: out += "\n\n" + render(bottom, "bottom")
    return out
```

## 50本の配置別クリック実測：末尾CTAが冒頭の1.9倍

UTMの `utm_content` 別に50本分のクリックを集計した結果が下表。比較表内（table）が最もCTRが高く0.42%、末尾（bottom）が0.30%で冒頭（top）0.16%を1.9倍上回った。冒頭リンクは離脱前に押される想定だったが、読了後の末尾と表内が勝った。

```python
# data/click_log.csv を集計
import csv
agg = {}
for r in csv.DictReader(open("data/click_log.csv", encoding="utf-8")):
    s = r["utm_content"]; a = agg.setdefault(s, {"imp": 0, "clk": 0})
    a["imp"] += int(r["imp"]); a["clk"] += int(r["clk"])
for slot, v in agg.items():
    ctr = v["clk"] / v["imp"] * 100
    print(f"{slot:7} CTR={ctr:.2f}%  clk={v['clk']}")
# table  CTR=0.42%  clk=38
# bottom CTR=0.30%  clk=27
# top    CTR=0.16%  clk=14
```

この実測から、次章の生成テンプレートでは比較表を必ず1つ生成し、その直下に商材リンクを置く構成へ寄せた。冒頭リンクは残すが、収益の主軸は table と bottom の2箇所に置く。
