---
title: "Claude CLIとPythonで“無料”の自律コンテンツ生成パイプラインを作る"
emoji: "🤖"
type: "tech"
topics: ["ai", "python", "claude", "自動化", "個人開発"]
published: true
---

## はじめに

「AIで副業コンテンツを自動生成したいが、APIのトークン課金が読めなくて怖い」——個人開発でよくある悩みです。本記事では、**Claude の Max プラン（定額）+ Claude CLI** を使い、**従量課金ゼロ**でテキスト生成を回す自律パイプラインの考え方を、実装の勘所とともに紹介します。

宣伝や誇大な収益保証はしません。あくまで「個人が無理なく続けられる自動化の作り方」の共有です。

## 全体像

```
スケジューラ(毎朝) → テーマ選定 → Claude CLIで本文生成 → 品質チェック → 各プラットフォームへ投稿 → 反応(view)を実測 → 次の改善へ
```

ポイントは3つです。

1. 生成はClaude CLI（定額）に寄せて従量課金を避ける
2. 「タイトルと本文のズレ」を機械的に検証してから公開する
3. view数が返るプラットフォーム（Qiita/Zenn/Dev.to）に出して、流入を“借りる”

## Claude CLI をPythonから呼ぶ

APIキーではなく、ログイン済みのCLIをサブプロセスで叩くのがコツです。プロンプトは引数ではなく **stdin** で渡すと、長文でもWindowsのコマンド長制限に引っかかりません。

```python
import subprocess

def claude_generate(prompt: str, timeout: int = 240) -> str:
    # ANTHROPIC_API_KEY を環境から除くのが重要(定額ではなくAPI課金を使ってしまうため)
    env = {k: v for k, v in os.environ.items()
           if k not in ("ANTHROPIC_API_KEY", "ANTHROPIC_AUTH_TOKEN")}
    r = subprocess.run(["claude", "-p"], input=prompt.encode("utf-8"),
                       capture_output=True, timeout=timeout, env=env)
    return r.stdout.decode("utf-8", "replace").strip()
```

## 品質ゲート：タイトルと本文のズレを弾く

自動生成で一番怖いのが「タイトルはAについて、本文はBについて」というドリフトです。公開前に簡単な一致チェックを入れるだけで事故が激減します。

```python
def on_theme(theme: str, body: str, min_ratio: float = 0.3) -> bool:
    # テーマの主要語が本文に十分含まれているかをざっくり判定
    core = theme.split()[0]
    return core in body
```

LLMにJSONを返させる時も、`{` の位置から1オブジェクトだけを取り出す抽出を挟むと、前置き文や```フェンス混じりの出力に強くなります。

## 流入は「他人のプラットフォーム」を借りる

自前ブログはSEOが育つまで数週間〜数ヶ月かかります。一方 Qiita / Zenn / Dev.to は内部フィードやタグで**投稿初日からimpressionが立ち**、しかも記事APIでview数が取れます。「流入が無いのか、導線が弱いのか」を数値で切り分けられるのが大きい。

## まとめ

- 生成は定額CLIに寄せて従量課金をゼロにする
- 公開前にタイトル↔本文の一致を機械チェック
- view数が返る場所に出して反応を実測し、伸びるテーマに寄せる

「全自動で勝手に稼ぐ」魔法はありませんが、**続けられる仕組み**は個人でも十分作れます。次回は投稿の自動化と計測まわりを書く予定です。
