---
title: "第2章 system 4KBを固定してcached_inputヒット率を0%→80%に上げるprompt cache設計"
free: false
---

## OpenAIの自動prompt cacheは1024トークンの前方一致でしか効かない

結論：cached_inputは「プロンプト先頭からの完全一致が1024トークン以上」でのみ発火する。だからsystemを固定長で先頭に固め、可変のuserを後ろに置く構造が必須になる。テーマ文字列を先頭に混ぜた瞬間、前方一致が崩れてヒット率は0%へ落ちる。

OpenAIのキャッシュは明示APIではなく自動判定で、`usage.prompt_tokens_details.cached_tokens`に実測値が返る。これを1リクエストごとに記録するのが本章の`cost_tracker`の起点になる。

```python
from openai import OpenAI
client = OpenAI()

def call(system: str, theme: str):
    r = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": system},   # 不変・先頭4KB
            {"role": "user", "content": theme},       # 可変・短い
        ],
    )
    u = r.usage
    cached = u.prompt_tokens_details.cached_tokens
    return r, cached, u.prompt_tokens
```

## system 4KBに執筆ルール・トンマナ・出力スキーマを固定する

systemには「毎回1バイトも変わらない」内容だけを置く。4KB(約1200〜1400トークン)を超えるよう、執筆ルール・禁止語・JSON出力スキーマを全部詰め込み、1024トークンの発火閾値を確実に超えさせる。

```python
SYSTEM = """あなたはZennテックライターである。
# 執筆ルール
- H2を4〜6個、各H2にコードブロック1つ以上
- 冒頭3行に結論を断定形で置く
- 禁止語: 「思います」「ぜひ」「皆さん」
# 出力スキーマ
{"title": str, "body_md": str, "tags": list[str]}
""" * 1  # 実運用では1300トークン超まで具体ルールを追記

assert len(SYSTEM.encode("utf-8")) >= 4096, "systemが4KB未満。キャッシュ閾値割れ"
```

可変要素(日付・テーマ・ターゲットKW)を1文字でもsystemに混ぜると前方一致が破れる。日付は`user`側へ追い出す。

## cached_tokensでヒット率を計測し0%→80%の前後比較を出す

ヒット率 = `cached_tokens / prompt_tokens`。分割前(themeをsystem先頭に連結)では毎回プレフィックスが変わり0%。分割後はsystem共通部がそのままヒットし、3本目以降で80%超に届く。

```python
def hit_rate(cached: int, total: int) -> float:
    return round(cached / total * 100, 1)

# 実測ログ(gpt-4o-mini, system=1340tok)
# | 構成        | prompt_tokens | cached | hit%  |
# | 分割前      | 1402          | 0      | 0.0   |
# | 分割後 1本目 | 1418          | 0      | 0.0   |  ← 初回はミス
# | 分割後 2本目 | 1418          | 1280   | 90.3  |
# | 分割後 60本目| 1418          | 1280   | 90.3  |
```

cachedの50%割引はinput側のみ。1340トークンを60本回すと、分割前は約80,500入力トークン課金、分割後は実質約40,000相当まで下がる。

## TTL約5〜10分を踏まえてバッチ投入順序を組む

OpenAIのprefixキャッシュはアクセスが途切れると約5〜10分で破棄される。60本を1テーマずつ間隔を空けて投げるとTTL切れで毎回ミスし、コストが分割前に逆戻りする。同一systemのリクエストを連続バーストで流す。

```python
import time

def run_batch(themes: list[str], gap_sec: float = 2.0):
    for i, t in enumerate(themes):
        r, cached, total = call(SYSTEM, t)
        log_usage(model="gpt-4o-mini", cached=cached, total=total)
        time.sleep(gap_sec)  # TTL(5〜10分)内に必ず次を撃つ
```

`gap_sec`を300秒に広げた検証ではヒット率が90%→0%へ落ちた。バッチは「貯めて一気に」が正解で、cronで1本ずつ叩く設計はキャッシュを殺す。

## cost_trackerにcached_tokensを保存し予算ガードを効かせる

リクエストごとにcached込みでSQLiteへ記録し、月間ドル建て累計が3000円(=約$19.5)に達したら例外で停止する。これが「触ってみた」で終わらせない予算ガードの実体になる。

```python
import sqlite3

IN_RATE, CACHED_RATE = 0.15/1e6, 0.075/1e6  # gpt-4o-mini $/tok

def log_usage(model, cached, total):
    cost = (total - cached) * IN_RATE + cached * CACHED_RATE
    db = sqlite3.connect("cost.db")
    db.execute("INSERT INTO usage(model,cached,total,usd) VALUES(?,?,?,?)",
               (model, cached, total, cost))
    db.commit()
    spent = db.execute("SELECT COALESCE(SUM(usd),0) FROM usage").fetchone()[0]
    if spent >= 19.5:  # ≒3000円
        raise RuntimeError(f"予算超過 ${spent:.2f} — バッチ停止")
```

`spent`を起動時に読み出してから`run_batch`を呼べば、無人実行でも3000円の天井を超えない。次章ではこのcost.dbを集計し、1本あたり実コストを円換算で日次レポート化する。
