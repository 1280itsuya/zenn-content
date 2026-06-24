---
title: "Claude APIパイプラインの設計図——個人開発に最適なアーキテクチャをPythonで組む"
free: true
---

## Claude APIパイプラインの設計図——個人開発に最適なアーキテクチャをPythonで組む

### 最初に作るべきは「土台」

Claude APIを使って作業を自動化しようとすると、最初はシンプルに動く。`anthropic.Anthropic().messages.create(...)` を呼ぶだけだ。だがタスクが増えるにつれ、コードは壊れ始める。

- プロンプトが散らかり、どこを直せばいいかわからなくなる
- レート制限エラーが出るたびに手で再実行する羽目になる
- 「あのタスクで使ったプロンプト」がどのファイルにあるかわからない

これを防ぐには、**最初から再利用可能な土台クラス**を作ることが鍵だ。

---

### PromptTemplateクラスで「プロンプトの散らかり」を防ぐ

プロンプトはコードに埋め込まず、テンプレートとして分離する。

```python
from string import Template
from dataclasses import dataclass

@dataclass
class PromptTemplate:
    system: str
    user_template: str  # ${topic} などのプレースホルダを含む

    def render(self, **kwargs) -> str:
        return Template(self.user_template).safe_substitute(**kwargs)


# 使い方
BLOG_TEMPLATE = PromptTemplate(
    system="あなたはSEOに詳しい技術ライターです。",
    user_template="「${topic}」について、初心者向けに${word_count}字で記事を書いてください。",
)

rendered = BLOG_TEMPLATE.render(topic="Claude API入門", word_count=800)
```

`safe_substitute` を使う理由は、未定義の変数が残っていても例外を出さず `${変数名}` のまま残してくれるためだ。バグの温床になる `format()` より安全に使える。

---

### ベースクラスにエラーハンドリングを組み込む

個人開発で最も多いエラーは次の3種類だ。

| エラー | 原因 | 対処 |
|--------|------|------|
| `RateLimitError` | APIの呼びすぎ | 指数バックオフで再試行 |
| `APITimeoutError` | ネットワーク遅延 | タイムアウト設定+リトライ |
| `APIStatusError` | 認証切れ・無効モデル名 | 即座に例外を上げる |

これを毎回書くのは非効率なので、ベースクラスに閉じ込める。

```python
import time
import anthropic
from typing import Optional

class ClaudeClient:
    def __init__(self, model: str = "claude-sonnet-4-6", max_retries: int = 3):
        self.client = anthropic.Anthropic()
        self.model = model
        self.max_retries = max_retries

    def call(
        self,
        template: PromptTemplate,
        max_tokens: int = 1024,
        **kwargs,
    ) -> Optional[str]:
        user_msg = template.render(**kwargs)

        for attempt in range(self.max_retries):
            try:
                response = self.client.messages.create(
                    model=self.model,
                    max_tokens=max_tokens,
                    system=template.system,
                    messages=[{"role": "user", "content": user_msg}],
                )
                return response.content[0].text

            except anthropic.RateLimitError:
                wait = 2 ** attempt  # 1s → 2s → 4s
                print(f"レート制限。{wait}秒後にリトライ ({attempt+1}/{self.max_retries})")
                time.sleep(wait)

            except anthropic.APITimeoutError:
                print(f"タイムアウト。リトライ中 ({attempt+1}/{self.max_retries})")
                time.sleep(1)

            except anthropic.APIStatusError as e:
                # 認証エラーなど即時失敗すべきもの
                raise RuntimeError(f"APIエラー {e.status_code}: {e.message}") from e

        return None  # 全リトライ失敗
```

---

### 落とし穴：`max_tokens` を省略してはいけない

`max_tokens` を渡さないとAPIがエラーを返す。`anthropic` ライブラリにデフォルト値はない。最低でも `256` は指定しよう。出力が長くなりそうなタスク（記事生成など）は `2048〜4096` が現実的だ。

---

### 実際の呼び出しはこうなる

```python
client = ClaudeClient()

result = client.call(
    template=BLOG_TEMPLATE,
    max_tokens=2000,
    topic="Python非同期処理入門",
    word_count=800,
)

if result:
    print(result)
else:
    print("生成失敗。ログを確認してください。")
```

この構造を基点にすれば、次章以降で扱うバッチ処理・コスト計測・並列実行はすべてこの `ClaudeClient` を継承して拡張できる。プロンプトは `PromptTemplate` として `prompts/` フォルダに集め、ロジックとテキストを分離することが、長期メンテナンスの要だ。
