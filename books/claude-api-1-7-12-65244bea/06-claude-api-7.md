---
title: "落とし穴と回避策：Claude APIパイプラインで実際にハマった7つの罠"
free: false
---

## 落とし穴と回避策：Claude APIパイプラインで実際にハマった7つの罠

---

### 罠1：レート制限で無言スキップ

APIが429を返してもリトライせず、その記事だけ静かに消える。ログを見ると「RateLimitError」の1行だけ。

```python
import time, anthropic

def call_with_retry(client, **kwargs, retries=5):
    for i in range(retries):
        try:
            return client.messages.create(**kwargs)
        except anthropic.RateLimitError:
            wait = 2 ** i * 10  # 指数バックオフ
            time.sleep(wait)
    raise RuntimeError("レート制限: 上限に達しました")
```

Qiitaなど外部APIも同様。最低5時間インターバルを環境変数で制御する設計にしておくと運用が楽になる。

---

### 罠2：プロンプトドリフト（タイトル≠本文）

「節約術」というタイトルで生成を始めたのに、本文がいつの間にか「投資戦略」になっていた。長いチェーンでコンテキストが薄れる典型パターン。

```python
def validate_content(title, body):
    # タイトルのキーワードが本文に含まれているか検証
    keywords = extract_keywords(title)
    missing = [kw for kw in keywords if kw not in body[:500]]
    if missing:
        raise ValueError(f"本文がタイトルと乖離: {missing}")
```

生成後に必ず検証し、不一致なら中止・再生成する。エージェントにタイトルを `system` プロンプトで固定するのも有効。

---

### 罠3：`git push` がTTYなしでハング

スケジュール実行（Task Scheduler、cron）では対話的なGitHub認証ダイアログが出て、そのまま2時間放置される。

```bash
# .env に書いておく
GIT_TERMINAL_PROMPT=0
GITHUB_TOKEN=ghp_xxxx

# リモートURLにPATを埋め込む
git remote set-url origin https://${GITHUB_TOKEN}@github.com/user/repo.git
```

PAT平文は `.env` に閉じ込め、`.gitignore` に追加する。リモートに平文でpushするのは厳禁。

---

### 罠4：トークン爆死

`max_tokens` を設定せずに長いプロンプトを投げると、出力が数千トークンに膨らんで予算が溶ける。月次レポートの出力が4000トークンを超えた時は目を疑った。

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=800,        # 必ず上限を設定
    temperature=0.7,
    messages=[{"role": "user", "content": prompt}]
)
```

用途ごとに上限を変える。ブログ本文は1200、タグ生成は50、要約は300が目安。

---

### 罠5：`~/.claude.json` の書き込み競合

対話セッション中に `claude -p` を手動起動すると、同じファイルへの書き込みが衝突して最初の生成が破損する。完走するが出力が壊れている。

```
# 症状: 生成結果がJSON途中で切れる
{"role": "assistant", "content": "記事を
```

回避策は**対話セッションを閉じてからスケジュール実行を起動**すること。本来はTask Scheduler前提の設計にしておく。

---

### 罠6：オーケストレーターがパイプ待ちでハング

`claude -p` にパイプ入力を渡すスクリプトが、内部でstdinを待ち続けてCPU 0%のまま6〜12時間フリーズ。投稿もレポートも届かない。

```python
# NG: stdinを使うとハングの原因になることがある
result = subprocess.run(["claude", "-p", prompt], stdin=subprocess.PIPE, ...)

# OK: プロンプトをファイル経由で渡す
with open("tmp_prompt.txt", "w") as f:
    f.write(prompt)
result = subprocess.run(["claude", "-p", "--file", "tmp_prompt.txt"], ...)
```

ハングしたら `taskkill /F /IM claude.exe`、ロックファイル削除、再実行の順で復旧する。

---

### 罠7：`load_dotenv()` の書き忘れで静かにスキップ

環境変数が読まれず、投稿処理全体が何も言わずにスルーされる。エラーログすら出ない。

```python
# 全ての poster.py の先頭に必ず入れる
from dotenv import load_dotenv
load_dotenv()  # ← これが抜けると ZENN_TOKEN 等が None になる

import os
token = os.getenv("ZENN_TOKEN")
if not token:
    raise EnvironmentError("ZENN_TOKEN が未設定です")  # 明示的に落とす
```

`if not token: return` とサイレントスキップするコードが最も発見しにくい。必ず `raise` で止める。

---

これら7つの罠に共通するのは「**エラーが出ない**」という点だ。失敗を沈黙で飲み込むコードが量産パイプラインを最も壊しやすい。各ステップで明示的に検証し、異常を大声で報告させることが、自動化を本番運用に耐えさせる唯一の方法である。
