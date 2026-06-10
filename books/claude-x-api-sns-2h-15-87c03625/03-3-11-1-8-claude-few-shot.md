---
title: "第3章 誤ミュート率11%→1.8%: Claudeプロンプトとfew-shotの詰め方"
free: false
---

## 初版プロンプトが有益ポストを11%取りこぼした原因

初版は「ノイズなら mute」とだけ指示し、ラベル定義が一行だった。180件の人手アノテーションと突き合わせると、誤ミュートの内訳は皮肉42%・自分の専門領域の宣伝告知31%・引用RT27%で、すべて「宣伝っぽい文面」に引きずられていた。

```python
# v1: ラベル定義が曖昧で誤ミュート11%
PROMPT_V1 = """次のポストがノイズなら mute、有益なら keep を返す。
post: {text}"""
```

## ルブリックで keep/mute の境界を固定する

曖昧語を消し、判定軸を4条件の明示ルブリックに置き換えた。とくに「皮肉・自虐は mute しない」「自分が追う技術領域の告知は keep」を独立条件にしたことで、宣伝文面への過剰反応が消えた。

```python
PROMPT_V2 = """あなたはSNS収集の選別器。次の rubric で判定する。
keep の条件(1つでも満たせば keep):
1. 技術・副業の一次情報や数値を含む
2. 自分の追跡領域(Claude/X API/自動化)の告知
3. 皮肉・自虐でも実質的な知見を含む
mute の条件:
A. 上記に該当せず、かつ純粋な商品宣伝/アフィのみ
B. 中身のない RT のみ
post: {text}
出力: keep か mute のみ"""
```

## 失敗3件をそのまま few-shot に埋める

180件のうち v2 でもズレた境界事例3件を few-shot として固定投入した。抽象的な良例より、過去に誤った実物を入れる方が同型エラーの再発を抑えた。

```python
FEWSHOT = """例1 post:"またこの情報商材か…" -> mute
例2 post:"自虐だけど、X APIの月¥2,400運用ログ公開した" -> keep
例3 post:"【拡散希望】副業セミナー残3枠" -> mute"""

def build_prompt(text):
    return PROMPT_V2 + "\n" + FEWSHOT + f"\npost: {text}"
```

## Haiku と Sonnet のコスト精度を実測で切り分ける

全件 Sonnet では月¥6,800。Haiku 単独は誤ミュート4.3%まで戻った。そこで confidence を返させ、低信頼のみ Sonnet に再判定するルーティングで月¥2,400に収めた。

```python
import anthropic
client = anthropic.Anthropic()

def judge(text, model="claude-haiku-4-5-20251001"):
    r = client.messages.create(
        model=model, max_tokens=20,
        messages=[{"role":"user",
            "content": build_prompt(text) +
            '\nJSON: {"label":..,"confidence":0-1}'}])
    return parse_json(r.content[0].text)
```

## confidence 0.75 の保留バッファで実害を1.8%に

confidence が 0.75 未満のものは即 mute せず保留トレイへ送り、Sonnet 再判定 → それでも低ければ keep 寄せにした。この緩衝で「実害ある誤ミュート」は1.8%、再判定にかかった追加コストは全体の14%だった。

```python
def route(text):
    h = judge(text)
    if h["confidence"] >= 0.75:
        return h["label"]
    s = judge(text, model="claude-sonnet-4-6")   # 14%だけ再判定
    return s["label"] if s["confidence"] >= 0.75 else "keep"
```

| 版 | 構成 | 誤ミュート率 | 月コスト |
|----|------|------------|---------|
| v1 | Sonnet・一行定義 | 11% | ¥6,800 |
| v2 | Haiku・rubric+few-shot | 4.3% | ¥1,900 |
| v3 | 保留バッファ併用 | 1.8% | ¥2,400 |
