---
title: "第2章 zenn-cli + GITHUB_TOKENのpermissions: contents: write — push 403を10分で消す最小権限設定"
free: false
---

## 結論: 403を消すのはpermissions 1行 + checkout + git configの3点だけ

自動pushの403 Permission deniedは、`GITHUB_TOKEN`が読み取り専用で渡るのが原因。下の3行を入れれば10分で通る。9割が「PAT(Personal Access Token)を発行し直す」遠回りをするが不要だ。

```yaml
permissions:
  contents: write   # これが無いと push は必ず 403
```

実際に出るログはこれ。`contents: write`を足すだけで消える。

```text
remote: Permission to user/repo.git denied to github-actions[bot].
fatal: unable to access 'https://github.com/...': The requested URL returned error: 403
```

## actions/checkout@v4 の fetch-depth と token を正しく渡す

`actions/checkout`はデフォルトで`GITHUB_TOKEN`を埋め込むが、`fetch-depth: 0`を付けないと差分pushで`shallow update not allowed`が出る。検証では`fetch-depth`未指定で3回連続失敗した。

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0
    token: ${{ secrets.GITHUB_TOKEN }}
    persist-credentials: true
```

## git config user.name 未設定の "empty ident" を潰す

bot実行ではcommit者情報が空で、`*** Please tell me who you are`で止まる。固定の2行で解決する。

```bash
git config user.name "github-actions[bot]"
git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
git add articles/
git commit -m "auto: $(date +%F) Claude生成記事" || echo "no changes"
git push origin main
```

## zenn-cli で slug 自動採番と frontmatter を埋める

`npx zenn-cli@latest new:article`はslugを自動生成するが、CIでは`--slug`に14桁日付を渡して衝突を防ぐ。`published: true`と`type: tech`を後段の`sed`で確実に書き換える。

```bash
SLUG="claude-$(date +%y%m%d%H%M%S)"   # 例: claude-260604071230
npx zenn-cli@latest new:article --slug "$SLUG"
F="articles/$SLUG.md"
sed -i 's/^published:.*/published: true/' "$F"
sed -i 's/^type:.*/type: "tech"/' "$F"
# published_at は予約投稿時のみ。即時公開なら frontmatter から削除する
sed -i '/^published_at:/d' "$F"
```

`published_at`を未来日時で残すとZenn側で下書き扱いになり公開されない。即時pushでは削除が正解だ。

## ANTHROPIC_API_KEY を登録しログ露出を mask で防ぐ

キーはSettings → Secrets and variables → Actions → New repository secretで`ANTHROPIC_API_KEY`として登録。ログへのecho混入を防ぐため`add-mask`を明示する。

```yaml
- name: Generate with Claude
  env:
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
  run: |
    echo "::add-mask::$ANTHROPIC_API_KEY"
    python scripts/generate.py   # claude-opus-4-8 を呼ぶ
```

`add-mask`を入れた後は、誤って`env | grep ANTHROPIC`してもログ上は`***`に置換される。ブラウザのGitHub画面だけで全設定が完結し、ローカル環境構築は1コマンドも要らない。
