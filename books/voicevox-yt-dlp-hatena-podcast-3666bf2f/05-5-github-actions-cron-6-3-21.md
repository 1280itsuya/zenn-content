---
title: "第5章 GitHub Actions cron毎朝6時で無人運用、3週間21回実行の失敗ログと月次コスト最適化"
free: false
---

## UTC指定とJSTのズレを吸収するcron式 `0 21 * * *`

GitHub ActionsのcronはUTC固定で、JST6時配信なら`0 21 * * *`(前日21時UTC)と書く。手元の`crontab.guru`感覚で`0 6`と書くと15時配信になり、3週間21回中の初回はこれで丸1日ズレた。

```yaml
on:
  schedule:
    - cron: "0 21 * * *"   # JST 06:00, UTC基準で-9時間
  workflow_dispatch:        # 手動再実行用、審査落ちデバッグで多用
```

`workflow_dispatch`を併記すると、Spotify審査落ち時に`0 21`を待たず即再実行できる。21回中6回はこの手動トリガーで回した。

## VOICEVOX Engineをservicesで起動しhealthcheck 50550番待ち

Engine起動タイムアウトが21回中2回発生。原因はコンテナ起動直後に`/audio_query`を叩いていたこと。`services`の`health-cmd`で50021番(ローカルは50550で代用しない)の応答を待つ。

```yaml
    services:
      voicevox:
        image: voicevox/voicevox_engine:cpu-ubuntu20.04-0.21.1
        ports: ["50021:50021"]
        options: >-
          --health-cmd "curl -f http://localhost:50021/version || exit 1"
          --health-interval 10s --health-retries 12
```

`--health-retries 12`(計120秒)で待つようにして、以降19回はタイムアウトゼロ。

## artifactで中間wavを退避しSlackへ実行結果を通知

合成済みwavは`upload-artifact`で7日保持。重複配信1回はRSSの`guid`未固定が原因だったため、artifactのwavハッシュをguidに採用して恒久対策にした。

```yaml
      - uses: actions/upload-artifact@v4
        with: { name: episode-wav, path: out/*.wav, retention-days: 7 }
      - name: Notify
        if: always()
        run: |
          curl -X POST -H 'Content-type: application/json' \
            --data "{\"text\":\"podcast ${{ job.status }} $(date -u)\"}" \
            ${{ secrets.SLACK_WEBHOOK_URL }}
```

`if: always()`で失敗時もSlackに飛ぶ。Pages反映遅延1回はこの通知で5分で検知できた。

## 合成キャッシュで2000分枠の420分を260分へ38%削減

無料枠2000分中、月実測420分(1回平均20分)を消費。本文ハッシュをキーに`actions/cache`でwavを再利用し、同一Hatena記事の再合成を止めて260分(-38%)まで圧縮した。

```yaml
      - uses: actions/cache@v4
        with:
          path: out/
          key: tts-${{ hashFiles('feed/article.txt') }}
```

```bash
# キャッシュヒット率の実測 (jq不要、API直叩き)
gh api /repos/$OWNER/$REPO/actions/cache/usage \
  --jq '.active_caches_size_in_bytes'
```

記事更新が無い日はヒットし合成2分が0分になる。21回中9日がヒットし、削減分の大半をここが稼いだ。

---

topics: `claude`, `python`, `automation`, `typescript`, `ai`
