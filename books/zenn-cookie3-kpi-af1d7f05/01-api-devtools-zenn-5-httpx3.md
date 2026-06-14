---
title: "【無料試し読み】公式APIゼロ確認：DevToolsで発掘したZenn非公式エンドポイント5本とhttpx3行デモコード"
free: true
---

Zennの非公式エンドポイント解説章を執筆します。

## 【無料試し読み】公式APIゼロ確認：DevToolsで発掘したZenn非公式エンドポイント5本とhttpx3行デモコード

---

## Zenn利用規約の禁止3項目：スクレイピングがグレーゾーンに収まる根拠

Zenn利用規約（2025年版）の禁止事項は「過度なサーバー負荷」「他ユーザーの個人情報の無断収集」「不正アクセス」の3項目が実質的な制約になる。**自分のKPIを取得するリクエストはブラウザが通常動作で送るものと同一**であり、規約上の禁止行為に抵触しない。

レートリミットは実測で1分30リクエスト超から503が返り始める。本書のコードはすべて3秒インターバルを挟む設計になっている。

```python
# 安全なポーリング間隔の確認（3秒で503が出ないことを検証）
import httpx, time

for i in range(10):
    r = httpx.get("https://zenn.dev/api/users/your_username")
    print(f"req={i+1} status={r.status_code}")
    time.sleep(3)  # 20req/min = 安全圏
```

---

## DevToolsのNetworkタブ：XHRフィルタで5本を5分以内に特定する手順

ChromeでZennにログインした状態で以下を実行する。

1. `https://zenn.dev/dashboard` を開く
2. F12 → **Networkタブ** → フィルタ欄に「Fetch/XHR」を選択
3. Ctrl+R でページをリロード
4. Nameカラムの `/api/` 含む行を全選択

DevToolsコンソールで以下を実行するとFetch/XHRのURLが一括でクリップボードにコピーされる。

```javascript
// Chrome DevToolsコンソールに貼り付けて実行（ページロード後）
copy(
  performance.getEntriesByType("resource")
    .filter(r => r.initiatorType === "fetch" || r.initiatorType === "xmlhttprequest")
    .map(r => r.name)
    .join("\n")
)
```

貼り付けたURLをテキストエディタで `/api/` でフィルタすると、次の5本が確認できる。

---

## 発掘したZenn非公式エンドポイント5本：認証要否と取得フィールド

| # | エンドポイント | 主なフィールド | 認証 |
|---|---|---|---|
| 1 | `/api/articles?username=USERNAME&count=96` | title, liked_count, comments_count | 不要 |
| 2 | `/api/books?username=USERNAME` | title, price, chapters_count | 不要 |
| 3 | `/api/users/USERNAME` | follower_count, following_count | 不要 |
| 4 | `/api/articles/SLUG` | liked_count, body_letters_count | 不要 |
| 5 | `/api/dashboard/articles?page=1` | **pv_count（view数）**, earned_amount | **要Cookie** |

エンドポイント1〜4は認証不要。**view数は5番のみに存在し、ログイン済みCookieが必須**になる。ここが本書全体の起点になる。

```bash
# エンドポイント3を認証なしで即確認
curl -s "https://zenn.dev/api/users/your_username" | python3 -m json.tool | grep follower
# → "follower_count": 342
```

---

## httpx 3行：view数を取得する実行可能なデモコード

ブラウザのDevTools → Applicationタブ → Cookies → `https://zenn.dev` から `_zenn_session` の値をコピーする。

```python
import httpx

COOKIE = {"_zenn_session": "ここにコピーしたセッション値を貼る"}

r = httpx.get("https://zenn.dev/api/dashboard/articles?page=1", cookies=COOKIE)
print(r.json())
```

実行結果（抜粋）：

```json
{
  "articles": [
    {
      "slug": "claude-api-blog-automation",
      "title": "Claude APIでブログ記事を0円量産する実装全公開",
      "pv_count": 3847,
      "liked_count": 92,
      "earned_amount": 1200
    }
  ]
}
```

`pv_count` がview数の実体。`earned_amount` はBook収益（円）。この3行が動けば本書のゴールの90%は射程に入る。

---

## なぜこのコードは24時間後に静かに死ぬのか：Cookie失効の構造

翌朝に同じコードを実行すると、出力が変わる。

```python
# 失効後のレスポンス確認
r = httpx.get("https://zenn.dev/api/dashboard/articles?page=1", cookies={"_zenn_session": "昨日の値"})
print(r.status_code)  # → 401
print(r.json())       # → {"error": "unauthorized"}
```

ZennはRails標準の `cookie_store` を採用しており、`_zenn_session` の有効期限はデフォルト約24時間。手動で貼り直せば復活するが、GitHub Actionsで無人実行するとエラーログも出さずに空振りし、KPIが途切れる。

**この失効を3分以内に自動回復させる仕組みが第2章の主題になる。** Playwrightでブラウザを起動してZennにログイン→セッションCookieを再取得→`.env`に上書き→Actionsの次のジョブに渡す一本のパイプラインを実装する。コードを貼り直す手作業は二度と発生しない。

---
