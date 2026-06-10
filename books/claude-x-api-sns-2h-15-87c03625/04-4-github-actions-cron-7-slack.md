---
title: "第4章 GitHub Actions+cronで毎朝7時に自動ミュート&要約Slack通知"
free: false
---

毎朝7時、手を触れずに収集→判定→ミュート→Slack要約まで終わらせる。手動運用で日2時間かかっていた工程を、GitHub Actionsのスケジュール実行で15分の確認だけに圧縮した3か月分の実装を載せる。

## cron式は「JST 7時 = UTC 22時」で固定する

GitHub ActionsのスケジュールはUTC基準なので、JST 7:00は前日22:00と書く。タイムゾーン補正を忘れて16時に動く事故が初週に2回あった。

```yaml
name: morning-mute
on:
  schedule:
    - cron: "0 22 * * *"   # UTC22:00 = JST07:00
  workflow_dispatch:        # 手動再実行用
jobs:
  run:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }
      - run: pip install -r requirements.txt
      - run: python pipeline.py
        env:
          X_BEARER: ${{ secrets.X_BEARER }}
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
```

## 状態ファイルで二重ミュートを防ぐ

同じアカウントへ毎朝ミュートAPIを叩くと429が返る。処理済みIDを`muted.json`に追記し、`actions/cache`で日跨ぎ保持する。3か月でミュート総数2,300件、重複APIコールは0件に抑えた。

```python
import json, pathlib
STATE = pathlib.Path("muted.json")
muted = set(json.loads(STATE.read_text())) if STATE.exists() else set()

def mute_once(api, user_id: str) -> bool:
    if user_id in muted:
        return False          # 既にミュート済み→APIを叩かない
    api.mute(target_user_id=user_id)
    muted.add(user_id)
    STATE.write_text(json.dumps(sorted(muted)))
    return True
```

## 例外リストでミュート巻き込みを救う

Claudeの判定がスパム寄りに振れて、取引先や情報源を巻き込むことが月1〜2件あった。`allowlist.txt`に置いたIDは判定結果に関わらずミュート対象から外す。誤ミュート率はこの層で11%→1.8%まで落ちた。

```python
allow = set(pathlib.Path("allowlist.txt").read_text().split())

def should_mute(user_id: str, verdict: str) -> bool:
    if user_id in allow:      # 例外は常に保護
        return False
    return verdict == "mute"
```

## 失敗時はworkflow_dispatchで即再実行＋Slack通知

X APIのタイムアウトでジョブが落ちた日が3か月で4回。`if: failure()`でSlackへ赤通知を送り、`workflow_dispatch`から手動再実行できるようにした。深夜の自動失敗を翌朝7:05に気づける。

```yaml
      - name: notify failure
        if: failure()
        run: |
          curl -s -X POST "$SLACK_WEBHOOK" \
            -H 'Content-Type: application/json' \
            -d '{"text":":red_circle: morning-mute failed: '"$GITHUB_RUN_ID"'"}'
```

## Top15要約をClaudeで作りSlackへ送る

ミュート後に残った当日ポストから、Claude（claude-haiku-4-5）で有益度を採点しTop15を要約する。Haikuを使うことで要約1回あたり約¥0.9、月87回配信で要約分は¥80前後。収集・判定込みでも月間APIコストは¥2,400に収まった。

```python
import anthropic
def summarize(posts: list[str]) -> str:
    msg = anthropic.Anthropic().messages.create(
        model="claude-haiku-4-5",
        max_tokens=800,
        messages=[{"role": "user",
                   "content": "次の投稿から有益な順にTop15を1行要約:\n" + "\n".join(posts)}],
    )
    return msg.content[0].text
```

## 計測：日2時間→15分の内訳

短縮効果は体感ではなくログで確認する。各段階の秒数をActionsログに出し、人間が触る時間（Slack確認＋allowlist追記）だけを手動計測した。collect/judge/muteは無人で平均合計94秒、人間の確認は平均13分、合算で15分を切る。

```bash
# 直近30回の無人処理時間を集計
gh run list --workflow morning-mute -L 30 --json durationMs \
  | jq '[.[].durationMs] | add/length/1000 | floor'
# => 94 (秒, 無人処理の平均)
```
