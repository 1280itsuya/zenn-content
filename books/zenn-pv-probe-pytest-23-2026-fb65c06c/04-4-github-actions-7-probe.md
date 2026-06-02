---
title: "第4章 GitHub Actionsで毎朝7時にprobeテスト→本番追記を自動回帰させる"
free: false
---

第4章の本文です。

---

## cron `0 22 * * *` で朝7時JSTにprobeテスト23件を回す

結論から言うと、手元実行をゼロにする鍵は「テストが緑のときだけ本番kpi.csvへ追記する」分岐をCIに組み込むことだ。UTCの`0 22 * * *`はJST朝7時に一致する。GitHub Actionsのスケジュールは数分遅延するため、KPI追記は「7時前後」で設計する。

```yaml
# .github/workflows/probe.yml
name: zenn-pv-probe
on:
  schedule:
    - cron: "0 22 * * *"   # UTC 22:00 = JST 07:00
  workflow_dispatch: {}
jobs:
  probe:
    runs-on: ubuntu-22.04
    timeout-minutes: 10
```

## actions/cache でpip依存を14秒に短縮しpytest 23件を実行

`actions/setup-python@v5`の`cache: pip`で2回目以降の依存解決が初回90秒台から14秒前後まで落ちる。テスト23件が1件でも赤なら、後続のKPI追記ジョブはスキップされる。

```yaml
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip
      - run: pip install -r requirements.txt
      - name: run 23 probe tests
        run: pytest tests/ -v --maxfail=1 --junitxml=report.xml
```

## secretsでZENN_SESSION・DISCORD_WEBHOOKを注入する

トークンをコードに置かない。`Settings > Secrets and variables > Actions`に登録し、`env:`で渡す。第3章のprobeはこの環境変数を読む前提にしておく。

```yaml
      - name: append KPI on green
        if: success()
        env:
          ZENN_SESSION: ${{ secrets.ZENN_SESSION }}
        run: |
          python probe/append_kpi.py >> kpi.csv
          git config user.name "probe-bot"
          git config user.email "bot@users.noreply.github.com"
          git add kpi.csv && git commit -m "kpi: $(date -u +%F)" && git push
```

## テスト失敗時はDiscord Webhookで「probeが壊れた」を即通知

偽の横ばいが起きる最悪のケースは、probeが静かに壊れてKPIが前日値のまま追記され続けることだ。`if: failure()`で必ず通知を飛ばし、横ばいの正体が「成長停止」か「計測故障」かを朝のうちに切り分ける。

```yaml
      - name: alert on broken probe
        if: failure()
        run: |
          curl -X POST "${{ secrets.DISCORD_WEBHOOK }}" \
            -H "Content-Type: application/json" \
            -d '{"content":"⚠️ 計測probeが壊れました。23テスト中失敗あり。kpi.csv追記はスキップ済み"}'
```

## 失敗ジョブのHTMLをartifactに保存して翌朝に原因解析する

probeが落ちた日のZenn側HTMLを`actions/upload-artifact@v4`で残す。セレクタ変更で壊れたのか、レイアウト改修なのかを、再現テストにそのHTMLを食わせて特定できる。

```yaml
      - name: save fetched html for postmortem
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: probe-html-${{ github.run_id }}
          path: probe/_dump/*.html
          retention-days: 14
```

artifactに保存したHTMLは`tests/test_parse_fixture.py`の固定入力に追加し、同じ壊れ方を二度と通さない回帰テストへ昇格させる。これで手元実行ゼロのまま、KPIが信頼できる状態で溜まり続ける。
