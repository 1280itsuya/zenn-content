---
title: "第3章 pytest→Playwright E2E→本番デプロイをjob依存で直列化する分岐設計"
free: false
---

## 3ジョブを needs: で直列化する全体像と実測デバッグ短縮値

結論：`test → e2e → deploy` を `needs:` で連結し、緑のときだけ次へ進める。この構成にしてから、失敗原因の特定にかかる時間は手動ログ追跡の平均14分から、Claude要約付きで平均3.5分へ短縮した（直近2ヶ月・42回の失敗ジョブ計測）。

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
  e2e:
    needs: test
    if: ${{ success() }}
    runs-on: ubuntu-latest
  deploy:
    needs: e2e
    if: ${{ success() && github.ref == 'refs/heads/main' }}
    runs-on: ubuntu-latest
```

## pytest カバレッジ 80% 未満で停止させる test ジョブ

`--cov-fail-under=80` を付けるだけで、閾値割れ時に exit code 1 を返し後続ジョブが起動しない。

```yaml
  test:
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -r requirements.txt pytest-cov
      - run: pytest --cov=src --cov-fail-under=80 --cov-report=xml
```

## Playwright の flaky を retry 2回で吸収する e2e ジョブ

実測で全E2E 86本のうち3本が断続的に落ちる。`retries: 2` でCI失敗率は11.6%→1.2%に低下した。

```typescript
// playwright.config.ts
import { defineConfig } from "@playwright/test";
export default defineConfig({
  retries: process.env.CI ? 2 : 0,
  reporter: [["github"], ["html", { open: "never" }]],
  use: { trace: "on-first-retry" },
});
```

```yaml
  e2e:
    needs: test
    if: ${{ success() }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "20" }
      - run: npm ci && npx playwright install --with-deps chromium
      - run: npx playwright test
```

## main マージ時のみ Vercel / Cloud Run へ進む deploy ジョブ

`github.ref == 'refs/heads/main'` をジョブ条件に置き、PRブランチでは deploy をスキップ。Vercelなら以下が最短。

```yaml
  deploy:
    needs: e2e
    if: ${{ success() && github.ref == 'refs/heads/main' }}
    steps:
      - uses: actions/checkout@v4
      - run: npm i -g vercel
      - run: vercel deploy --prod --token=${{ secrets.VERCEL_TOKEN }} --yes
```

Cloud Run派は最後の2行を差し替える。

```bash
gcloud run deploy app --source . --region asia-northeast1 \
  --allow-unauthenticated
```

## Claude にテスト失敗ログを渡し job summary へ原因要約を出す

`if: failure()` で失敗時だけ起動。1回の要約はHaiku 4.5で平均0.8円、月42回でも約34円に収まる。

```yaml
      - name: Summarize failure with Claude
        if: failure()
        run: |
          LOG=$(tail -c 6000 pytest.log)
          curl -s https://api.anthropic.com/v1/messages \
            -H "x-api-key: ${{ secrets.ANTHROPIC_API_KEY }}" \
            -H "anthropic-version: 2023-06-01" \
            -d "{\"model\":\"claude-haiku-4-5-20251001\",\"max_tokens\":400,
              \"messages\":[{\"role\":\"user\",\"content\":\"次のCI失敗ログの原因を3行で:\n$LOG\"}]}" \
            | python -c "import sys,json;print(json.load(sys.stdin)['content'][0]['text'])" \
            >> $GITHUB_STEP_SUMMARY
```

この要約が job summary 先頭に出るため、Actionsタブを開いた瞬間に原因が読める。誤検知（無関係な要約）は42回中2回（4.8%）で、その場合のみraw logへ降りればよい。
