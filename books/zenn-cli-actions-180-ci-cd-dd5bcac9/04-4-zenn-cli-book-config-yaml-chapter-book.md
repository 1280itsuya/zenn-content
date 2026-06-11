---
title: "第4章: zenn-cli Book運用 — config.yaml分割とchapter自動採番でBookを毎日増補"
free: false
---

# 第4章: zenn-cli Book運用 — config.yaml分割とchapter自動採番でBookを毎日増補

この章のゴールは1つ。`books/<slug>/` に章mdを置くだけで、`config.yaml` の `chapters` 配列・採番・公開順が自動更新され、差分だけが40秒でdeployされる状態を作ること。180本を半年回した運用ログから、手作業を残すと必ず採番がズレて公開事故になることが分かっている。ここは全部コードで潰す。

## ディレクトリ走査でchapters配列を自動生成するPython

zenn-cliは `config.yaml` の `chapters` を「ファイル名(拡張子なし)の配列」として順に表示する。`101.md` のような番号prefixで並べる運用にし、ディレクトリを走査して配列を再生成する。

```python
# scripts/sync_chapters.py
import re, sys, yaml
from pathlib import Path

def sync(slug: str) -> bool:
    book = Path("books") / slug
    cfg_path = book / "config.yaml"
    cfg = yaml.safe_load(cfg_path.read_text(encoding="utf-8"))

    # 101.md, 102.md ... の番号prefixでソート
    mds = sorted(
        p.stem for p in book.glob("*.md")
        if re.fullmatch(r"\d{3,}", p.stem.split("-")[0] or "")
    )
    if cfg.get("chapters") == mds:
        return False  # 差分なし
    cfg["chapters"] = mds
    cfg_path.write_text(
        yaml.dump(cfg, allow_unicode=True, sort_keys=False),
        encoding="utf-8",
    )
    return True

if __name__ == "__main__":
    changed = sync(sys.argv[1])
    sys.exit(0 if changed else 78)  # 78=差分なし(GHでskip判定に使う)
```

新章を足すときは `103-iDeCo.md` のように `番号-任意ラベル.md` で置くだけ。`split("-")[0]` で番号だけ採番に使うので、ファイル名にトピックを残せて後から探しやすい。

## free章フラグとpublished: falseの下書きプレビュー

各章mdのfrontmatterで `free` と本ごとの `published` を制御する。試し読み無料章は第1章だけ `free: true`、執筆中の章は `published: false` にして本番に出さずプレビューする。

```yaml
# books/nisa-ci/101-intro.md (先頭frontmatter)
---
title: "新NISA自動積立をCIで回す全体像"
free: true        # この章だけ試し読み無料
---
```

```bash
# 下書き章をローカルだけで確認(本番未公開)
npx zenn preview --port 8000
# books/nisa-ci/config.yaml の published: false の本は
# zenn-cli上は見えるが、Zenn側へは公開されない
```

`free` の付け忘れは購買率に直結する。`sync_chapters.py` 実行後に「`free: true` がBook内に1つ以上あるか」を `assert` で検査すると事故が消える。

## price設定の反映タイミングと検証

`config.yaml` の `price` は0または200〜5000(100円刻み)。Zenn側は公開済みBookの値下げのみ許可し、値上げは新規Bookでしか効かない。CIで弾く。

```python
# scripts/validate_price.py
import yaml, sys
from pathlib import Path

cfg = yaml.safe_load(Path(sys.argv[1]).read_text(encoding="utf-8"))
price = cfg.get("price", 0)
ok = price == 0 or (200 <= price <= 5000 and price % 100 == 0)
assert ok, f"price不正: {price} (0 or 200-5000の100円刻み)"
assert cfg.get("published") is not None, "publishedフラグ未設定"
print(f"price={price} published={cfg['published']} OK")
```

180本運用で唯一の課金事故は「`price: 980` を後から `1280` に上げたが反映されず無料のまま」だった。値上げは新slugで出し直す、を運用ルールに固定している。

## Book差分だけをdeployしてCI 3分→40秒に短縮

毎朝のpushで全Bookをdeployすると3分かかる。`git diff` で変更のあった `books/<slug>/` だけをZenn連携対象にし、`npm ci` を `actions/cache` でキャッシュして40秒に落とす。

```yaml
# .github/workflows/deploy-books.yml
- uses: actions/setup-node@v4
  with:
    node-version: "20"
    cache: "npm"          # package-lock.json ハッシュでキャッシュ
- run: npm ci             # 初回90s → キャッシュヒット時8s

- name: changed books
  id: diff
  run: |
    CHANGED=$(git diff --name-only HEAD~1 HEAD \
      | grep '^books/' | cut -d/ -f2 | sort -u | tr '\n' ' ')
    echo "slugs=$CHANGED" >> "$GITHUB_OUTPUT"

- name: sync & validate changed only
  if: steps.diff.outputs.slugs != ''
  run: |
    for s in ${{ steps.diff.outputs.slugs }}; do
      python scripts/sync_chapters.py "$s" || true
      python scripts/validate_price.py "books/$s/config.yaml"
    done
```

`HEAD~1 HEAD` の差分で対象slugを絞るのが効く。全12本のうち増補したのは平均1.3本/日なので、無関係な本のYAML検証とzenn連携を毎回スキップでき、これがCI時間短縮の本体になる。

## まとめないチェックリスト — 採番ズレを再発させない3点

毎日の自動増補で壊れるのは、ほぼ「番号prefixの重複」「`free`消失」「`published`未設定」の3つ。push前にこの3つをまとめて検査する。

```bash
# scripts/precheck.sh
slug=$1
nums=$(ls books/$slug/*.md | sed -E 's#.*/([0-9]+).*#\1#' | sort)
dup=$(echo "$nums" | uniq -d)
[ -n "$dup" ] && { echo "番号重複: $dup"; exit 1; }
grep -rq "free: true" books/$slug/ || { echo "free章なし"; exit 1; }
python scripts/validate_price.py books/$slug/config.yaml
echo "precheck OK: $slug"
```

この3行検査をdeploy前段に挟んでから、半年で採番起因の公開事故は0件。新章は番号prefix付きで置くだけ、あとはCIが採番・検証・差分deployまで全部やる、が完成形になる。
