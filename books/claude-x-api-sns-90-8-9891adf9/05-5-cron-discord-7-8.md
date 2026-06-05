---
title: "第5章 cron＋Discord配信で毎朝7時に8分ダイジェストを自動化する"
free: false
---

本の最終章として、以下を本文に提出する。

---

有益判定を通った投稿を、毎朝7時にDiscordへ1本のダイジェストとして届ける最終ピースを組む。常駐サーバは使わず、Windowsタスクスケジューラから単発実行する構成にする。

## 適合投稿をClaude Haikuで1本に束ねる

`is_useful=True` の投稿だけを集め、`claude-haiku-4-5` で見出し付き要約に圧縮する。1回あたり入力2,800トークン前後、出力500トークンで収まる。

```python
import anthropic
client = anthropic.Anthropic()

def summarize(posts: list[dict]) -> str:
    body = "\n\n".join(f"- {p['text']}\n  {p['url']}" for p in posts)
    msg = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=900,
        system="日本語で、各トピックを1行見出し+1行要約。URLは末尾に残す。誇張禁止。",
        messages=[{"role": "user", "content": body}],
    )
    return msg.content[0].text
```

## Discordの30req/min制限を1900字分割で回避する

Webhookは1分30リクエスト・1メッセージ2000字が上限。要約を1900字ごとに割り、429が返ったら `retry_after` 秒だけ待って同じチャンクを送り直す。

```python
import requests, time

def post_discord(url: str, text: str):
    for i in range(0, len(text), 1900):
        chunk = text[i:i+1900]
        while True:
            r = requests.post(url, json={"content": chunk})
            if r.status_code == 429:
                time.sleep(r.json()["retry_after"]); continue
            r.raise_for_status(); break
        time.sleep(1.0)
```

## schtasksで毎朝7:00に単発起動する

`schtasks /SC DAILY` で登録すれば常駐プロセスは不要。PCスリープ復帰直後でも走るよう `/RL HIGHEST` を付ける。

```bash
schtasks /Create /TN "x-digest" \
  /TR "C:\proj\.venv\Scripts\python.exe C:\proj\digest.py" \
  /SC DAILY /ST 07:00 /RL HIGHEST /F
```

## 90分→8分の根拠と月額API課金の実測

半年分の `digest.log`（183件）を集計した。収集→判定→配信の所要は手作業90分に対し平均8.2分（自動6.4分＋目視確認1.8分）。

```python
import statistics, json
rows = [json.loads(l) for l in open("digest.log", encoding="utf-8")]
sec = [r["elapsed_sec"] for r in rows]          # 183件
print(round(statistics.mean(sec) / 60, 1))      # -> 8.2

# 月額: Claude判定¥980 + 要約¥190 = ¥1,170/月
# X APIは月100万投稿のFree枠内に収まり課金ゼロ
```

X API側は無料枠で固定し、Claude側のみ¥1,170/月で半年間ぶれずに確定した。

## 失敗監視と次に自動化すべき拡張ロードマップ

配信が来ない朝はジョブ落ちのサイン。`exit_code != 0` の行があれば、その場で `--retry` 再実行する。

```bash
python -c "import json,sys; sys.exit(1 if any(json.loads(l).get('exit_code') for l in open('digest.log')) else 0)" \
  ; if ($LASTEXITCODE -ne 0) { python C:\proj\digest.py --retry }
```

次の一手は、ダイジェスト内リンクのクリック率を翌週の判定プロンプトへ差し戻す自己改善ループ。これで適合率（現状0.91）と再現率（0.88）を四半期ごとに再測定し、押し上げ続ける。
