---
title: "第3章: tiktokenとusageでトークンを実測しSQLiteに刻むcost_tracker.pyの実装"
free: false
---

この章のZennトピック（slug）: `openai` / `python` / `automation` / `api` / `cost`

## cost_tracker.pyのSQLiteスキーマを1リクエスト1行で確定する

結論から書く。推定値で家計簿をつけると必ずズレる。OpenAI APIのレスポンス`usage`を唯一の正とし、`model`・`prompt_tokens`・`cached_tokens`・`completion_tokens`・`cost_yen`を1リクエスト=1行でSQLiteに刻む。cache未設計で月¥9,200だったコストが、このテーブルで漏れを可視化した結果¥2,847まで落ちた。まずスキーマを作る。

```python
import sqlite3, time
DB = "cost.db"
def init_db():
    con = sqlite3.connect(DB)
    con.execute("""CREATE TABLE IF NOT EXISTS api_log(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        ts INTEGER, article_id TEXT, model TEXT,
        prompt_tokens INTEGER, cached_tokens INTEGER,
        completion_tokens INTEGER, cost_yen REAL)""")
    con.commit(); con.close()
init_db()
```

## 料金テーブルをdictで持ちcached_tokensに0.5倍を適用する円換算ロジック

gpt-4o-miniの料金は1Mトークンあたりinput $0.15 / output $0.60。cached_tokensはinput単価の0.5倍で課金されるため、別係数で計算する。為替レートは`USD_JPY`に差し込み口を1つ設け、ここだけ書き換えれば全行に反映される。

```python
USD_JPY = 158.0
PRICE = {  # USD / 1M tokens
    "gpt-4o-mini": {"in": 0.15, "cached": 0.075, "out": 0.60},
}
def calc_yen(model, p_tok, c_tok, o_tok):
    r = PRICE[model]
    uncached = p_tok - c_tok
    usd = (uncached*r["in"] + c_tok*r["cached"] + o_tok*r["out"]) / 1_000_000
    return round(usd * USD_JPY, 4)
```

## usageを正としてlog_usage()でAPIレスポンスを記録する

`response.usage`からトークンを取り出す。cached_tokensは`prompt_tokens_details.cached_tokens`に入る。推定を挟まず、返ってきた実数だけを記録する。

```python
def log_usage(article_id, resp):
    u = resp.usage
    cached = getattr(u, "prompt_tokens_details", None)
    c_tok = cached.cached_tokens if cached else 0
    yen = calc_yen(resp.model, u.prompt_tokens, c_tok, u.completion_tokens)
    con = sqlite3.connect(DB)
    con.execute("INSERT INTO api_log(ts,article_id,model,prompt_tokens,"
        "cached_tokens,completion_tokens,cost_yen) VALUES(?,?,?,?,?,?,?)",
        (int(time.time()), article_id, resp.model,
         u.prompt_tokens, c_tok, u.completion_tokens, yen))
    con.commit(); con.close()
    return yen
```

## tiktokenで送信前に概算しusageとの誤差を検証する

tiktokenで送信前トークンを概算し、APIの`prompt_tokens`と突き合わせる。誤差が±5%を超える章はテンプレートにシステムメッセージの数え漏れがある。

```python
import tiktoken
def estimate(text, model="gpt-4o-mini"):
    enc = tiktoken.encoding_for_model(model)
    return len(enc.encode(text))

est = estimate(prompt_text)        # 送信前
# 実測 u.prompt_tokens と比較
err = abs(est - u.prompt_tokens) / u.prompt_tokens
print(f"推定{est} / 実測{u.prompt_tokens} / 誤差{err:.1%}")
```

## 日次・記事別の集計クエリでcache効果を数値で確認する

最後に集計する。記事60本量産時に「どの記事が高いか」「cacheが効いた割合」を1クエリで出す。cache率が60%を切る記事は、共通プレフィックスがprompt先頭に固定されていない。

```sql
-- 日次コスト
SELECT date(ts,'unixepoch','localtime') d, ROUND(SUM(cost_yen),1) yen,
       COUNT(*) reqs FROM api_log GROUP BY d ORDER BY d DESC;

-- 記事別のcache率
SELECT article_id, SUM(cost_yen) yen,
       ROUND(100.0*SUM(cached_tokens)/SUM(prompt_tokens),1) cache_pct
FROM api_log GROUP BY article_id ORDER BY yen DESC LIMIT 10;
```

この`cache_pct`列が¥9,200→¥2,847の差を生んだ計測点だ。次章ではこの数値を使い、cache率が低い記事のプロンプト構造を再設計する。
