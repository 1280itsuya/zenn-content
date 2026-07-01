---
title: "月10万円への拡張ロードマップ——パイプラインをGumroad商品・Booth教材・記事代行に派生させる"
free: false
---

## 月10万円への拡張ロードマップ——パイプラインをGumroad商品・Booth教材・記事代行に派生させる

ここまでで「記事を自動生成して自分のメディアに投稿する」エンジンが動いている。同じコードを3方向に派生させると、収益の天井が大きく上がる。優先度の高い順に解説する。

---

## ルート①　記事代行（最速・最小投資）

**月収見込み：¥30,000〜¥80,000**

クライアントからキーワードとターゲット情報を受け取り、自動生成した記事を納品する。自分のメディアに投稿するのと手順は同じで、送り先が変わるだけだ。

```python
# client_service.py の核心部分
def generate_for_client(client_id: str, keyword: str, target: str) -> str:
    prompt = f"""
    ターゲット: {target}
    キーワード: {keyword}
    トーン: プロフェッショナル・親しみやすい
    文字数: 2000字以上
    """
    article = call_claude(prompt)  # 既存の生成関数をそのまま流用
    save_to_client_dir(client_id, article)
    return article
```

落とし穴は**クライアントごとのブランドトーンのズレ**だ。プロンプトにクライアント固有の「禁止ワード」と「文体サンプル」を渡すフィールドを設けると差し戻しが激減する。ランサーズ・ランサーブとの組み合わせで1記事¥3,000〜¥5,000が相場。月20本で¥60,000〜¥100,000に届く計算になる。

---

## ルート②　Gumroadデジタル商品（自動・複利型）

**月収見込み：¥5,000〜¥30,000（累積型）**

生成パイプライン自体をZIPで配布する。「動くコードを買えば明日から使える」という価値設計が重要で、ドキュメントは最小限でよい。

```python
# gumroad_upload.py
import requests, os

def upload_product(zip_path: str, title: str, price_cents: int):
    headers = {"Authorization": f"Bearer {os.environ['GUMROAD_TOKEN']}"}
    with open(zip_path, "rb") as f:
        requests.post(
            "https://api.gumroad.com/v2/products",
            headers=headers,
            data={"name": title, "price": price_cents},
            files={"file": f},
        )
```

**落とし穴**：Gumroadは支払い方法が未接続だと商品が「下書き」のまま公開されない。アカウント設定の「Payouts」でStripe接続を先に済ませること。価格は¥980〜¥2,480が手に取られやすい。

---

## ルート③　Booth教材（中期・ブランド構築）

**月収見込み：¥10,000〜¥40,000**

Gumroadより日本ユーザーが多く、クリエイター文化との相性が良い。「コード＋解説PDF」のセット販売が効く。PDFはパイプラインで自動生成できる。

```python
# booth_product_builder.py
def build_pdf_from_md(md_path: str, output_path: str):
    import subprocess
    subprocess.run([
        "pandoc", md_path,
        "-o", output_path,
        "--pdf-engine=xelatex",
        "-V", "CJKmainfont=IPAexMincho",
    ], check=True)
```

落とし穴は**フォント設定**。日本語PDFをxelatexで出すにはIPAexフォントのインストールが必要で、CI/CD環境では要確認。教材にGitHubリポジトリURLを同梱し「コードはGitHubで管理・更新」とすると価値が下がらずリピート購入にも繋がる。

---

## 3ルートの優先順位と行動順序

| 順序 | ルート | 初動工数 | 月収到達の速さ |
|------|--------|----------|----------------|
| 1 | 記事代行 | 低（既存コードを流用） | 最速（受注即収益） |
| 2 | Gumroad商品 | 中（ZIP整備+支払い設定） | 中（累積型） |
| 3 | Booth教材 | 中（PDF生成+商品ページ） | 中〜長期 |

まず記事代行で**実績と月3〜5万円のキャッシュフロー**を作り、その実績をもとにGumroad・Booth商品の信頼性を高める順序が最短だ。3ルートを同時に立ち上げようとすると全部が中途半端になる。記事代行→Gumroad→Boothの順で1ヶ月ずつ仕込むのが現実的なペース配分になる。
