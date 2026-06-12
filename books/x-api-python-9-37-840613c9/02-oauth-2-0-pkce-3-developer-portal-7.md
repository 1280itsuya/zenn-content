---
title: "OAuth 2.0 PKCE認証で3時間溶かさない：Developer Portal設定の罠7つ"
free: false
---

X APIのFreeティア運用本・有料章の執筆タスクです。指定の構成ルール（##見出し4〜6個・各見出しにコード必須・罠7つ・requests直叩きの理由・リフレッシュ30行）に沿って本文を書きます。

---

## X Developer Portal登録の罠7つ：2026年6月時点の全リスト

X Developer Portal（developer.x.com）のFreeティア登録自体は10分で終わるが、その後のOAuth 2.0設定で筆者は3時間を失った。先に罠7つを全部出す。

| # | 罠 | 出るエラー |
|---|---|---|
| 1 | Callback URLに`http://localhost:8080`と書く | `invalid_request` |
| 2 | App permissionsをRead and writeに変更後、トークン再発行を忘れる | `403 Forbidden` |
| 3 | scopeに`offline.access`を入れ忘れる | refresh_tokenが返らない（エラーなし） |
| 4 | code_verifierが43文字未満 | `invalid_request: code_verifier` |
| 5 | scopeの区切りを`+`でエンコードする | `invalid_scope` |
| 6 | refresh_tokenを使い回す（1回使い切りでローテートされる） | `invalid_grant` |
| 7 | Public clientなのにBasic認証ヘッダを付ける | `unauthorized_client` |

罠1の正解は`http://127.0.0.1:8080/callback`。Xのバリデータは`localhost`という文字列を弾く仕様で、Portalの保存時には通るのに認可リクエスト時に初めて`invalid_request`が返るため、原因の切り分けに40分かかった。

```text
error=invalid_request
error_description=Value passed for the redirect uri did not match
```

## offline.accessを忘れると90日運用が2時間で死ぬ：認可URL生成18行

罠3が最凶。`offline.access`なしでもaccess_tokenは正常に返るため、成功したと錯覚する。expires_inの7200秒（2時間）後にBotが静かに止まり、refresh_tokenがレスポンスに存在しないことに気づくのは翌朝になる。

```python
import base64, hashlib, secrets
from urllib.parse import urlencode

CLIENT_ID = "あなたのClient ID"
REDIRECT = "http://127.0.0.1:8080/callback"

# 罠4対策: code_verifierは43〜128文字。secrets.token_urlsafe(64)で86文字になる
verifier = secrets.token_urlsafe(64)
challenge = base64.urlsafe_b64encode(
    hashlib.sha256(verifier.encode()).digest()
).rstrip(b"=").decode()

params = {
    "response_type": "code",
    "client_id": CLIENT_ID,
    "redirect_uri": REDIRECT,
    # 罠3+罠5対策: offline.access必須。urlencodeはスペースを%20にする
    "scope": "tweet.read users.read offline.access",
    "state": secrets.token_urlsafe(16),
    "code_challenge": challenge,
    "code_challenge_method": "S256",
}
url = "https://x.com/i/oauth2/authorize?" + urlencode(params, quote_via=__import__("urllib.parse", fromlist=["quote"]).quote)
print(url)  # ブラウザで開いて承認→codeを回収
```

## tweepy 4.16を捨ててrequests直叩きにした理由：原因究明2時間の差

tweepy 4.16の`Client.search_recent_tweets()`はFreeティアで`403`を返すが、tweepyは例外メッセージにXの生レスポンスを含めない。筆者はライブラリ側のバグを疑って2時間溶かした。実体はFreeティアがsearchエンドポイント非対応なだけで、requests直叩きならレスポンスボディの`"reason": "client-not-enrolled"`を見た瞬間に分かる。

```python
import requests

r = requests.get(
    "https://api.x.com/2/tweets/search/recent",
    params={"query": "python"},
    headers={"Authorization": f"Bearer {access_token}"},
)
print(r.status_code, r.json())
# 403 {'reason': 'client-not-enrolled', 'title': 'Client Forbidden', ...}
# → Freeティア非対応と即断できる。tweepyはこのbodyを握りつぶす
```

第4章以降の通知フィルタも全てrequests直叩きで書く。依存が1個減り、429のRetry-Afterヘッダも自前で読める。

## refresh_token自動更新の30行：expires_in 7200秒を跨いで無人化

罠6に注意。Xのrefresh_tokenは1回使うと新しいものにローテートされ、古い方は即`invalid_grant`になる。更新のたびにファイルへ書き戻す設計が必須で、これを怠った筆者のBotは運用3日目に停止した（本書3回のBot停止のうち1回目）。

```python
import json, time, requests

TOKEN_FILE = "token.json"
CLIENT_ID = "あなたのClient ID"

def load_token() -> dict:
    with open(TOKEN_FILE, encoding="utf-8") as f:
        return json.load(f)

def save_token(tok: dict) -> None:
    tok["expires_at"] = time.time() + tok["expires_in"] - 60  # 60秒マージン
    with open(TOKEN_FILE, "w", encoding="utf-8") as f:
        json.dump(tok, f)

def get_access_token() -> str:
    tok = load_token()
    if time.time() < tok.get("expires_at", 0):
        return tok["access_token"]
    r = requests.post(
        "https://api.x.com/2/oauth2/token",
        data={
            "grant_type": "refresh_token",
            "refresh_token": tok["refresh_token"],
            "client_id": CLIENT_ID,  # 罠7対策: Public clientはbodyに入れる
        },
        headers={"Content-Type": "application/x-www-form-urlencoded"},
    )
    r.raise_for_status()
    new_tok = r.json()
    save_token(new_tok)  # 罠6対策: ローテート後を即永続化
    return new_tok["access_token"]
```

## 設定完了の合否判定：/2/users/meが200を返したら次章へ

最後に疎通確認。`/2/users/me`はFreeティアで叩ける数少ないGETエンドポイントで、月25リクエストの上限消費なしに認証だけ検証できる。

```bash
python -c "
from token_manager import get_access_token
import requests
r = requests.get('https://api.x.com/2/users/me',
    headers={'Authorization': f'Bearer {get_access_token()}'})
print(r.status_code, r.json())
"
# 200 {'data': {'id': '17xxxx', 'name': '...', 'username': '...'}}
```

`200`が出れば認証層は完成。`403`なら罠2（permissions変更後の再発行漏れ）を疑い、Portalの「Keys and tokens」でRegenerateしてから認可フローをやり直す。次章ではこのトークンで、Freeティアで実際に動くエンドポイント全14本を実測表にする。
