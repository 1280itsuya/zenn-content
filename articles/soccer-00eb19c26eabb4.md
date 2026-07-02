---
title: "Pythonで読む堂安律のスタッツ分析"
emoji: "⚽"
type: "tech"
topics: ["Python", "データ分析", "機械学習"]
published: true
---

## この記事でやること

堂安律はフライブルク（現在はPSV→フライブルクとキャリアを重ねてきた）を経てブンデスリーガで活躍するアタッカーです。本記事では、架空のサンプルスタッツを使いながら `pandas` と `matplotlib` で選手データを可視化・分析するチュートリアルを紹介します。

実際の公式スタッツは [FBref](https://fbref.com) や [Understat](https://understat.com) から取得可能ですが、ここでは説明用にサンプルデータを手動構築します。**実測値ではなく「例として」の数値**であることを前提に読み進めてください。

---

## 分析用データを用意する

まずシーズン別の基本スタッツをDataFrameで構築します。

```python
import pandas as pd
import matplotlib.pyplot as plt
import matplotlib.ticker as ticker

# 例として設定したサンプルデータ（実測値ではありません）
data = {
    "season": ["2020-21", "2021-22", "2022-23", "2023-24"],
    "club": ["PSV", "PSV", "Freiburg", "Freiburg"],
    "appearances": [28, 31, 29, 26],
    "goals": [7, 9, 5, 6],          # 例として設定
    "assists": [4, 6, 5, 4],         # 例として設定
    "xG": [5.8, 8.1, 4.9, 5.5],     # 例として設定
    "xA": [3.5, 5.2, 4.1, 3.8],     # 例として設定
}

df = pd.DataFrame(data)

# 得点効率 = goals / appearances
df["goal_per_game"] = (df["goals"] / df["appearances"]).round(2)

# xG超過分（実際の得点 - 期待得点）
df["xG_diff"] = (df["goals"] - df["xG"]).round(2)

print(df[["season", "club", "goals", "xG", "xG_diff", "goal_per_game"]])
```

出力例:

```
    season      club  goals   xG  xG_diff  goal_per_game
0  2020-21       PSV      7  5.8      1.2           0.25
1  2021-22       PSV      9  8.1      0.9           0.29
2  2022-23  Freiburg      5  4.9      0.1           0.17
3  2023-24  Freiburg      6  5.5      0.5           0.23
```

次に、ゴール・アシスト・xGをまとめて折れ線グラフで可視化します。

```python
fig, ax = plt.subplots(figsize=(8, 4))

ax.plot(df["season"], df["goals"],   marker="o", label="Goals")
ax.plot(df["season"], df["assists"], marker="s", label="Assists")
ax.plot(df["season"], df["xG"],      marker="^", linestyle="--", label="xG")
ax.plot(df["season"], df["xA"],      marker="v", linestyle="--", label="xA")

ax.set_title("堂安律 シーズン別スタッツ（サンプル）", fontsize=13)
ax.set_xlabel("シーズン")
ax.set_ylabel("値")
ax.yaxis.set_major_locator(ticker.MultipleLocator(2))
ax.legend()
ax.grid(axis="y", alpha=0.4)
plt.tight_layout()
plt.savefig("dooan_stats.png", dpi=120)
plt.show()
```

---

## 読み取れること

サンプルデータから読み取れるポイントを整理します。

**xG_diff が正の値**（例: 2020-21 は +1.2）であれば、期待得点を上回る決定力を示しています。堂安律のように右足のシュートに鋭さがある選手では、xGモデルが過小評価しやすいシチュエーションがある可能性があります。

**ブンデスリーガ移籍後のgoal_per_game低下**は、例として設定した数値でも見られます。これはクラブの役割変化（守備貢献増加など）によるものかもしれず、単純な"衰え"とは区別して解釈すべきです。

`xG_diff` の列は以下のように集計して確認できます。

```python
summary = df.groupby("club")[["goals", "xG", "xG_diff"]].mean().round(2)
print(summary)
```

クラブ別に分けると「どのリーグ・役割で得点効率が高かったか」が見えやすくなります。

---

## まとめ

本記事では堂安律を題材に、pandas での DataFrame 構築・集計と matplotlib による折れ線グラフ可視化の基本を示しました。

- サンプルの数値はあくまで「例として」の値であり、実測値は FBref 等から CSV/API で取得してください
- `xG_diff` のような派生指標をカラム追加することで、生スタッツだけでは見えない傾向が浮かびます
- クラブ・リーグ別の比較には `groupby` が有効です

実データを差し込むだけでそのまま動くコード構成にしているので、好きな選手のスタッツに差し替えて試してみてください。

---

## データで予想を楽しむなら
- AIが競馬・競艇・競輪を実測実績つきで無料予想する[AI予想サイト](https://1280itsuya.github.io/ai-yosou/)
