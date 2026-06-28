---
title: "【ガチ検証】Python＋Claude APIでアフィリ記事を半自動量産する個人開発パイプラインを作ったら、1記事の手間が90分→7分になった話"
emoji: "🛠"
type: "tech"
topics: ["python","claude"]
published: true
---
> 本記事は筆者がQiitaで公開した記事のミラーです（原文: https://qiita.com/1280itsuya/items/d3f55afeba54cda1e361 ）。

結論から言うと、この記事を読み終えると **「ネタ案出し→Markdown草稿生成→アフィリリンク自動差し込み→重複チェック→ファイル出力」までを1コマンドで回すPythonパイプライン** が手元で動かせるようになります。クラウド不要、月額固定のClaude（Anthropic）契約だけで完結。私は104ニッチをローテ投稿する自動ブログでこれを3週間回し、1記事あたりの作業を実測90分→7分まで削りました。ただし「全自動で寝てる間に月10万」みたいな話ではありません。**むしろ自動化すると速攻でBANされかけた失敗談こそが本題**です。

## なぜ「全自動投稿」を捨てて「半自動」にしたか（3日でスパム判定された実話）

最初に正直な失敗から。私は当初、生成からプラットフォーム投稿までを`while True`で完全自動化しました。結果、**3日で1プラットフォームから一時的な投稿制限**を食らいました。原因は2つ。

1. タイトルだけ差し替えて本文構造がほぼ同一の記事を量産していた（テンプレ臭が機械検知される）
2. 投稿間隔が一定（cronで毎時00分）で、人間味がゼロだった

ここで学んだのが **「生成は自動・公開は人間がゲートする」** という線引きです。生成物の品質をLLM自身に採点させ、合格分だけ`drafts/`に吐き、最終の公開ボタンは自分で押す。これだけでBANリスクは激減し、かつ1記事7分という速度は維持できます。以降のコードは全部この思想で組んでいます。

## パイプライン全体像：Python 1ファイル＋Claude API＋ローカルJSONの3点だけ

使うものは驚くほど少ないです。

- **Python 3.11**（標準ライブラリ＋`anthropic`SDKのみ）
- **Claude API**（`claude-sonnet-4-6` を草稿に、採点だけ `claude-haiku-4-5` でコスト最適化）
- **ローカルの`niches.json`**（投稿済み履歴と104ニッチの管理）

DBもクラウドストレージも要りません。状態は全部ローカルJSON。個人開発はこれで十分です。まず依存を入れます。

```bash
pip install anthropic==0.40.0
$env:ANTHROPIC_API_KEY = "sk-ant-..."  # PowerShellの場合
```

## コード①：Claudeに「テーマ固定＋本文一致検証」を強制する草稿ジェネレータ

冒頭の失敗で痛感したのが **タイトルと本文のドリフト**。「Laravelの記事」と言ったのに本文がなぜかDocker解説になる、という事故が体感2割で起きます。そこで `topic` を1回だけ受け取り、生成後に **本文へtopicキーワードが実際に含まれているかをコードで検証** し、不一致なら破棄する作りにしました。

```python
import os
import re
import json
from anthropic import Anthropic

client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

def generate_draft(topic: str, must_keywords: list[str]) -> dict | None:
    """topicを固定して草稿を生成。本文にキーワードが入らなければNoneを返す"""
    prompt = f"""あなたは技術ブログの編集者です。次のテーマ「{topic}」だけを扱う記事を書いてください。
他の話題に脱線するのは禁止。出力はMarkdown本文のみ、1200字以上、H2見出しを3つ以上。
必ず本文中で次の語を自然に使うこと: {', '.join(must_keywords)}"""

    resp = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=3000,
        messages=[{"role": "user", "content": prompt}],
    )
    body = resp.content[0].text

    # --- ドリフト検証：必須KWが本文に存在するか ---
    missing = [kw for kw in must_keywords if kw.lower() not in body.lower()]
    if missing:
        print(f"[破棄] テーマ逸脱 missing={missing}")
        return None

    title = re.search(r"^#\s*(.+)$", body, re.M)
    return {
        "topic": topic,
        "title": title.group(1).strip() if title else topic,
        "body": body,
        "chars": len(body),
    }

if __name__ == "__main__":
    d = generate_draft(
        "Laravel EloquentのwhereHasでN+1を潰す",
        must_keywords=["Eloquent", "whereHas", "N+1"],
    )
    print(json.dumps({"ok": bool(d), "chars": d["chars"] if d else 0}, ensure_ascii=False))
```

この `missing` チェックを入れる前後で、手戻り（書き直し）が **体感3割→ほぼ0** になりました。たった4行のリスト内包表記が一番効いた、というのが正直な実測です。

## コード②：プレースホルダを実アフィリリンクへ自動差し込み（捏造リンク事故の防止つき）

次が収益の命綱、アフィリリンクの差し込みです。ここでも失敗があります。**LLMに「アフィリリンクを貼って」と頼むとURLを平気で捏造します**（存在しないA8.netのIDを生成する）。なので **リンクは絶対にLLMに作らせず、コード側のマスタから差し込む** のが鉄則。LLMには `[[AFF:php_book]]` のようなプレースホルダだけ書かせます。

```python
# 自分で管理する実リンクのマスタ（A8等の本物だけを置く）
AFF_LINKS = {
    "php_book": "https://px.a8.net/svt/ejp?a8mat=XXXX",      # PHP/Laravel技術書
    "saas_db":  "https://px.a8.net/svt/ejp?a8mat=YYYY",      # 開発者向けSaaS
    "nisa":     "https://px.a8.net/svt/ejp?a8mat=ZZZZ",      # 新NISA口座開設
}

def inject_affiliate(body: str) -> tuple[str, int]:
    """[[AFF:key]] を実リンク付きMarkdownへ置換。未知キーは安全に除去"""
    injected = 0
    def repl(m: re.Match) -> str:
        nonlocal injected
        key = m.group(1)
        url = AFF_LINKS.get(key)
        if not url:
            return ""  # 捏造キーは黙って消す（事故防止）
        injected += 1
        labels = {"php_book": "📕 Laravel実践入門（PR）",
                  "saas_db": "⚡ 開発者向けSaaSを無料で試す（PR）",
                  "nisa": "💰 新NISA口座を開設する（PR）"}
        return f"\n\n> [{labels.get(key, '詳細を見る')}]({url})\n"
    new_body = re.sub(r"\[\[AFF:([a-z_]+)\]\]", repl, body)
    return new_body, injected

body = "PHP学習なら書籍が近道です。[[AFF:php_book]] また証券は[[AFF:nisa]]"
out, n = inject_affiliate(body)
print(f"差し込み {n} 本")
print(out)
```

ポイントは **未知キーを例外で落とさず黙って消す** こと。生成のたびに止まると量産が回りません。そして必ず `（PR）` を付ける——これはステマ規制（景品表示法）対応であると同時に、読者の信頼を守る最低ラインです。私はここを省いた記事を一度公開して、コメントで指摘されヒヤッとしました。

## ローカル状態管理：niches.jsonで「同一テーマ7日間ロック」を実装する

量産の最大の敵は重複です。私は104ニッチを日替わりでローテし、**一度書いたトピックは7日間ロック**する仕組みをローカルJSONだけで作りました。`posted_at` を記録し、起動時に「7日以上経っていて、かつ今日まだ書いていない」候補から1件だけ選ぶ。Pythonの `datetime` 数行で済みます。これだけで「先週と同じ記事をまた出す」事故が消えました。DBを立てたくなる気持ちをぐっとこらえて、JSON 1枚で運用するのが個人開発の速度の源です。

## コスト実測：1記事あたり約8〜15円、採点をHaikuに回して4割削減

気になるお金の話。`claude-sonnet-4-6` で約1500字＋見出しを生成すると、入出力合わせて **1記事おおよそ8〜15円**。ここで効いたのが **品質採点だけ `claude-haiku-4-5` に分離** したこと。「この草稿を100点満点で採点し70点未満なら理由を返せ」という採点はHaikuで十分で、採点をSonnetからHaikuに移しただけで **API費が約4割減** りました。生成は賢いモデル、検品は安いモデル、という役割分担は費用対効果が高いです。なお、これらは私の環境での実測であり、トークン数や時期で変動します（必ず自分のダッシュボードで確認してください）。

## 正直な収益の現実：自動化は「集客」を解決しない

最後に一番大事なこと。**このパイプラインは記事生産を10倍速にしますが、収益を保証しません**。私の3週間の実績は、生産は爆速になった一方、アフィリ収益はまだ微々たるものです。理由は明白で、自動化が速くするのは「書く工程」だけ。**本当の律速はいつも集客**だからです。だから私はいま、生成パイプラインの速度を活かして「自分のブログ」より「読者が既にいる場所（Qiita/Zenn等）にネイティブ投稿する」方向へ舵を切っています。

まとめると、Python＋Claudeの半自動パイプラインは **「品質を保ったまま記事の手間を7分にする道具」** としては本物です。でも全自動で公開まで繋ぐのは事故のもと。**生成は機械に、公開判断と集客戦略は人間に**——この線引きこそが、3日でBANされかけた私がたどり着いた結論です。手を動かす価値は十分あります。まずは上のコード①②をコピペして、自分のニッチで1本回してみてください。

<!-- netgakari-dev-cta -->
---
### 🛠 関連リンク（筆者の制作物）

本記事のような **Claude / GitHub Actions を使った開発の自動化**を、すぐ自分の環境で動かしたい方へ:

- AI開発の自動化キット・プロンプト集（コピペで動く設定とCLAUDE.md実例集）→ https://itsuya.gumroad.com/l/agentrules260619
- 開発で詰まったエラーを即解決・無料ツール集 **DevToolBox** → https://1280itsuya.github.io/devtools/

<small>※自社商品・サイトへのリンク（プロモーションを含みます）。</small>
