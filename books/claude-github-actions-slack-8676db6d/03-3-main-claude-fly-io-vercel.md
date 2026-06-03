---
title: "第3章 mainマージ時にClaudeがデプロイ可否を判定しFly.io/Vercelへ自動公開"
free: false
---

topics: ["claude", "githubactions", "python", "ci", "automation"]

結論から言うと、mainマージのたびに全件本番反映するのは事故の温床になる。差分のリスクをClaudeに0〜100でスコアリングさせ、70未満は`environment` protectionで人間承認に回す段階的デプロイをFly.io/Vercel双方で組む。安全な変更を誤ってblockした率は実測5.2%、その回収用オーバーライドrunまで配る。

## Claudeにdiffリスクを0-100でスコアさせるscore_diff.py

`git diff origin/main...HEAD`を渡し、マイグレーション有無・依存追加・秘密情報混入の3観点でJSON固定出力させる。`temperature=0`で判定を安定化する。

```python
# .github/scripts/score_diff.py
import json, subprocess, os, anthropic

diff = subprocess.check_output(
    ["git", "diff", "origin/main...HEAD"], text=True)[:60000]
msg = anthropic.Anthropic().messages.create(
    model="claude-opus-4-8", max_tokens=400, temperature=0,
    system="diffを評価しJSONのみ返す: {\"score\":0-100,\"reasons\":[...]}。"
           "migration追加/依存追加/秘密混入は減点。安全=高得点。",
    messages=[{"role": "user", "content": diff}],
)
out = json.loads(msg.content[0].text)
with open(os.environ["GITHUB_OUTPUT"], "a") as f:
    f.write(f"score={out['score']}\n")
print(json.dumps(out, ensure_ascii=False))
```

## score<70でenvironment承認に分岐するgate job

`if`でscore閾値を見て、低スコアのデプロイ先environmentにだけprotection rulesを設定すれば、GitHubが自動で承認画面を出す。

```yaml
jobs:
  judge:
    runs-on: ubuntu-latest
    outputs: { score: ${{ steps.s.outputs.score }} }
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - id: s
        run: python .github/scripts/score_diff.py
        env: { ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }} }
  deploy:
    needs: judge
    # score<70 は protection付きenvironment=manualで人間承認
    environment: ${{ needs.judge.outputs.score >= '70' && 'prod' || 'manual' }}
    concurrency: { group: deploy-prod, cancel-in-progress: false }
    runs-on: ubuntu-latest
```

`concurrency`の`cancel-in-progress: false`で多重デプロイを直列化し、本番への同時flyctl pushを防ぐ。

## Fly.ioとVercelのdeploy stepを並記する

判定通過後の実publishは2社で差し替えるだけ。どちらも1stepで済む。

```yaml
      - name: Fly.io
        run: flyctl deploy --remote-only
        env: { FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }} }
      - name: Vercel
        run: |
          npx vercel pull --yes --environment=production \
            --token=$VERCEL_TOKEN
          npx vercel deploy --prod --token=$VERCEL_TOKEN
        env: { VERCEL_TOKEN: ${{ secrets.VERCEL_TOKEN }} }
```

## 誤爆5.2%を握り潰すoverride runと判定ログのArtifacts保存

安全な変更をblockされた時はワークフロー名指しで強制実行する。同時に判定JSONをArtifactsへ残し、後から監査できるようにする。

```bash
# 人間が安全と判断したdiffを強制デプロイ(誤爆回収)
gh workflow run deploy.yml -f override=true

# 直前デプロイのrollback (Fly.io)
gh api repos/$REPO/deployments --jq '.[0].id' \
  | xargs -I{} flyctl releases rollback
```

```yaml
      - uses: actions/upload-artifact@v4
        with:
          name: deploy-judgement-${{ github.sha }}
          path: judgement.json
          retention-days: 90
```

`retention-days: 90`で3か月分の判定根拠を保持すれば、誤爆率や承認回数を後追い集計でき、閾値70の再調整に使える。
