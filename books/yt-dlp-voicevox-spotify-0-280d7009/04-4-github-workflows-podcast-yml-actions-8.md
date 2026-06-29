---
title: "第4章 .github/workflows/podcast.yml全文：差分生成キャッシュでActions時間を8分→90秒に削減"
free: false
---

## JST 07:00スケジュール起動とSHA256キャッシュキー設計

`cron: '0 22 * * *'`（UTC）がJST 07:00に対応する。GitHub-hostedランナー（Ubuntu 22.04）でそのまま実行できるため、self-hostedは不要だ。

キャッシュの核心は「既に音声化済みのエピソードURLをスキップする」ことにある。`scripts/gen_audio.py`は処理済みURLのSHA256を`.cache/audio_hashes.json`に書き込む。ワークフローは`hashFiles('.cache/audio_hashes.json')`をキャッシュキーのサフィックスに使い、変化がなければ`restore-keys`でヒットさせて音声生成ステップ自体を`if: steps.cache.outputs.cache-hit != 'true'`で条件スキップする。

```bash
# scripts/hash_episodes.sh — 全エピソードURLをSHA256化してキャッシュキー用ファイルを生成
python3 -c "
import hashlib, json, pathlib
urls = pathlib.Path('data/episodes.txt').read_text().splitlines()
hashes = {u: hashlib.sha256(u.encode()).hexdigest() for u in urls if u}
pathlib.Path('.cache').mkdir(exist_ok=True)
pathlib.Path('.cache/audio_hashes.json').write_text(json.dumps(hashes, indent=2))
"
```

---

## podcast.yml全文：contents:write最小権限＋差分スキップ構成

```yaml
# .github/workflows/podcast.yml
name: Podcast Auto-Publish

on:
  schedule:
    - cron: '0 22 * * *'   # JST 07:00
  workflow_dispatch:

permissions:
  contents: write            # gh release upload に必要な最小権限

jobs:
  generate-and-publish:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python 3.11
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Generate episode hash manifest
        run: bash scripts/hash_episodes.sh

      - name: Cache audio files
        id: cache
        uses: actions/cache@v4
        with:
          path: audio/
          key: audio-${{ hashFiles('.cache/audio_hashes.json') }}
          restore-keys: audio-

      - name: Generate audio via VOICEVOX (skip if cache hit)
        if: steps.cache.outputs.cache-hit != 'true'
        run: python3 scripts/gen_audio.py
        env:
          VOICEVOX_URL: ${{ secrets.VOICEVOX_URL }}

      - name: Upload mp3 to GitHub Releases
        run: |
          TAG="podcast-$(date -u +%Y%m%d)"
          gh release create "$TAG" --title "Podcast $TAG" --notes "" 2>/dev/null || true
          for f in audio/*.mp3; do
            gh release upload "$TAG" "$f" --clobber
          done
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Inject asset URLs into RSS enclosure tags
        run: python3 scripts/update_rss.py
        env:
          GITHUB_REPOSITORY: ${{ github.repository }}

      - name: Commit updated RSS
        run: |
          git config user.email "actions@github.com"
          git config user.name "GitHub Actions"
          git add rss/feed.xml .cache/audio_hashes.json
          git diff --cached --quiet || git commit -m "chore: update podcast RSS $(date -u +%Y-%m-%d)"
          git push
```

---

## gh release upload後にPythonがenclosure URLを自動挿入

`update_rss.py`はGitHub CLIでasset一覧を取得し、`<enclosure>`タグにURLとバイト数を上書きする。

```python
# scripts/update_rss.py
import os, re, subprocess, json
from pathlib import Path

tag = subprocess.check_output(
    ["gh", "release", "list", "--limit", "1", "--json", "tagName", "-q", ".[0].tagName"],
    text=True,
).strip()

assets = json.loads(
    subprocess.check_output(
        ["gh", "release", "view", tag, "--json", "assets", "-q", ".assets"],
        text=True,
    )
)
url_map = {a["name"]: (a["url"], a["size"]) for a in assets}

rss = Path("rss/feed.xml").read_text()
for fname, (url, size) in url_map.items():
    rss = re.sub(
        rf'(<enclosure[^>]+name="{re.escape(fname)}"[^/]*/?>)',
        f'<enclosure url="{url}" length="{size}" type="audio/mpeg"/>',
        rss,
    )
Path("rss/feed.xml").write_text(rss)
```

`gh release view`が返すasset URLはCDN直リンクのため、SpotifyのRSSバリデーターがそのまま解決できる。

---

## 実測値：差分0件スキップ時90秒・全件生成時8分の内訳

| 条件 | ジョブ時間 | 無料枠2,000分の消費 |
|------|-----------|-------------------|
| 全6エピソード新規生成（初回） | 8分12秒 | 0.41% |
| 差分1件のみ生成 | 2分34秒 | 0.13% |
| 差分0件（キャッシュ完全ヒット） | **1分27秒** | **0.07%** |

週1本の新エピソード追加ペースで計算すると、月30回のジョブ消費は約55分（枠の2.75%）に収まる。キャッシュヒット確認はActionsの"Cache audio files"ステップに`Cache restored successfully`と表示されるかどうかで即判断できる。

---

## book.yaml topics 5スラッグ：Zenn公開画面ブロック回避

`topics`未設定のままZenn公開設定画面を開くと「トピックを1つ以上設定してください」でブロックされる。`book.yaml`（`config.yaml`）に以下を追記してからCI経由でプッシュすること。

```yaml
# book.yaml
title: "yt-dlp＋VOICEVOXで記事をSpotify配信・年0円自動化"
summary: "ブログ記事をそのままPodcastエピソードに変換し、GitHub Actions CIで毎朝自動配信する完全ガイド"
topics: ["python", "podcast", "voicevox", "automation", "github-actions"]
published: true
price: 500
```

`github-actions`は正規スラッグとして登録済みで検索流入も期待できる。`zenn-cli`は5スラッグまで受け付けるため、この5本でちょうど上限に達する。

---

## failure()フックにcurl 1行でSlack通知を追加

ジョブ末尾にステップを1つ追加するだけで障害検知できる。

```yaml
      - name: Notify Slack on failure
        if: failure()
        run: |
          curl -s -X POST "${{ secrets.SLACK_WEBHOOK_URL }}" \
            -H "Content-Type: application/json" \
            -d "{\"text\":\"Podcast CI failed — ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}\"}"
```

`SLACK_WEBHOOK_URL`はリポジトリの`Settings > Secrets > Actions`に登録する。`if: failure()`は音声生成・アップロード・RSS更新のどの段階で落ちても発火するため、障害箇所をActionsの実行URLを踏むだけで特定できる。
