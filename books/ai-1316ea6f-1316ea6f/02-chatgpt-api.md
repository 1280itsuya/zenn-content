---
title: "自動化エンジンの作り方——ChatGPTとAPIで「稼ぐ仕組み」を組み立てる"
free: false
---

## 自動化エンジンとは何か

「稼ぐ仕組み」を作るとは、**インプット→生成→アウトプット**の流れを人の手なしで回し続けることだ。ChatGPT（OpenAI API）はその生成エンジン。APIを叩けば、文章・コード・要約を数秒で生み出す。

---

## 最小構成：3ステップで動くパイプライン

最初から複雑に作らなくていい。核心は以下の3層だ。

```
① トリガー（スケジューラ／入力データ）
② 生成エンジン（OpenAI API）
③ 出力先（ブログ・note・メール等）
```

たとえば「毎朝7時にキーワードを与えてブログ記事を生成→Zennに投稿」という流れをPythonで実装する。

---

## APIキーの取得とセットアップ

まずOpenAIのAPIキーを用意し、環境変数に格納する。

```bash
pip install openai python-dotenv
```

`.env` ファイル：

```
OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxxxxxx
```

---

## 記事生成エンジンの実装

```python
import os
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()
client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

def generate_article(keyword: str) -> str:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "あなたは副業・AI活用に詳しいライターです。"},
            {"role": "user", "content": f"「{keyword}」について800字の解説記事を書いてください。"}
        ]
    )
    return response.choices[0].message.content
```

`gpt-4o-mini` を使えば1記事のAPI代は約**0.3〜1円**。月100本生成しても数百円に収まる。

---

## スケジューラで「自動」にする

Windowsならタスクスケジューラ、LinuxならCronで毎朝7時に実行させる。

```bash
# cron設定例
0 7 * * * /usr/bin/python3 /home/user/auto_blog/run.py
```

`run.py` は①キーワードをリストから取得 → ②`generate_article()` を呼ぶ → ③Zenn/note APIで公開、という3行の処理だ。

---

## 「稼ぐ」に繋げるポイント

生成記事にアフィリエイトリンクを差し込むだけでは弱い。重要なのは**記事末尾への誘導設計**だ。

- 記事の結論部分でGumroad商品やBoothキットへCTAを設置
- 「詳しくはこちら→〇〇ツールキット（¥980）」と1行添える
- Zenn・Qiitaに投稿すればビューが自然発生し、流入コスト0

自動生成→自動投稿→読者が記事を読む→リンクをクリック——この回路が一度動き出せば、あとはキーワードリストを補充するだけで収益の分母が増え続ける。

---

## 次章への橋渡し

エンジン単体ではまだ「仕掛け」に過ぎない。第3章では、このエンジンに**複数の出力先（マルチチャネル）**を接続し、1回の生成で複数プラットフォームへ同時配信する構成を解説する。
