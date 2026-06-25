---
title: "第3章：Claude APIで記事を自動生成する——個人開発で使えるプロンプト設計と品質ゲート"
free: false
---

## 第3章：Claude APIで記事を自動生成する——個人開発で使えるプロンプト設計と品質ゲート

### なぜ「タイトル≠本文」が起きるのか

Claude APIで記事を量産し始めると、必ず直面するバグがある。タイトルは「Pythonで競艇予想AIを作る」なのに、本文を読むと全然違うテーマの話が書いてある——いわゆる「テーマドリフト」だ。これはモデルの問題ではなく、プロンプト設計の問題だ。テーマをシステムプロンプトに書くだけでは不十分で、**生成後に本文がテーマと一致しているかを検証する仕組み**が必要になる。

### プロンプト設計：テーマ固定の3原則

第一に、テーマをシステムプロンプトとユーザーメッセージの両方に埋め込む。片方だけでは長い生成の途中でテーマが揺れる。

```python
def build_prompts(title: str, keywords: list[str]) -> tuple[str, str]:
    theme_line = f"テーマ: 「{title}」。このテーマから一切逸れないこと。"
    system = f"""あなたは技術ブログライターです。
{theme_line}
キーワード: {', '.join(keywords)}
本文の全段落で上記テーマに関連する内容のみを書くこと。"""

    user = f"""以下の記事を書いてください。
タイトル: {title}
制約: {theme_line}
構成: 導入→課題→実装→まとめ の順。"""
    return system, user
```

第二に、温度パラメータは`temperature=0.5`以下に絞る。創造性よりテーマ遵守を優先するフェーズでは高温度は敵だ。

第三に、出力フォーマットを指定してJSONで受け取る。後段の検証が格段に楽になる。

### 本文一致検証：生成直後に機械チェック

生成が終わったら即座に「タイトルのキーワードが本文に何回出てくるか」を数える。これだけでドリフトの大半を弾ける。

```python
def verify_theme_consistency(title: str, body: str) -> dict:
    title_words = set(title.replace("・", " ").split())
    body_lower = body.lower()

    hits = {w: body_lower.count(w.lower()) for w in title_words if len(w) > 1}
    total_hits = sum(hits.values())
    
    return {
        "passed": total_hits >= 3,        # 最低3回言及が必要
        "hits": hits,
        "total": total_hits,
    }
```

`passed=False`なら生成をリトライする。リトライ上限（通常2回）を超えたら記事を**廃棄**する。公開しないことが最善の品質管理だ。

### LLM-as-judge：GPTではなくClaudeに採点させる

機械チェックを通過した記事は、別のClaudeコールで品質採点する。同じモデルに「採点者」の役割を与えるのがポイントで、追加コストは1記事あたり約¥0.5前後と安い。

```python
def llm_judge(title: str, body: str, client) -> dict:
    prompt = f"""以下の記事を採点してください。
タイトル: {title}
本文:
{body}

採点基準（各20点、合計100点）:
1. タイトルと本文の一致度
2. 具体性（数字・コード・固有名詞があるか）
3. 読者にとっての有用性
4. 構成の自然さ
5. 誤情報や矛盾がないか

JSON形式で返してください:
{{"score": 0-100, "breakdown": {{}}, "reject_reason": "合格なら空文字"}}"""

    resp = client.messages.create(
        model="claude-haiku-4-5-20251001",   # 採点はHaikuで十分・安い
        max_tokens=300,
        messages=[{"role": "user", "content": prompt}]
    )
    import json
    return json.loads(resp.content[0].text)
```

スコアが70未満なら`reject_reason`をログに残して廃棄する。このログが後から「なぜ記事が少ない日があるか」の説明になる。

### パイプライン全体を繋ぐ

```python
def generate_article(title: str, keywords: list[str], client) -> str | None:
    system, user = build_prompts(title, keywords)
    
    for attempt in range(3):
        resp = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=2000,
            temperature=0.4,
            system=system,
            messages=[{"role": "user", "content": user}]
        )
        body = resp.content[0].text

        # ステップ1：機械チェック
        check = verify_theme_consistency(title, body)
        if not check["passed"]:
            continue   # リトライ

        # ステップ2：LLM採点
        verdict = llm_judge(title, body, client)
        if verdict["score"] >= 70:
            return body  # 合格
        
        print(f"[REJECT] score={verdict['score']} reason={verdict['reject_reason']}")
    
    return None  # 3回失敗→廃棄
```

### 落とし穴：廃棄率が高すぎるとき

実運用で廃棄率が30%を超えたら、**プロンプトのキーワードが曖昧すぎる**サインだ。「節約術」より「楽天ポイントを月3万貯める節約術」のように具体的にする。LLM-as-judgeの閾値を下げるより、入力を直す方が根本解決になる。
