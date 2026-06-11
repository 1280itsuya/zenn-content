---
title: "第5章: GitHub Actions×Discordで毎朝PV差分を通知する配線"
free: false
---

## 2段ゲート: test_probe.py が23ケース全greenの時だけkpi.csvへ追記する

このBookで何度も壊れたのは「probeのコードは無事なのにkpi.csvが汚れる」事故だった。原因の8割は、HTML構造変更でviewが0になった行をそのまま追記したこと。だからActions側で`pytest`をpre-stepに置き、exit code 0以外なら追記処理に進ませない。

```yaml
# .github/workflows/pv-probe.yml
name: pv-probe
on:
  schedule:
    - cron: '0 22 * * *'   # JST 07:00
  workflow_dispatch:
jobs:
  probe:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - run: pip install -r requirements.txt
      - name: gate1 - 23 test cases
        run: pytest tests/test_probe.py -q   # 1件でもredならここで停止
      - name: gate2 - append kpi
        run: python probe.py --append kpi.csv
```

`pytest`が落ちれば`gate2`は実行されない。半年で14回、この壁が壊れたcsvへの追記を止めた。

## actions/checkout + commit で kpi.csv を永続化し履歴を残す

Actionsのランナーは毎回破棄されるので、前日比を取るには`kpi.csv`をリポジトリへcommitして戻す。差分計算の基準点を失うと`diff_pv`が全slugでNaNになり、これが過去に3回起きた。

```bash
# append後にcommit。変更が無い日はcommitしない(空commit汚染を防ぐ)
git config user.name "pv-bot"
git config user.email "bot@users.noreply.github.com"
git add kpi.csv
git diff --cached --quiet || git commit -m "kpi: $(date -u +%F)"
git push
```

`git diff --cached --quiet`を挟まないと、view据え置きの日でも空pushが走りhistoryが膨らむ。

## PAT平文混入を防ぐ secrets 運用と90日ローテーション

`git push`の認証でPATをymlに直書きすると、Actionsログとリポジトリ両方に平文が残る。`secrets.PV_PAT`経由でのみ注入し、URLにも露出させない。

```yaml
      - name: push with PAT
        env:
          PV_PAT: ${{ secrets.PV_PAT }}
        run: |
          git remote set-url origin \
            "https://x-access-token:${PV_PAT}@github.com/${GITHUB_REPOSITORY}.git"
          git push
```

PATは`contents:write`のfine-grained1個に絞り、有効期限90日で発行。期限切れの`gate2`サイレント失敗を避けるため、Settings → Secrets画面に更新日をメモしてカレンダー登録する。

## diff_pv > 50 の slug だけを Discord Webhook へ通知する

全slugを毎朝流すと通知が80行を超えて誰も読まなくなった。前日比+50超だけに絞ると、通知は平均2.7行に収まり「伸びた記事=勝ちテーマ」が一目で分かる。

```python
import csv, os, requests
prev, cur = load_two_days("kpi.csv")   # {slug: views} を2日分
hits = [(s, cur[s]-prev.get(s, 0)) for s in cur if cur[s]-prev.get(s, 0) > 50]
if hits:
    body = "\n".join(f"📈 {s} +{d}" for s, d in sorted(hits, key=lambda x:-x[1]))
    requests.post(os.environ["DISCORD_WEBHOOK"], json={"content": body}, timeout=10)
```

しきい値50は、自然変動(±20前後)とバズの境目を実データから決めた値。10にすると毎朝ノイズで埋まる。

## ワークフロー失敗時も Discord へ飛ばし沈黙故障を潰す

最悪のパターンは「通知が来ない＝平和」と錯覚し、実は4日間`gate1`がredで計測が止まっていた事故。成功通知だけでは沈黙故障を検知できないので、`if: failure()`で失敗も必ず鳴らす。

```yaml
      - name: alert on failure
        if: failure()
        run: |
          curl -s -H "Content-Type: application/json" \
            -d "{\"content\":\"🛑 pv-probe failed: ${GITHUB_RUN_ID}\"}" \
            "${{ secrets.DISCORD_WEBHOOK }}"
```

これで「通知が来た＝勝ち記事 or 故障」の二値に集約され、来ない朝＝正常稼働、という解釈が成立する。23ケースのテストとこの失敗通知が揃って、初めてKPI可観測化は閉じる。
