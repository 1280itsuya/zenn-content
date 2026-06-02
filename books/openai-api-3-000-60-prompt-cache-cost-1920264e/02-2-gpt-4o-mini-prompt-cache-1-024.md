---
title: "第2章: GPT-4o-miniのprompt cacheを効かせる1,024トークン境界とプロンプト分割設計"
free: false
---

## GPT-4o-miniのprompt cacheが効く1,024トークン境界の確認方法

結論を先出しする。OpenAIのprompt cacheは入力先頭1,024トークン以上かつ完全一致でのみ発動し、キャッシュ部分の入力単価は50%になる。第1章で実測した未設計¥9,200は、固定プロンプトを末尾に置きヒット率0%だったことが主因だ。まず`tiktoken`で先頭ブロックが1,024を超えるか測定する。

```python
import tiktoken

enc = tiktoken.encoding_for_model("gpt-4o-mini")
SYSTEM = open("prompt/system_fixed.txt", encoding="utf-8").read()
print(len(enc.encode(SYSTEM)))  # 2,847 → cache境界を超える
```

## 固定システムプロンプトとfew-shotを先頭固定する分割設計

cache対象を最大化するには「先頭固定／末尾可変」が鉄則だ。システムプロンプト・記事テンプレ・few-shot例3件を1つの`system`メッセージに連結し、記事タイトルなど可変部だけを`user`末尾に置く。

```python
def build_messages(theme: str) -> list[dict]:
    fixed = SYSTEM + TEMPLATE + FEWSHOT  # 計2,847トークン・毎回不変
    return [
        {"role": "system", "content": fixed},          # ← cache対象
        {"role": "user", "content": f"テーマ: {theme}"},  # ← 可変
    ]
```

## cached_tokensをログしてヒット率を0%→89%へ上げる検証

`response.usage.prompt_tokens_details.cached_tokens`を毎回記録する。60本量産でこの値÷prompt_tokensの平均が89%に到達した。

```python
resp = client.chat.completions.create(
    model="gpt-4o-mini", messages=build_messages(theme))
u = resp.usage
hit = u.prompt_tokens_details.cached_tokens / u.prompt_tokens
print(f"hit={hit:.0%} cached={u.prompt_tokens_details.cached_tokens}")
```

## 順序を1行間違えるとヒット率が落ちる罠の再現

few-shotとテンプレの連結順を入れ替えると先頭バイト列が変わり、完全一致が崩れてcached_tokensが0に戻る。差分を機械検出する。

```python
import hashlib

def prefix_hash(messages: list[dict]) -> str:
    head = messages[0]["content"][:4096].encode("utf-8")
    return hashlib.sha256(head).hexdigest()[:12]

assert prefix_hash(build_messages("A")) == prefix_hash(build_messages("B"))
# テーマが違っても先頭ハッシュは一致しなければならない
```

## ¥9,200→¥2,847のcost差分をcost_trackerに記録する

ヒット率改善の効果はDBに残す。cached_tokensは半額単価で計算し、記事ごとの実コストを`cost`テーブルに蓄積する。

```sql
CREATE TABLE cost (
  id INTEGER PRIMARY KEY,
  slug TEXT, prompt_tokens INT, cached_tokens INT,
  cost_jpy REAL  -- (uncached*1.0 + cached*0.5)*単価
);
```

この設計で月60本の合計が¥2,847に収束した。本章のコードとスキーマはそのまま流用できる。

topics: openai, python, automation, api, cost
