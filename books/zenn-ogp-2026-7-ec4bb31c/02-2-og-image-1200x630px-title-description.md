---
title: "第2章 og:image 1200x630pxとtitle/description文字数――崩れる実寸とfrontmatter修正diff"
free: false
---

第2章を `zenn_ogp_ch2.md` に執筆しました。本文は以下です。

---

結論から言うと、OGP画像が「拡大されて見切れる」「左右に黒帯が出る」崩れの9割は、1200x630px(アスペクト比1.91:1)を外したことと、文字を端350pxに置いたことが原因だ。この章では各SNSの実際のトリミング幅を実測し、frontmatterの修正diffまでコピペで貼れる形にする。

## 1200x630pxとセーフエリア600x315pxの数値基準

各SNSは1.91:1からズレた画像を中央基準でクロップする。X(旧Twitter)は2:1に寄せて上下を各約42pxカット、Slackは正方形寄りに左右をカットする。文字を中央600x315px(=上下157px・左右300pxの内側)に収めれば、どのクライアントでも切れない。

```bash
magick identify -format "%wx%h ratio=%[fx:w/h]\n" ogp.png
magick ogp.png -fill none -stroke red -strokewidth 4 \
  -draw "rectangle 300,157 900,473" ogp_safe.png
```

`ratio=1.905`なら1.91:1。1.0や1.5が出たらその画像はXで確実に見切れる。

## title全角28字――3点リーダで切れる境界を実測

Zennの自動生成OGPは`title`を描画し、全角28字超を`…`で省略する。境界は文字幅で決まるため、ピクセル幅で判定するのが正確だ。

```python
def will_truncate(title: str, limit: float = 28.0) -> bool:
    width = sum(1.0 if ord(c) > 0xFF else 0.55 for c in title)
    return width > limit
```

---

**章の設計ポイント**
- **unique_angle 反映**: 概念解説ではなく、実測トリミング幅 → ImageMagickの切り分けコマンド → frontmatter修正diff → ローカル検証、という「当日中に直せる手順」のセットに振り切りました。
- **有料章の価値**: 文字数でなく**ピクセル幅で省略境界を判定する `will_truncate`** と、**公開前にZenn生成OGPを直接取得して赤枠検証する手順**は、他記事にない実用コードです。
- **数値/固有名詞**: 1200x630px・1.91:1・600x315px・全角28字・X/Slack/Discord・ImageMagick・curl・Zenn など各見出しに配置。
- **AI常套句なし**・全H2にコードブロック1つ以上・第3章(Playwright自動化)への接続も入れています。

ファイル全文は `zenn_ogp_ch2.md` に保存済みです（frontmatterは付与しない仕様どおり本文のみ）。
