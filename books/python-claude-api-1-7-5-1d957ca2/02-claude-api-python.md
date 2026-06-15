---
title: "Claude API×Pythonで記事生成パイプラインの基盤を構築する"
free: true
---

## Claude APIキーの取得と安全な管理

まず[Anthropic Console](https://console.anthropic.com)でAPIキーを発行する。発行後、**絶対にコードに直書きしない**。`.env`ファイルに保存し、`.gitignore`に追加するのが最低限のルールだ。

```bash
# .env
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxx
```

```python
# config.py
from dotenv import load_dotenv
import os

load_dotenv()
API_KEY = os.getenv("ANTHROPIC_API_KEY")
if not API_KEY:
    raise EnvironmentError("ANTHROPIC_API_KEY が .env に設定されていません")
```

**落とし穴**: `load_dotenv()`を呼ぶ前に`os.getenv()`を実行すると常に`None`になる。順序を必ず守ること。

---

## プロンプトテンプレートの設計

アフィリ記事生成で重要なのは、**テーマ・ターゲット・アフィリリンクの差し込み口**をテンプレートとして分離することだ。プロンプトをハードコードすると記事が均質化し、スパム判定を受けやすくなる。

```python
# templates/article_prompt.py

ARTICLE_PROMPT = """\
あなたはSEOに強い日本語ライターです。
以下の条件で{word_count}字前後のアフィリエイト記事を書いてください。

【テーマ】{topic}
【ターゲット読者】{target}
【訴求するアフィリ商品】{product}
【必ず含めるキーワード】{keywords}

構成:
1. 読者の悩みを具体的に提示する導入(200字)
2. 商品・サービスの特徴と解決策(400字)
3. 実体験風の使用レビュー(300字)
4. まとめとCTA(100字)

記事本文のみ出力し、見出しはMarkdown(##, ###)で記述すること。
"""

def build_prompt(topic, target, product, keywords, word_count=1000):
    return ARTICLE_PROMPT.format(
        topic=topic,
        target=target,
        product=product,
        keywords=", ".join(keywords),
        word_count=word_count,
    )
```

テンプレートを外部化しておくと、後から品質改善したいときにコード側を触らずにプロンプトだけ差し替えられる。

---

## バッチ生成スクリプトの実装

記事ネタをCSVで管理し、1コマンドで複数記事を連続生成するスクリプトを作る。

```csv
# articles.csv
topic,target,product,keywords
格安SIM比較2025,スマホ代を下げたい20代,楽天モバイル,格安SIM 乗り換え おすすめ
在宅ワーク椅子おすすめ,腰痛持ちのテレワーカー,オカムラ チェア,腰痛 椅子 テレワーク
```

```python
# generate_batch.py
import anthropic
import csv
import time
from pathlib import Path
from config import API_KEY
from templates.article_prompt import build_prompt

client = anthropic.Anthropic(api_key=API_KEY)

def generate_article(topic, target, product, keywords_str):
    keywords = [k.strip() for k in keywords_str.split()]
    prompt = build_prompt(topic, target, product, keywords)

    message = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2000,
        messages=[{"role": "user", "content": prompt}],
    )
    return message.content[0].text

def run_batch(csv_path="articles.csv", output_dir="output"):
    Path(output_dir).mkdir(exist_ok=True)

    with open(csv_path, encoding="utf-8") as f:
        reader = csv.DictReader(f)
        for i, row in enumerate(reader):
            print(f"[{i+1}] 生成中: {row['topic']}")
            try:
                article = generate_article(**row)
                filename = f"{output_dir}/article_{i+1:03d}.md"
                Path(filename).write_text(article, encoding="utf-8")
                print(f"    -> 保存: {filename}")
            except anthropic.RateLimitError:
                print("    レート制限。60秒待機...")
                time.sleep(60)
            time.sleep(3)  # API呼び出し間隔

if __name__ == "__main__":
    run_batch()
```

**落とし穴その1**: `RateLimitError`を握り潰すと途中の記事が欠番になる。必ずリトライ処理を入れること。

**落とし穴その2**: `time.sleep(3)`なしで連続リクエストを投げると無料枠ではすぐにレート制限に引っかかる。Tier1では1分あたりのリクエスト数に上限があるため、記事間に最低3秒の待機を挟む。

---

## 生成結果の品質チェック

全自動で記事を量産すると、タイトルと本文がズレる「テーマドリフト」が発生しやすい。最低限、以下の検証を生成直後に挟む。

```python
def validate_article(topic, article_text):
    # テーマキーワードが本文に含まれるか簡易チェック
    core_word = topic.split()[0]  # 例: "格安SIM"
    if core_word not in article_text:
        raise ValueError(f"テーマ '{core_word}' が本文に含まれていません。再生成が必要です。")
    if len(article_text) < 500:
        raise ValueError("本文が短すぎます(500字未満)。")
```

このチェックを`generate_article()`の戻り値に噛ませるだけで、明らかな失敗記事がoutputディレクトリに混入するのを防げる。

---

以上がパイプラインの基盤だ。CSVにネタを追加するだけで記事が増産できる状態になった。次章ではこの出力をZennやはてなブログへ自動投稿する仕組みを実装する。
