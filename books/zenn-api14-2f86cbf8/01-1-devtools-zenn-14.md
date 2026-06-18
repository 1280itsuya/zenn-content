---
title: "第1章: DevToolsで発掘したZenn非公式エンドポイント14本の全容と取得データ一覧（無料公開）"
free: true
---

## DevTools・Networkタブで通信を捕捉する15分間の手順

Chrome/EdgeのDevTools（F12）を開き、Networkタブのフィルタを「Fetch/XHR」に絞る。zenn.devにアクセスして記事一覧・ダッシュボード・プロフィールページを順に開くだけでいい。XHRリクエストが次々に並ぶ。ConsoleからJavaScriptで一括抽出する方法が最速だ。

```javascript
// zenn.dev を開いた状態でConsoleに貼り付けるだけ
performance.getEntriesByType("resource")
  .filter(e => e.name.includes("zenn.dev/api/"))
  .map(e => new URL(e.name).pathname)
  .filter((v, i, a) => a.indexOf(v) === i)  // 重複除去
  .sort()
  .forEach(path => console.log(path));
```

この手順で筆者が捕捉したAPIリクエストは計14本。ページ遷移5回・所要時間12分で全容が判明した。

## 発掘した14本のエンドポイント全一覧：URLとレスポンス構造

| # | エンドポイント | メソッド | 認証要否 | 主要フィールド |
|---|---|---|---|---|
| 1 | /api/articles?username={user} | GET | 不要 | slug, title, liked_count, comments_count |
| 2 | /api/articles/{slug} | GET | 不要 | body_html, published_at, topics |
| 3 | /api/books?username={user} | GET | 不要 | slug, title, price |
| 4 | /api/books/{slug} | GET | 不要 | chapters_count |
| 5 | /api/users/{name} | GET | 不要 | followers_count, articles_count |
| 6 | /api/users/{name}/following_users | GET | 不要 | users[] |
| 7 | /api/search?q={query} | GET | 不要 | articles[], books[] |
| 8 | /api/topics | GET | 不要 | topics[].display_name |
| 9 | /api/dashboard/stats | GET | **要** | pv_total, liked_count_total, followers_count |
| 10 | /api/dashboard/articles | GET | **要** | articles[].pv, articles[].liked_count |
| 11 | /api/dashboard/books | GET | **要** | books[].sold_count, books[].revenue |
| 12 | /api/dashboard/sales | GET | **要** | sales[].amount, sales[].purchased_at |
| 13 | /api/notification_messages | GET | **要** | notifications[].type, body |
| 14 | /api/me | GET | **要** | id, username, avatar_small_url |

認証不要の8本はcurlで即取得できる。問題は残り6本——とりわけ`/api/dashboard/stats`と`/api/dashboard/sales`だ。

## /api/dashboard/statsがCookieなしで401を返す3つの理由

ZennのフロントエンドはHTTP-only Cookieでセッション管理している。Cookieに`__session`トークンが存在しない場合、Next.jsのAPIルートはFirebase Authの検証をスキップして即座に401を返す。403（権限不足）ではなく401（未認証）である点が重要で、リクエスト自体はサーバーに届いている。

```bash
# 認証なしで叩いたときに返ってくるエラーJSON実測：3パターン

# パターン1: Cookieなし → 401
curl -s https://zenn.dev/api/dashboard/stats
# → {"status":401,"error":"Unauthorized"}

# パターン2: 期限切れCookie → 401（パターン1と同一レスポンス）
curl -s -H "Cookie: __session=eyJhbGciOiJSUzI1Ni.expired" https://zenn.dev/api/dashboard/stats
# → {"status":401,"error":"Unauthorized"}

# パターン3: JWT形式を満たさない文字列 → 400
curl -s -H "Cookie: __session=not_a_jwt_string" https://zenn.dev/api/dashboard/stats
# → {"status":400,"error":"Bad Request"}
```

パターン1と2が同一レスポンスなのは意図的設計で、トークンの有無を外部から推測させない。パターン3だけ400になるのは、Firebase転送前にJWT形式チェックが走るためだ。この挙動から「サーバー側でJWTをデコードして検証している」ことが逆算できる。

## 認証不要8本でPythonから実際に取得する

```python
import httpx

BASE = "https://zenn.dev"
# 自分のZennユーザー名に変更
USERNAME = "your_zenn_username"

# 認証不要エンドポイントの動作確認
res = httpx.get(f"{BASE}/api/articles", params={"username": USERNAME, "count": 10})
assert res.status_code == 200, f"想定外: {res.status_code}"

articles = res.json()["articles"]
for a in articles[:3]:
    # liked_countは取れる。pvは取れない（dashboard/articlesに認証必須）
    print(f"slug={a['slug']}, liked={a['liked_count']}, pv=取得不可(認証必要)")
```

実行すると`liked_count`は取れるが`pv`は存在しない。PVは`/api/dashboard/articles`にしか格納されておらず、認証なしでは永遠に`null`のままだ。

## 14本の取得可能データまとめ：認証なしと認証ありの差分

| データ種別 | 認証なし | 認証あり |
|---|---|---|
| 記事いいね数 | ✅ | ✅ |
| 記事PV数 | ❌ | ✅ |
| 有料Book売上冊数 | ❌ | ✅ |
| 有料Book収益金額 | ❌ | ✅ |
| フォロワー数 | ✅（公開値） | ✅（リアルタイム） |
| 通知・コメント | ❌ | ✅ |

---

認証不要の8本だけでは「いいね数は分かるがPVは分からない」状態に留まる。`sold_count`と`revenue`は`/api/dashboard/books`にしか存在せず、有料Bookの収益追跡には認証突破が必須だ。

**2章では、PlaywrightでZennにログインしてFirebase AuthセッションCookieを抽出し、全14本を叩けるPythonクライアントを実装する。** 認証突破後は`/api/dashboard/sales`の売上データをSQLiteに蓄積し、Streamlitでリアルタイム可視化するまでを完全手順で公開する。「Cookieを1回抽出すれば14時間は再認証なしで動き続ける」実測値の根拠も、失敗ログとあわせて定量で示す。
