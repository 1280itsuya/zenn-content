---
title: "第2章 Cookie認証を突破するhttpx probe：__Host-session抽出とセッション失効を検知する実装"
free: false
---

## 結論：`__Host-session` を httpx に注入すれば 401 は消え、200 で統計 JSON が返る

第1章で特定した `https://api.zenn.dev/me/articles?page=1` は認証必須で、Cookie なしの素の `requests.get()` は必ず `401 Unauthorized` を返す。突破口は1つだけ——ブラウザのログインセッション Cookie `__Host-session` を取り出し、`httpx.Client` の `cookies` に渡すことだ。

```python
import httpx

def fetch_stats(session: str) -> dict:
    client = httpx.Client(
        base_url="https://api.zenn.dev",
        cookies={"__Host-session": session},
        headers={"User-Agent": "stat-probe/1.0"},
        timeout=10.0,
    )
    r = client.get("/me/articles", params={"page": 1})
    r.raise_for_status()  # 401/403 をここで例外化
    return r.json()
```

## Chrome DevTools の Application タブから `__Host-session` を1分で抜く

スクレイピングではなく、ブラウザが既に持つ正規 Cookie を再利用する。Zenn にログイン後、`F12` → Application → Cookies → `https://zenn.dev` を開き、`__Host-` 接頭辞の付いた行の Value をコピーする。`__Host-` 接頭辞は RFC 6265bis 準拠で `Secure` かつ `Path=/` 固定のため、ドメインを跨いで漏れにくい。

```bash
# コピーした値を .env に書く（クォート不要）
echo "ZENN_SESSION=eyJhbGciOi...（実際の値）" >> .env
```

## `python-dotenv` で読み、`.gitignore` で平文コミット事故を防ぐ

セッション値はパスワード同等だ。過去に `.env` を `git add .` で巻き込み、`git push` 後に気付いて全セッション失効＋ローテーションに30分溶かした。先に除外設定を入れる。

```bash
printf '.env\n*.session\n__pycache__/\n' >> .gitignore
```

```python
import os
from dotenv import load_dotenv

load_dotenv()
session = os.environ["ZENN_SESSION"]  # 未設定なら KeyError で即停止
```

## セッション失効を `status` と JSON 構造の二重検証で即検知する

最大の罠は、失効時に `200 OK` で空配列 `{"articles": []}` が返り、「いいね0件」と誤認してデータが静かに欠落する事故だ。HTTP ステータスだけでは防げない。期待するキーの存在も同時に検証する。

```python
class SessionExpired(Exception):
    pass

def validate(r: httpx.Response) -> dict:
    if r.status_code in (401, 403):
        raise SessionExpired(f"Cookie失効: {r.status_code}")
    data = r.json()
    # ログイン中なら必ず articles キーを含む
    if "articles" not in data:
        raise SessionExpired("200だがarticlesキー無し→失効疑い")
    return data
```

## ローカルと GitHub Actions で Cookie 受け渡しを切り替える

ローカルは `.env`、CI では `.env` をコミットできないため Secrets を環境変数へ注入する。同じ `os.environ["ZENN_SESSION"]` で両対応でき、コード分岐は不要だ。

```yaml
# .github/workflows/probe.yml
jobs:
  probe:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install httpx python-dotenv
      - run: python probe.py
        env:
          ZENN_SESSION: ${{ secrets.ZENN_SESSION }}
```

セッションの寿命は実測で約14日。CI が `SessionExpired` を送出したら Secrets を手動更新する運用が、空データ汚染を防ぐ最も確実な防壁になる。
