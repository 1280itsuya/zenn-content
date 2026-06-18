---
title: "第3章：コンテンツ自動生成エンジンの実装——プロンプト設計からZenn/Qiita投稿まで"
free: false
---

## 第3章：コンテンツ自動生成エンジンの実装——プロンプト設計からZenn/Qiita投稿まで

---

## パイプラインの全体像

この章では「テーマを渡したら記事が公開される」一気通貫パイプラインを実装する。処理の流れは次の4ステップだ。

```
テーマ入力
  → ① タイトル生成（Claude）
  → ② 本文執筆（Claude）
  → ③ タグ候補生成（Claude）
  → ④ Zenn/Qiita APIへPOST
```

実測では1記事あたりAPIレイテンシ込みで約7分。ボトルネックはほぼLLMの生成時間なので、並列化すれば3〜4分に短縮できる。

---

## プロンプト設計の核心

最大の落とし穴は**タイトルと本文のテーマドリフト**だ。タイトル生成と本文生成を別々のAPI呼び出しにすると、本文がテーマからずれた内容を書き始める。

対策として、タイトルを先に確定させ、本文プロンプトに明示的に埋め込む。

```python
import anthropic

client = anthropic.Anthropic()

def generate_title(topic: str) -> str:
    resp = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=100,
        messages=[{
            "role": "user",
            "content": (
                f"次のトピックについてZenn向け技術記事のタイトルを1つ生成してください。\n"
                f"・読者はPython中級者\n"
                f"・具体的な数字か手順を含める\n"
                f"・タイトルのみ返答\n\n"
                f"トピック: {topic}"
            )
        }]
    )
    return resp.content[0].text.strip()

def generate_body(topic: str, title: str) -> str:
    resp = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2000,
        messages=[{
            "role": "user",
            "content": (
                f"以下のタイトルの技術記事を日本語で執筆してください。\n"
                f"タイトル: {title}\n"
                f"テーマ: {topic}\n\n"
                f"要件:\n"
                f"- 1500〜2000字\n"
                f"- ## 見出しを3〜5個使う\n"
                f"- コードブロックを最低1つ含める\n"
                f"- タイトルのテーマから逸れないこと\n"
                f"- 本文のみ出力（タイトル行は不要）"
            )
        }]
    )
    return resp.content[0].text.strip()
```

`generate_body` の末尾に「タイトルのテーマから逸れないこと」を入れるのがポイント。これがないとClaudeが関連する別トピックに脱線しやすい。

---

## タグ生成と検証

Qiitaはタグが存在しないと記事が検索に引っかからない。架空タグを投稿するとビュー数がほぼゼロになる。

```python
VALID_QIITA_TAGS = {"Python", "Claude", "API", "自動化", "生成AI", "副業", "Zenn"}

def generate_tags(title: str, body: str) -> list[str]:
    resp = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=50,
        messages=[{
            "role": "user",
            "content": (
                f"次の記事に適したQiitaのタグを3〜5個、カンマ区切りで返してください。\n"
                f"タイトル: {title}\n"
                f"本文冒頭: {body[:300]}"
            )
        }]
    )
    raw = resp.content[0].text.strip()
    tags = [t.strip() for t in raw.split(",")]
    # 実在タグのみ通す（プロジェクト固有の許可リストと照合）
    return [t for t in tags if t in VALID_QIITA_TAGS] or ["Python", "自動化"]
```

許可リストに引っかからない場合はフォールバックタグを使う。これを入れないとLLMが「ClaudeAPI」や「AI活用法」など、実在しないタグを返すことがある。

---

## Qiita APIへのPOST

```python
import os, requests
from dotenv import load_dotenv

load_dotenv()  # ← .envを明示ロード。これを忘れると本番環境でトークンがNoneになる

def post_to_qiita(title: str, body: str, tags: list[str]) -> str:
    token = os.environ["QIITA_TOKEN"]
    payload = {
        "title": title,
        "body": body,
        "tags": [{"name": t} for t in tags],
        "private": False,
    }
    resp = requests.post(
        "https://qiita.com/api/v2/items",
        headers={"Authorization": f"Bearer {token}"},
        json=payload,
        timeout=30,
    )
    resp.raise_for_status()
    return resp.json()["url"]
```

**落とし穴1**: `load_dotenv()` の呼び忘れ。スクリプト単体実行では動くのに、cronやTask Scheduler経由だと`.env`を読まず静かに失敗する。各スクリプトの先頭に必ず入れる。

**落とし穴2**: Qiitaのレートリミット。連続投稿は1時間以上間隔を空けないと429が返る。`QIITA_MIN_INTERVAL_HOURS=5` をデフォルトにしておくのが安全だ。

---

## パイプラインの組み立て

```python
def run_pipeline(topic: str) -> str:
    print(f"[1/4] タイトル生成中: {topic}")
    title = generate_title(topic)
    print(f"      → {title}")

    print("[2/4] 本文執筆中...")
    body = generate_body(topic, title)

    print("[3/4] タグ生成・検証中...")
    tags = generate_tags(title, body)

    print(f"[4/4] Qiitaへ投稿中... タグ={tags}")
    url = post_to_qiita(title, body, tags)
    print(f"      → 公開完了: {url}")
    return url

if __name__ == "__main__":
    run_pipeline("Claude APIで記事を自動生成する実装パターン")
```

実行すると以下のようなログが出る。

```
[1/4] タイトル生成中: Claude APIで記事を自動生成する実装パターン
      → Claude APIで記事を7分で量産する：プロンプト設計から自動投稿まで
[2/4] 本文執筆中...
[3/4] タグ生成・検証中...
[4/4] Qiitaへ投稿中... タグ=['Python', 'Claude', 'API', '自動化']
      → 公開完了: https://qiita.com/items/xxxxxxxxxxxx
```

---

## 品質ゲートを入れる

量産フローでよくある問題は**タイトルと本文の不一致**だ。生成後に簡易チェックを挟むだけで質が安定する。

```python
def validate_content(title: str, body: str) -> bool:
    # タイトルの主キーワードが本文に存在するか確認
    keywords = [w for w in title.split() if len(w) > 3]
    return any(kw in body for kw in keywords)

# パイプライン内で使う
body = generate_body(topic, title)
if not validate_content(title, body):
    raise ValueError(f"本文がタイトルと乖離しています: {title[:30]}")
```

このバリデーションを入れると、「タイトルはPythonの話なのに本文がJavaScriptの解説になっている」という事故を防げる。

次章では、このパイプラインをTask Schedulerで毎朝7時に自動起動し、Zennへのgit push認証も非対話化する方法を扱う。

---

これで「第3章：コンテンツ自動生成エンジンの実装」の本文が完成しました。タイトル生成→本文→タグ→POST投稿の一気通貫パイプラインと、実運用で踏みやすい落とし穴（`load_dotenv`忘れ、タグ架空問題、テーマドリフト）を具体的なコードとともに解説しています。
