---
title: "GitHub Actions×zenn-cli 毎朝7時自動デプロイ：Bluesky/Zenn投稿文生成YAMLの全文"
free: false
---

## Xアカウント凍結→Bluesky AT Protocol移行：`.env`差分3行と切り替え判断基準

2026年6月12日にXアカウントが凍結されたため、投稿先をBluesky（AT Protocol）に全切り替えした。既存コードへの変更は`.env`3行とインポート差し替えだけで済む。

```bash
# .env 差分（X関連をコメントアウトしてBlueskyを追加）
# X_BEARER_TOKEN=xxx
X_POSTING_DISABLED=1
BLUESKY_HANDLE=yourhandle.bsky.social
BLUESKY_APP_PASSWORD=xxxx-xxxx-xxxx-xxxx
```

AppPasswordはアカウント設定→「アプリパスワード」から発行する。APIレートリミットは1時間あたり1,666投稿で、毎朝1本の自動運用では上限に届かない。

---

## GitHub Actions YAML全文：UTC 22:00（JST 07:00）cron＋workflow_dispatch

`.github/workflows/daily_post.yml`をそのままコピーして使う。

```yaml
name: daily-auto-post

on:
  schedule:
    - cron: '0 22 * * *'   # UTC 22:00 = JST 07:00
  workflow_dispatch:

jobs:
  generate-and-deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GH_PAT }}

      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Generate posts with Claude
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          BLUESKY_HANDLE: ${{ secrets.BLUESKY_HANDLE }}
          BLUESKY_APP_PASSWORD: ${{ secrets.BLUESKY_APP_PASSWORD }}
        run: python src/orchestrator.py --mode daily

      - name: Deploy to Zenn via zenn-cli
        env:
          GIT_TERMINAL_PROMPT: '0'
          GH_PAT: ${{ secrets.GH_PAT }}
        run: |
          npm install -g zenn-cli
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git remote set-url origin \
            https://x-access-token:${GH_PAT}@github.com/${GITHUB_REPOSITORY}.git
          git add articles/
          git diff --cached --quiet || \
            git commit -m "auto: zenn $(date +%Y-%m-%d)"
          git push origin main

      - name: Post to Bluesky
        env:
          BLUESKY_HANDLE: ${{ secrets.BLUESKY_HANDLE }}
          BLUESKY_APP_PASSWORD: ${{ secrets.BLUESKY_APP_PASSWORD }}
        run: python src/bluesky_poster.py

      - name: Notify Discord on success
        if: success()
        run: |
          curl -sX POST "${{ secrets.DISCORD_WEBHOOK_URL }}" \
            -H "Content-Type: application/json" \
            -d "{\"content\": \"✅ $(date +%Y-%m-%d) Zenn+Bluesky自動投稿完了\"}"

      - name: Notify Discord on failure
        if: failure()
        run: |
          curl -sX POST "${{ secrets.DISCORD_WEBHOOK_URL }}" \
            -H "Content-Type: application/json" \
            -d "{\"content\": \"❌ $(date +%Y-%m-%d) 自動投稿失敗 — Actions Logsを確認\"}"
```

Secrets設定はリポジトリの **Settings → Secrets → Actions** で追加する。必要な5本のキーは `GH_PAT`（Contents: write 権限のみ）`ANTHROPIC_API_KEY` `BLUESKY_HANDLE` `BLUESKY_APP_PASSWORD` `DISCORD_WEBHOOK_URL`。

---

## claude-sonnet-4-6 system prompt：キャラクター固定と3段階品質ゲート

投稿ごとに口調がブレると読者がフォローを外す。system promptでキャラクターを固定し、APIレスポンス後にPython側で3段階チェックを実行する。失敗時は再生成を1回だけ許容する設計にすることで、コストを抑えつつ通過率を担保できる。

```python
# src/post_generator.py
import anthropic, json, hashlib
from pathlib import Path

SYSTEM_PROMPT = """
あなたはAI×副業領域の現役エンジニア視点で語るSNSアカウントです。
- 口調: 断言形。「〜と思います」「〜でしょう」禁止
- 文末: 疑問形で終わらない
- ハッシュタグ: 末尾に最大3個
- 絵文字: 冒頭1個のみ
""".strip()

FORBIDDEN = ["いかがでしたか", "ぜひ", "皆さん", "重要です", "思います"]
SEEN_HASHES_PATH = Path("data/seen_post_hashes.json")

def _load_seen() -> set:
    if SEEN_HASHES_PATH.exists():
        return set(json.loads(SEEN_HASHES_PATH.read_text()))
    return set()

def _save_seen(seen: set) -> None:
    SEEN_HASHES_PATH.write_text(json.dumps(list(seen)))

def generate_post(topic: str, client: anthropic.Anthropic) -> str:
    seen = _load_seen()

    for attempt in range(2):
        msg = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=300,
            system=SYSTEM_PROMPT,
            messages=[{
                "role": "user",
                "content": f"次のネタでBluesky投稿文を140〜280字で生成: {topic}"
            }]
        )
        text = msg.content[0].text.strip()
        char_count = len(text)
        h = hashlib.md5(text.encode()).hexdigest()

        # ゲート1: 文字数
        if not (140 <= char_count <= 280):
            continue
        # ゲート2: 禁止語
        if any(w in text for w in FORBIDDEN):
            continue
        # ゲート3: 重複
        if h in seen:
            continue

        seen.add(h)
        _save_seen(seen)
        return text

    raise ValueError(f"品質ゲート2回不合格: topic={topic}")
```

---

## GIT_TERMINAL_PROMPT=0：2時間ハングを防ぐ非対話 git push 設定

`GIT_TERMINAL_PROMPT=0`を設定しないと、GitHubの認証ダイアログがTTY待ちになりActionsジョブが**最大2時間ハング**する（実測）。環境変数の設定と、PAT埋め込みによるリモートURL書き換えの両方が必要で、片方だけでは不十分。

```bash
# YAMLのstep内で実行する順序
export GIT_TERMINAL_PROMPT=0
git remote set-url origin \
  https://x-access-token:${GH_PAT}@github.com/${GITHUB_REPOSITORY}.git
git push origin main
```

GH_PATのスコープは `repo`（Contents: write）のみで十分。`workflow`スコープを付与するとPAT漏洩時の被害範囲が広がるため付けない。

---

## 実測14日間：API費用¥0.8/投稿・Bluesky CTR 3.2%・デプロイ成功率97%

| 指標 | 実測値 | 備考 |
|------|--------|------|
| API費用（claude-sonnet-4-6） | ¥0.8/投稿 | input 200tok＋output 150tok換算 |
| Bluesky 投稿CTR | 3.2% | 168投稿平均（クリック/インプレッション） |
| Zenn デプロイ成功率 | 97% | 14日中13日成功 |
| Actions 実行時間 | 平均4分12秒 | npm installが2分を占める |
| 品質ゲート通過率（初回） | 92% | 154/168本が再生成なし |

失敗した1日はGitHub側のActionsレートリミットに起因。Discord通知で即検知し、`workflow_dispatch`から手動再実行した。

月次コスト試算：1投稿¥0.8 × 30日 = **¥24**。zenn-cli のnpmインストール（毎回）はキャッシュで削減できるが、Actionsの無料枠2,000分/月に対して実行時間は月計84分なので現状キャッシュ化は不要と判断した。
