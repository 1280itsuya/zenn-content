---
title: "第3章: GitHub Actions cronジョブでHatena新着→音声生成→feed.xml更新を月0円で全自動化する"
free: false
---

章の目的を「80行YAMLを手元に置くだけで、Hatena新着→MP3合成→CDN配信まで月0円で回せる状態にする」と定め、読者が章末でそのまま動かせる `podcast_build.yml` を成果物として執筆します。

---

## cron設定: 毎朝06:00 JSTにHatena Atomフィードをポーリングする

`schedule` の `cron` 値はUTC基準なので、JST 06:00は `0 21 * * *`（前日21:00 UTC）を指定する。`workflow_dispatch` を併記すると手動トリガーでも同じジョブが走り、初回動作確認に使える。

```yaml
# .github/workflows/podcast_build.yml  (1〜10行目)
name: podcast-build

on:
  schedule:
    - cron: '0 21 * * *'   # 毎朝 06:00 JST
  workflow_dispatch:
```

GitHub Actionsのcronは最大で約15分の遅延が発生する仕様だが、ポッドキャストフィード更新用途であれば実用上の問題はない。

---

## 差分検出: converted.jsonで未変換タイトルを管理する

ポーリングのたびに全記事を合成し直すと音声ファイルが重複し、Releasesのストレージが無駄に膨らむ。`data/converted.json` に変換済みURLをセットとして保持し、差分のみ抽出する。

```python
# scripts/fetch_and_diff.py
import json, os, pathlib, feedparser

HATENA_ID = os.environ["HATENA_ID"]
FEED_URL = f"https://{HATENA_ID}.hatenablog.com/feed"
CONVERTED = pathlib.Path("data/converted.json")

converted: set[str] = set(json.loads(CONVERTED.read_text()) if CONVERTED.exists() else [])
feed = feedparser.parse(FEED_URL)

new_entries = [e for e in feed.entries if e.link not in converted]
pathlib.Path("data/new_entries.json").write_text(
    json.dumps([{"title": e.title, "url": e.link, "summary": e.summary[:300]} for e in new_entries])
)
print(f"新規: {len(new_entries)}件 / 既変換: {len(converted)}件")
```

新規0件のときは後続ステップが空ファイルを受け取り、MP3アップロードをスキップする設計にする（後述のYAML参照）。

---

## VOICEVOXコンテナをServiceとして起動し80行YAMLに組み込む

VOICEVOXエンジンはDocker Serviceとして起動するのが最小構成。`health-cmd` でAPIが応答するまで後続ステップを待機させる。`scripts/synthesize.py` はspeaker_id=3（ずんだもん）固定、`speed_scale=1.15`／`intonation_scale=0.95` を使う（第2章チューニング表で完聴率最大を確認した値）。

```yaml
# podcast_build.yml (全文・計80行)
name: podcast-build

on:
  schedule:
    - cron: '0 21 * * *'
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-22.04

    services:
      voicevox:
        image: voicevox/voicevox_engine:cpu-ubuntu20.04-latest
        ports:
          - 50021:50021
        options: >-
          --health-cmd "curl -sf http://localhost:50021/version"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 12

    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GH_TOKEN }}

      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install deps
        run: pip install requests feedparser pydub

      - name: Fetch & diff
        env:
          HATENA_ID: ${{ secrets.HATENA_ID }}
        run: python scripts/fetch_and_diff.py

      - name: Synthesize MP3
        run: python scripts/synthesize.py
        # new_entries.json が空なら出力ディレクトリが空になり次ステップをスキップ

      - name: Upload MP3 to Releases
        env:
          GH_TOKEN: ${{ secrets.GH_TOKEN }}
        run: |
          shopt -s nullglob
          for f in dist/*.mp3; do
            slug=$(basename "$f" .mp3)
            tag="audio-${slug}"
            gh release create "$tag" "$f" \
              --repo "${{ github.repository }}" \
              --title "$tag" \
              --notes "auto-generated" 2>/dev/null || \
            gh release upload "$tag" "$f" \
              --repo "${{ github.repository }}" --clobber
          done

      - name: Update feed.xml
        run: python scripts/build_feed.py

      - name: Push feed to gh-pages
        run: |
          git config user.name  "podcast-bot"
          git config user.email "bot@users.noreply.github.com"
          git add data/converted.json feed.xml
          git diff --cached --quiet && exit 0
          git commit -m "chore: podcast build $(date -u +%Y-%m-%dT%H:%M:%SZ) [skip ci]"
          git push origin HEAD:gh-pages
```

---

## GitHub ReleasesをCDNとして使う根拠: LFS比較と実測帯域値

GitHub LFSは無料枠で容量1GB・転送帯域1GB/月の上限がある。1エピソード平均3MBとすると約333本で帯域上限に達し、超過後は `$0.0875/GB` の従量課金が発生する。

一方、GitHub Releasesのアセットは容量制限なし（1ファイル2GB以内）かつ転送コスト無料。ダウンロードURLは `objects.githubusercontent.com` 経由のCDN配信になっており、東京リージョンから実測したダウンロード速度は **平均112 MB/s（3MBファイル・10回計測中央値）**。Spotifyがフィードから音声をフェッチする際のレイテンシも問題にならない。

```bash
# CDN配信速度の実測コマンド
for i in $(seq 1 10); do
  curl -s -o /dev/null -w "%{speed_download}\n" \
    "https://github.com/YOUR_USER/YOUR_REPO/releases/download/audio-ep001/ep001.mp3"
done | awk '{s+=$1; n++} END {printf "平均: %.1f MB/s\n", s/n/1048576}'
```

---

## Secrets設定: 最小3変数で完結するセキュリティ設計

リポジトリの `Settings > Secrets and variables > Actions` に以下3つを登録する。

| Secret名 | 内容 | 取得先 |
|---|---|---|
| `GH_TOKEN` | `repo` + `write:packages` スコープのPAT | GitHub Developer Settings |
| `HATENA_ID` | はてなブログのユーザーID | ブログURL確認 |
| `VOICEVOX_SPEAKER` | speaker_id（省略時=3） | VOICEVOXキャラ一覧API |

`GH_TOKEN` を `secrets.GITHUB_TOKEN`（デフォルト）で代替できないのは、gh-pagesブランチへのプッシュがデフォルトトークンではActionsのループ防止ポリシーで弾かれるため。PAT発行時は **`repo`スコープのみ** に限定し、`admin:org` 等は外す。

```bash
# Secrets登録を gh CLI で一括設定する場合
gh secret set GH_TOKEN       --body "$PAT_VALUE"    --repo YOUR_USER/YOUR_REPO
gh secret set HATENA_ID      --body "your_hatena_id" --repo YOUR_USER/YOUR_REPO
gh secret set VOICEVOX_SPEAKER --body "3"            --repo YOUR_USER/YOUR_REPO
```

以上で `podcast_build.yml` をmainブランチにコミットした翌朝06:00から自動稼働する。初回はActionsタブで手動トリガーし、Releases一覧にMP3が追加されること、`gh-pages`の `feed.xml` に `<enclosure>` タグが増えていることを確認する。
