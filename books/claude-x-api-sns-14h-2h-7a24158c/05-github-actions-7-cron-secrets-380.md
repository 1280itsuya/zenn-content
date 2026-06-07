---
title: "GitHub Actionsで毎朝7時に無人運用：cron・secrets・コスト¥380の維持"
free: false
---

## UTC固定のcronでJST6:50に起動する

GitHub Actionsの`schedule`はUTC固定で、JSTの指定はできない。JST 7:00直前に動かすなら9時間引いて前日22:50 UTCになる。`'50 22 * * *'`だ。

```yaml
# .github/workflows/mute.yml
name: x-auto-mute
on:
  schedule:
    - cron: '50 22 * * *'   # = JST 07:50? いいえ、UTC22:50 = JST07:50ではなく翌日07:50… 計算注意
  workflow_dispatch:          # 手動実行ボタンも残す
```

JST = UTC+9。「JST 6:50に動かす」なら `50 21 * * *`（UTC21:50）。ここを1時間ずらすと収集が回らないので、初回は`workflow_dispatch`で手押し検証する。

## secretsにAPIキーとcookieを入れる

XのログインcookieとClaude APIキーはリポジトリに直書きせず`Settings > Secrets`へ。cookieはJSON1個を丸ごと文字列で格納する。

```bash
gh secret set X_COOKIE_JSON < cookies.json
gh secret set ANTHROPIC_API_KEY --body "sk-ant-..."
gh secret set DISCORD_WEBHOOK --body "https://discord.com/api/webhooks/..."
```

```yaml
      - name: Run
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          X_COOKIE_JSON: ${{ secrets.X_COOKIE_JSON }}
        run: python mute.py
```

## Playwrightのブラウザを30秒でキャッシュする

Actions上で毎回`playwright install`すると約90秒かかる。`actions/cache`でChromiumを保存すると2回目以降は約30秒に縮む。

```yaml
      - uses: actions/cache@v4
        with:
          path: ~/.cache/ms-playwright
          key: pw-${{ hashFiles('requirements.txt') }}
      - run: pip install -r requirements.txt
      - run: playwright install --with-deps chromium
```

## 失敗時のみDiscordへ通知する

成功通知は不要。`if: failure()`で落ちた時だけ叩く。これで通知ノイズがゼロになり、放置運用が成立する。

```yaml
      - name: Notify on failure
        if: failure()
        run: |
          curl -H "Content-Type: application/json" \
            -d '{"content":"❌ x-auto-mute failed: ${{ github.run_id }}"}' \
            "${{ secrets.DISCORD_WEBHOOK }}"
```

## 月¥380・稼働率99%の実コスト

Actionsはpublicリポジトリで無料、privateでも月2,000分枠に対し本ジョブは1日約2分=月60分で枠内。費用はClaude API（Haiku、約1.2万件分類）の¥380のみ。

```python
# 月次コスト概算
runs_per_month = 30
minutes_per_run = 2
print(runs_per_month * minutes_per_run, "min / 2000 free")  # 60 min
claude_haiku_yen = 380
print("monthly total:", claude_haiku_yen, "yen")            # 380 yen
```

半年で停止は2回。①X仕様変更でセレクタ失効→`mute.py`のXPath修正、②cookie失効→`gh secret set X_COOKIE_JSON`で再投入。どちらも復旧は10分以内、稼働率は99.0%（181/183日）だった。
