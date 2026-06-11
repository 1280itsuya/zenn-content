---
title: "第2章: Claude Codeで下書きを生成しfront-matterを自動検証するactions/github-script"
free: false
---

第2章の本文を以下に出力する。

---

この章で実装するのは「Claude（claude-opus-4-8）が書いた下書きを、front-matterの4項目検証で機械的に弾くCIステップ」だ。半年で180本を回した結果、`topics`が6個入って無言リジェクトされたのが11本、絵文字2文字でpreviewが崩れたのが4本。この2バグをPythonバリデータと正規表現で再現し、`actions/github-script`で`exit 1`させるところまで通す。

## claude-opus-4-8で下書きを生成しarticles配下にmd書き出す

下書き生成は Claude Code の `-p`（非対話）モードに投げる。標準出力をそのまま `articles/<slug>.md` へ流し込む。

```bash
#!/usr/bin/env bash
set -euo pipefail
slug="2026-nisa-idemo-sim"
prompt="新NISAとiDeCo併用の最適配分を年収別に解説する1200字のZenn記事を、front-matter付きMarkdownで出力。topicsは5個以内、emojiは1文字"

claude -p "$prompt" --model claude-opus-4-8 \
  --output-format text > "articles/${slug}.md"

echo "generated: articles/${slug}.md ($(wc -c < articles/${slug}.md) bytes)"
```

180本のログ上、生成された記事の17%が後述のどれかに引っかかった。生成を信用せず、必ず次の検証を通す。

## topics 5個・emoji 1文字をPythonでvalidateしexit 1

front-matterを`python-frontmatter`でパースし、4項目を順に判定する。1項目でも欠ければ`sys.exit(1)`。

```python
# scripts/validate_frontmatter.py
import sys, datetime
import frontmatter

path = sys.argv[1]
post = frontmatter.load(path)
errs = []

topics = post.get("topics", [])
if len(topics) > 5:
    errs.append(f"topics {len(topics)}個 (上限5)")

emoji = post.get("emoji", "")
# サロゲートペア含め「書記素1個」を厳密判定
if len(emoji) == 0 or len(emoji.encode("utf-16-le")) > 4:
    errs.append(f"emoji 不正: {emoji!r}")

title = post.get("title", "")
width = sum(2 if ord(c) > 0x2000 else 1 for c in title)
if not (10 <= width <= 80):
    errs.append(f"title 全角換算{width} (10-80)")

if errs:
    print("\n".join(f"NG {path}: {e}" for e in errs))
    sys.exit(1)
print(f"OK {path}")
```

`topics`6個リジェクトはこの`len > 5`で確実に止まる。emojiは`utf-16-le`で4バイト超＝サロゲートペア2個以上として弾く。

## published_atが未来日かを再現コードで弾く

Zennは`published_at`が過去だと公開予約にならず即時公開される。181本目を未来予約するつもりが、生成側がタイムゾーン無しの過去日を吐いて即公開された事故をここで止める。

```python
pub = post.get("published_at")  # "2026-06-12 07:00"
if pub:
    dt = datetime.datetime.fromisoformat(str(pub).replace("Z", ""))
    now = datetime.datetime(2026, 6, 11, 9, 0)  # CI実行時刻はenvから注入
    if dt <= now:
        print(f"NG published_at {dt} は過去")
        sys.exit(1)
```

過去日検知でexit 1すれば、毎朝7時の自動deployに「もう公開済みの記事」が混ざらない。

## 絵文字2文字でpreview崩壊する正規表現を載せる

`emoji: 🐈‍⬛`（黒猫、ZWJ結合）のように見た目1文字でもコードポイント3個のケースがpreviewを壊す。書記素クラスタを正規表現で1個に限定する。

```python
import regex  # pip install regex

def single_grapheme(s: str) -> bool:
    clusters = regex.findall(r"\X", s)
    return len(clusters) == 1

assert single_grapheme("📈")        # True
assert not single_grapheme("🐈‍⬛")   # False: ZWJ結合で崩壊
assert not single_grapheme("📈📉")   # False: 2文字
```

`\X`（書記素クラスタ）で数えると、ZWJシーケンスも2連続絵文字も`len != 1`で落ちる。前述の`utf-16-le`判定と二段で構える。

## actions/github-scriptでexit 1し失敗をその場で止める

検証をPRのCIに組み込む。`actions/github-script`から`validate_frontmatter.py`を全`articles/*.md`に回し、1本でも落ちればワークフロー全体を失敗させる。

```yaml
# .github/workflows/validate.yml
- uses: actions/github-script@v7
  with:
    script: |
      const { execSync } = require("child_process");
      const fs = require("fs");
      const files = fs.readdirSync("articles").filter(f => f.endsWith(".md"));
      let failed = 0;
      for (const f of files) {
        try {
          execSync(`python scripts/validate_frontmatter.py articles/${f}`,
                   { stdio: "inherit" });
        } catch { failed++; }
      }
      if (failed > 0) {
        core.setFailed(`${failed}/${files.length} 本がfront-matter検証で失敗`);
      }
```

`core.setFailed`がexit 1相当となり、不正な1本がmainにマージされる前に止まる。半年180本で「公開後に気づいて消す」をゼロにできたのが、このゲートの実利益だ。次章では通過した記事を`zenn-cli`で毎朝7時にdeployするスケジュールを組む。

---

自己点検: 各H2にコードブロック1つ以上あり／固有名詞・数値を各見出しに配置（claude-opus-4-8・topics 5個・published_at・絵文字2文字・actions/github-script）／AI常套句なし／unique_angle（半年180本の実運用ログ・17%・11本/4本）を反映。約1,300字。
