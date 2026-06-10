---
title: "第5章 poster.py+Crontabで毎朝7時に無人投稿：zenn-cli/WordPress二系統とDiscord通知の運用半年ログ"
free: false
---

毎朝7時、PCの前に座らずに記事が公開される状態をつくる。ただし投稿の最終トリガーだけは品質ゲートを通した後の「人手承認1箇所」を残す——これが半年間でBANゼロを保った設計の核心になる。

## poster.pyでzenn-cli配置とgit pushを自動化する

zenn-cliは`articles/<slug>.md`を置いてpushするだけで公開される。poster.pyは生成済みMarkdownをslug付きで配置し、frontmatterの`published: true`を立ててからコミットする。

```python
import subprocess, pathlib, datetime

def post_zenn(slug: str, body: str, title: str):
    fm = f"---\ntitle: \"{title}\"\nemoji: \"📝\"\ntype: \"tech\"\npublished: true\n---\n\n"
    p = pathlib.Path(f"articles/{slug}.md")
    p.write_text(fm + body, encoding="utf-8")
    msg = f"post: {slug} {datetime.date.today()}"
    subprocess.run(["git", "add", str(p)], check=True)
    subprocess.run(["git", "commit", "-m", msg], check=True)
    subprocess.run(["git", "push", "origin", "main"], check=True)
```

`published: true`を入れ忘れるとpushは成功するのにZennで非公開のまま——これが後述する「静かなスキップ」障害の正体だった。

## WordPress REST APIへ投稿しenvで二系統を切替える

ZennとWordPressは`.env`の有無だけで切り替える。`WP_URL`が未設定ならWP投稿はスキップし、Zennのみ走る。鍵はApplication Passwords（Basic認証）。

```python
import os, requests, base64

def post_wp(title: str, content: str) -> int | None:
    url = os.getenv("WP_URL")
    if not url:           # env未設定なら無効化（半自動の切替点）
        return None
    user, pw = os.getenv("WP_USER"), os.getenv("WP_APP_PASS")
    token = base64.b64encode(f"{user}:{pw}".encode()).decode()
    r = requests.post(
        f"{url}/wp-json/wp/v2/posts",
        headers={"Authorization": f"Basic {token}"},
        json={"title": title, "content": content, "status": "publish"},
        timeout=30,
    )
    r.raise_for_status()
    return r.json()["id"]
```

## Crontabとタスクスケジューラで毎朝7時に起動する

Linuxはcron、Windowsはタスクスケジューラで同一の`run.bat`を叩く。7時固定は、Zennのトレンド入り枠が午前に動くため。

```bash
# Linux: crontab -e
0 7 * * * cd /home/itsuya/auto-money && .venv/bin/python poster.py >> log/post.log 2>&1
```

```powershell
# Windows: タスクスケジューラ登録
schtasks /create /tn "auto-money-7am" /tr "C:\auto-money\run.bat" /sc daily /st 07:00
```

## notifier.pyで投稿数・原価・失敗をDiscordへ通知する

成功・原価・例外を1本のWebhookに集約する。1記事の実原価は中央値3.1円（Claude API分のみ、人件費除く）。

```python
import requests, os

def notify(posted: int, cost_yen: float, errors: list[str]):
    body = f"✅ {posted}本公開 / 原価 ¥{cost_yen:.1f}"
    if errors:
        body += "\n⚠️ " + "\n".join(errors)
    requests.post(os.getenv("DISCORD_WEBHOOK"),
                  json={"content": body}, timeout=10)
```

## 半年運用ログ：累計287本・総原価¥891・障害3件の実数

181日で287本公開、総原価¥891。発生した障害を時系列で報告する。

- **3月：`.claude.json`競合** 対話セッションと7時起動が同時にホームの`.claude.json`へ書込→初回生成破損。対処＝CLI自動回復で完走、本来はセッション外起動を厳守。
- **4月：Qiita API 429** 連続投稿でレート超過。poster側に最低5時間間隔の集中ゲート（`QIITA_MIN_INTERVAL_HOURS`）を実装し解消。
- **5月：Zenn静かなスキップ** `load_dotenv()`欠落でPAT未読込→pushが無言で失敗。明示ロード追加で当日3本を遡及公開。

```bash
# 障害検知: published件数とpush成功数の差分を毎朝照合
git log --since="1 day ago" --oneline | grep -c "post:" || echo "0件=要調査"
```

半年の結論は1行に収束する——**配管は完成、唯一の律速は集客**。287本の累計PVは伸び悩み、原価¥891に対し収益は四桁に届かなかった。次の一手は生成本数の増加ではなく、流入導線（実測view上位テーマへの集中投下と外部シェア文の自動生成）への原資の付け替えになる。半自動パイプラインが渡してくれるのは「時間」であり、その時間を集客に再投資できるかが収益の分岐点になる。
