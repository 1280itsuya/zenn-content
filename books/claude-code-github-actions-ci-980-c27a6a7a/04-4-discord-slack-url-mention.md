---
title: "第4章 Discord/Slack通知とデプロイURL自動貼付｜失敗時だけ@mentionする制御"
free: false
---

## 失敗時のみ`@mention`する条件分岐をjob outputで実装する

通知は「成功はサイレント、失敗だけ`<@USER_ID>`を付ける」が開封率を保つ唯一の正解だ。GitHub Actionsの`needs.<job>.result`を参照し、`success`なら無言、それ以外でメンション文字列を生成する。

```yaml
notify:
  needs: [build, deploy]
  if: always()
  runs-on: ubuntu-latest
  steps:
    - name: Build mention
      id: m
      run: |
        if [ "${{ needs.deploy.result }}" = "success" ]; then
          echo "mention=" >> "$GITHUB_OUTPUT"
        else
          echo "mention=<@${{ vars.ONCALL_DISCORD_ID }}> デプロイ失敗" >> "$GITHUB_OUTPUT"
        fi
```

`if: always()`を外すと失敗ジョブで通知自体が走らないため必須だ。

## Claudeに変更点を3行要約させ通知本文へ埋め込む

`anthropic`のクライアントへ`git diff --stat`を渡し、`claude-haiku-4-5`で3行に圧縮する。Haiku採用で1通知あたり約$0.0004、月300通知でも$0.12に収まる。

```python
import subprocess, anthropic
diff = subprocess.check_output(["git", "diff", "HEAD~1", "--stat"]).decode()
msg = anthropic.Anthropic().messages.create(
    model="claude-haiku-4-5-20251001",
    max_tokens=180,
    messages=[{"role": "user",
        "content": f"次のdiffを日本語3行・箇条書きで要約:\n{diff}"}],
)
print(msg.content[0].text)
```

## デプロイURLとコミットdiffリンクをDiscord Webhookへ自動添付する

Discordの`embeds`に`fields`でプレビューURLとdiff比較リンクを2列に並べる。`color`を成功`3066993`(緑)/失敗`15158332`(赤)で出し分け、ミュート中でも一覧で識別できる。

```bash
curl -H "Content-Type: application/json" -d '{
  "content": "'"${MENTION}"'",
  "embeds": [{
    "title": "deploy '"${GITHUB_SHA:0:7}"'",
    "color": '"${COLOR}"',
    "fields": [
      {"name": "Preview", "value": "'"${PREVIEW_URL}"'"},
      {"name": "Diff", "value": "https://github.com/'"${GITHUB_REPOSITORY}"'/commit/'"${GITHUB_SHA}"'"}
    ]
  }]
}' "${DISCORD_WEBHOOK_URL}"
```

## 通知過多で全員ミュートした失敗を重要度別チャンネルで回復させる

運用1ヶ月目は1チャンネルに成功も流し、1日42通知でメンバー6人中5人がミュートした。`#deploy-fail`と`#deploy-ok`へWebhookを分離し、失敗のみ`@mention`に絞った結果、失敗通知の開封(リアクション)率は8%→71%へ回復した。

```yaml
env:
  DISCORD_WEBHOOK_URL: >-
    ${{ needs.deploy.result == 'success'
        && secrets.WEBHOOK_OK
        || secrets.WEBHOOK_FAIL }}
```

成功用Webhookはミュートしてもらってよいチャンネルへ向け、失敗用だけ全員参加にする。

## GitHub Deployments APIでデプロイ状態をPRに逆連携する

Webhook通知と同じURLを`POST /repos/{owner}/{repo}/deployments`の`deployment_status`へ送ると、PR画面のEnvironmentsバッジにプレビューURLが表示され、Slack/Discordを見落としてもPR上で追える。

```bash
curl -X POST \
  -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  "https://api.github.com/repos/${GITHUB_REPOSITORY}/deployments/${DEP_ID}/statuses" \
  -d '{"state":"'"${STATE}"'","environment_url":"'"${PREVIEW_URL}"'","log_url":"'"${RUN_URL}"'"}'
```

`state`は`success`/`failure`/`error`を渡す。これで通知レイヤーとGitHub UIの両方に同一URLが残り、障害時の一次切り分けがチャンネル横断不要になる。
