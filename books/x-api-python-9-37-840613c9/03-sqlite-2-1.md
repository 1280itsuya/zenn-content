---
title: "ノイズ判定スコアリング実装：正規表現+SQLiteで誤ミュート率2.1%に抑える"
free: false
---

## 判定1件¥0・3msの根拠：Claude Haiku APIとの費用対効果を実測比較

判定エンジンにLLM APIを使わない理由は信条ではなく算数だ。1日600通知×90日=54,000件を判定した場合の実測コストを並べる。

| 方式 | 単価 | 54,000件 | レイテンシ(中央値) |
|---|---|---|---|
| Claude Haiku 4.5 API | 約¥0.08/件 | 約¥4,300 | 820ms |
| 正規表現+SQLite | ¥0 | ¥0 | 3ms |

通知ノイズ判定は「絵文字が異常に多い」「プロフィールに副業ワード」という表層パターンで決着する問題で、文脈理解が要らない。要らない能力に¥4,300払う理由はない。

```bash
pip install emoji==2.12.1  # 依存はこれ1個。sqlite3とreは標準ライブラリ
```

## 4軸スコアのSQLiteスキーマ：scoresテーブルに90日分54,000行

スコアは通知元アカウント単位で集計する。1ツイート単位だと判定が暴れる（後述の誤ミュート事故の遠因）。

```sql
CREATE TABLE IF NOT EXISTS account_scores (
  user_id TEXT PRIMARY KEY,
  emoji_density REAL,      -- 絵文字数/本文文字数の平均
  link_ratio REAL,         -- 外部リンク含有ツイート率
  interval_cv REAL,        -- 投稿間隔の変動係数(低い=機械的)
  profile_ng INTEGER,      -- NGワードヒット数
  total_score REAL,
  whitelisted INTEGER DEFAULT 0,
  updated_at TEXT
);
```

`interval_cv`がこの設計の肝で、bot系アカウントは投稿間隔の変動係数が0.3を切る。人間は0.8以上に散らばる。実測54,000行での分離は明瞭だった。

## 正規表現4軸の実装：プロフィールNGワードは12パターンで足りる

```python
import re, emoji, statistics

NG_PROFILE = re.compile(
    r"副業|権利収入|不労所得|DM(ください|まで|誘導)|"
    r"公式LINE|稼げる|月収\d+万|脱サラ|自由なライフ|"
    r"投資で人生|限定\d+名|無料プレゼント"
)

def score_account(tweets: list[str], profile: str, intervals: list[float]) -> float:
    text = "".join(tweets)
    emoji_density = emoji.emoji_count(text) / max(len(text), 1)
    link_ratio = sum("https://" in t for t in tweets) / len(tweets)
    cv = statistics.stdev(intervals) / statistics.mean(intervals) if len(intervals) > 2 else 1.0
    ng_hits = len(NG_PROFILE.findall(profile))
    # 重みは90日ログのロジスティック回帰係数を丸めた値
    return emoji_density * 25 + link_ratio * 2.0 + max(0, 0.3 - cv) * 10 + ng_hits * 1.5
```

閾値は3.0。これを超えたアカウントをミュート候補キューに積む。

## 誤ミュート19件の事故解剖：技術カンファレンス公式を巻き込んだ

初版（閾値2.5・ホワイトリストなし）を1週間回した結果、ミュート226件中19件が誤検知（誤検知率8.4%）。内訳を出すと原因は2系統に絞れた。

```python
# 誤ミュート19件の分類（実ログから集計）
false_positives = {
    "カンファレンス公式(絵文字+リンク多用)": 11,  # PyCon JP等。告知は構造的にスパムと同型
    "定時投稿の技術bot(interval_cv<0.3)": 6,     # GitHub Trending通知など意図的に機械的
    "NGワード偶発ヒット": 2,                      # 「副業禁止規定の解説」記事author
}
```

告知アカウントは「絵文字多い・リンク多い・定時投稿」とスパムの3軸が全部一致する。スコア設計だけでの分離は不可能と判断し、機構を1個足した。

## ホワイトリストで8.4%→2.1%：フォロー済み+認証済みは判定前に除外

修正は2コミット。①自分がフォロー中のアカウントを判定前に弾く、②`verified`フラグ持ちは閾値を3.0→4.5に引き上げる。

```python
def should_mute(user_id: str, score: float, is_following: bool, verified: bool, db) -> bool:
    if is_following:
        return False  # commit 8f3c2a1: フォロー済みは無条件除外
    threshold = 4.5 if verified else 3.0  # commit b71e9d4: 認証済みは閾値1.5倍
    if score < threshold:
        return False
    db.execute("UPDATE account_scores SET whitelisted=0, total_score=? WHERE user_id=?",
               (score, user_id))
    return True
```

再計測（その後30日・ミュート187件）で誤検知4件、誤検知率2.1%。残る4件は捨てアカウントの転生で、これは次章のレートリミット監視と組み合わせて潰す。フォロー除外だけで19件中17件が消えた事実は、「ノイズ判定の最強の特徴量は自分の過去の意思決定」という身も蓋もない結論を示している。
