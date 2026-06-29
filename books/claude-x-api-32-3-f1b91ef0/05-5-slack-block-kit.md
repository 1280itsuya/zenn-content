---
title: "第5章：Slack Block Kit週次サマリー＋中間帯ハイブリッドレビューで運用監視を完成させる"
free: false
---

## Slack Incoming Webhook Block Kit の週次ペイロード設計

週次サマリーに必要なデータは3種類。①7日間のミュート実行件数、②スコア分布（0〜4/5〜6/7〜10の3バンド）、③誤爆疑い件数（スコア5〜6）。SQLite集計→JSON整形→Block Kit送信の順でパイプライン化する。

```python
import sqlite3, os, requests
from datetime import datetime, timedelta

DB_PATH = "mute_log.db"
SLACK_WEBHOOK = os.environ["SLACK_WEBHOOK_URL"]

def fetch_weekly_stats(db_path: str) -> dict:
    since = (datetime.utcnow() - timedelta(days=7)).isoformat()
    con = sqlite3.connect(db_path)
    cur = con.cursor()
    total = cur.execute(
        "SELECT COUNT(*) FROM mute_log WHERE muted_at >= ?", (since,)
    ).fetchone()[0]
    low  = cur.execute(
        "SELECT COUNT(*) FROM mute_log WHERE score <= 4 AND muted_at >= ?", (since,)
    ).fetchone()[0]
    mid  = cur.execute(
        "SELECT COUNT(*) FROM mute_log WHERE score BETWEEN 5 AND 6 AND muted_at >= ?", (since,)
    ).fetchone()[0]
    high = cur.execute(
        "SELECT COUNT(*) FROM mute_log WHERE score >= 7 AND muted_at >= ?", (since,)
    ).fetchone()[0]
    con.close()
    return {"total": total, "low": low, "mid": mid, "high": high}
```

`muted_at` カラムは UTC ISO 形式で保存しておく（第2章参照）。

## スコア5〜6の中間帯 48 件を人間レビューキューへ振り分けるロジック

スコア5〜6は「AIが自信を持てない判断」を表す。このバンドを自動ミュートせず、Slack の interactive ボタン経由で人間に確認させる。`human_reviewed = 0` のフラグ管理を徹底することで、同一アカウントが翌週も再表示されるのを防ぐ。

```python
def fetch_mid_band_accounts(db_path: str, limit: int = 20) -> list[dict]:
    since = (datetime.utcnow() - timedelta(days=7)).isoformat()
    con = sqlite3.connect(db_path)
    rows = con.execute(
        """
        SELECT username, score, reason
        FROM mute_log
        WHERE score BETWEEN 5 AND 6
          AND muted_at >= ?
          AND human_reviewed = 0
        ORDER BY score DESC
        LIMIT ?
        """,
        (since, limit),
    ).fetchall()
    con.close()
    return [{"username": r[0], "score": r[1], "reason": r[2]} for r in rows]

def build_review_blocks(accounts: list[dict]) -> list[dict]:
    blocks = [
        {"type": "header", "text": {"type": "plain_text", "text": "🟡 中間帯レビューキュー（スコア5〜6）"}},
    ]
    for acc in accounts[:5]:  # Slack メッセージ上限対策で先頭5件に絞る
        blocks.append({
            "type": "section",
            "text": {"type": "mrkdwn", "text": f"*@{acc['username']}* (score={acc['score']})\n{acc['reason']}"},
            "accessory": {
                "type": "button",
                "text": {"type": "plain_text", "text": "ミュート確定"},
                "style": "danger",
                "value": acc["username"],
                "action_id": "confirm_mute",
            },
        })
    return blocks
```

## Block Kit mrkdwn で ASCII ヒストグラムを生成する send 関数一式

画像埋め込みを使わずとも `mrkdwn` コードブロックで傾向が一目で分かる。

```python
def ascii_bar(count: int, max_count: int, width: int = 20) -> str:
    ratio = count / max_count if max_count > 0 else 0
    return "█" * int(ratio * width) + "░" * (width - int(ratio * width))

def build_summary_blocks(stats: dict) -> list[dict]:
    max_c = max(stats["low"], stats["mid"], stats["high"], 1)
    hist = (
        f"低 (0〜4):  {ascii_bar(stats['low'],  max_c)} {stats['low']}件\n"
        f"中 (5〜6):  {ascii_bar(stats['mid'],  max_c)} {stats['mid']}件\n"
        f"高 (7〜10): {ascii_bar(stats['high'], max_c)} {stats['high']}件"
    )
    return [
        {"type": "header", "text": {"type": "plain_text", "text": "📊 週次ミュートサマリー"}},
        {"type": "section", "text": {"type": "mrkdwn", "text": f"*合計ミュート:* {stats['total']}件（過去7日）"}},
        {"type": "section", "text": {"type": "mrkdwn", "text": f"```\n{hist}\n```"}},
        {"type": "divider"},
    ]

def send_weekly_report(db_path: str = DB_PATH) -> None:
    stats    = fetch_weekly_stats(db_path)
    accounts = fetch_mid_band_accounts(db_path)
    blocks   = build_summary_blocks(stats) + build_review_blocks(accounts)
    resp = requests.post(SLACK_WEBHOOK, json={"blocks": blocks}, timeout=10)
    resp.raise_for_status()
    print(f"Slack 送信完了: HTTP {resp.status_code}")

if __name__ == "__main__":
    send_weekly_report()
```

## GitHub Actions で毎週月曜 09:00 JST に cron 自動実行する設定

```yaml
# .github/workflows/weekly_mute_report.yml
name: Weekly Mute Report

on:
  schedule:
    - cron: "0 0 * * 1"   # UTC 00:00 = JST 09:00（月曜）
  workflow_dispatch:

jobs:
  report:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install requests
      - name: Download latest DB artifact
        uses: dawidd6/action-download-artifact@v3
        with:
          name: mute-log-db
          path: .
      - name: Send Slack report
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
        run: python src/weekly_report.py
```

`mute_log.db` は第3章のワークフローが artifact としてアップロードするものを流用する。`SLACK_WEBHOOK_URL` を Repository secrets に追加する手順は Settings → Secrets and variables → Actions → New repository secret。

## 月次 API 費用試算: Claude API ¥54 + X API $100 の内訳

| 項目 | 単価 | 月間使用量 | 月額 |
|------|------|-----------|------|
| claude-haiku-4-5 input | $0.80/1M token | 約250,000 token | $0.20（¥30） |
| claude-haiku-4-5 output | $4.00/1M token | 約40,000 token | $0.16（¥24） |
| X API Basic プラン | 月固定 | — | $100 |
| **合計** | | | **約$100.36（¥15,100〜）** |

1日300件判定・1件あたり平均850 input / 130 output token で算出。スコア5〜6の中間帯（全体の約15%）は追加で Claude を呼ばず、ルールベースでキューに振るためコストに計上しない。X API $100 が支配的。月500件以上の手動ミュートを代替できれば時給換算で十分に黒字化する。

## 最終システム構成: 誤爆ログをプロンプトへ還元する自己改善ループ全体図

```
┌──────────────────────────────────────────────────────┐
│  X Timeline Fetcher（X API Basic / 15分間隔）         │
└──────────────────────┬───────────────────────────────┘
                       │ ツイート JSON
                       ▼
┌──────────────────────────────────────────────────────┐
│  Claude Scorer（claude-haiku-4-5）                   │
│  スコア 0〜10 + reason を JSON で返す                 │
└───────┬──────────────────────────┬────────────────────┘
        │ score <= 4 or >= 7        │ score 5〜6
        ▼                           ▼
┌──────────────┐         ┌──────────────────────┐
│ 自動ミュート  │         │ Slack レビューキュー  │
│ mute_log.db  │         │（Block Kit ボタン）   │
└──────┬───────┘         └──────────┬───────────┘
       │                            │ 人間が確定 / 却下
       └────────────┬───────────────┘
                    ▼
┌──────────────────────────────────────────────────────┐
│  週次レポーター（GitHub Actions / 月曜 09:00 JST）    │
│  Block Kit サマリー → Slack                          │
└──────────────────────────────────────────────────────┘
       │ is_false_positive=1 の誤爆フラグ
       ▼
┌──────────────────────────────────────────────────────┐
│  Prompt Refiner（第4章）                              │
│  誤爆ログ → Claude へフィードバック → 精度向上        │
└──────────────────────────────────────────────────────┘
```

中間帯の人間判定結果は `is_false_positive` カラムに書き戻し、第4章の Prompt Refiner が次の改善サイクルの素材として使う。「AIが自信を持てない判断だけ人間が確認し、その結果がプロンプトに還元される」ループを3ヶ月回し続けた結果が、誤爆率 32%→3% という実測値の正体。
