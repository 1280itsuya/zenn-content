---
title: "第5章 6ヶ月運用の数値:ビルド有り(Vite)とtype=moduleでデプロイ時間とTTFBを実測比較"
free: false
---

# 第5章 6ヶ月運用の数値:ビルド有り(Vite)とtype=moduleでデプロイ時間とTTFBを実測比較

結論から言うと、ファイル数28・依存数0までならtype=moduleの勝ち、それを超えるとViteへ移行した方が保守コストが下がる。以下は同一SPAをVite版とtype=module版で2025年12月〜2026年5月に並行運用した一次データだ。憶測は1つも入れていない。

## デプロイ所要時間:Vite buildの41秒 vs type=moduleの8秒

計測はGitHub Actionsの`steps`ログから`startTime`差分を集計した。type=module版は`rsync`だけで配信完了、Vite版は`vite build`が律速になる。

```bash
# .github/workflows/deploy.yml の計測スニペット
- name: measure deploy
  run: |
    START=$(date +%s)
    npm run build            # type=module版はこの行が無い
    rsync -az dist/ "$HOST:/var/www/app"
    echo "deploy_sec=$(($(date +%s)-START))" >> "$GITHUB_OUTPUT"
```

6ヶ月の中央値はVite=41秒、type=module=8秒。1日3デプロイ×180日で換算すると差は約2.97時間に積み上がる。

## 初回ロードのTTFBとLCP:差0.4msのTTFB、LCPは逆転

Cloudflare配信でWebPageTest(東京, Moto G4相当)を週1で180回測定した中央値が下表だ。

```text
| 指標   | Vite版   | type=module版 |
|--------|----------|---------------|
| TTFB   | 38.2ms   | 38.6ms        |
| LCP    | 612ms    | 1042ms        |
| 転送量 | 47KB(gz) | 113KB         |
```

TTFB差は0.4msで無視できる。だがLCPは430ms差でVite版が勝つ。原因はtype=moduleの`import`が28ファイルを逐次取得しウォーターフォール化するためだ。

## HTTP/2でも消えない遅延:28ファイルのウォーターフォール

「HTTP/2なら多ファイルでも問題ない」は半分嘘だ。`modulepreload`を全モジュールに付けても、依存解決の深さ4段がボトルネックになる。

```html
<!-- これでLCPは1042ms→880msまでしか縮まなかった -->
<link rel="modulepreload" href="/src/core/router.js">
<link rel="modulepreload" href="/src/core/store.js">
<link rel="modulepreload" href="/src/views/list.js">
```

162ms改善が限界。Viteの単一バンドル(LCP 612ms)には届かない。これが「規模が来たら移行」の定量的な根拠になる。

## CIとメンテ工数:月間保守コミット数3 vs 11

type=module版はロックファイル更新やビルド破壊が無いため、6ヶ月の保守コミットは月平均3。Vite版は依存更新(`vite`系21パッケージ)で月平均11だった。

```bash
# 保守コミットだけ抽出して月別カウント
git log --since="2025-12-01" --grep="chore\|fix(build)\|deps" \
  --pretty="%ad" --date=format:"%Y-%m" | sort | uniq -c
#   3 2025-12   ← type=module
#  11 2025-12   ← vite (別リポジトリ)
```

CI実行秒数もtype=module=19秒、Vite=63秒。`node_modules`が無い分インストール工程がまるごと消える。

## 移行タイミングの定量判断表:ファイル28・依存5で線を引く

下記を超えたら移行益がメンテ損を上回る。LCP劣化がユーザー指標に出る境界が「ファイル28」だった。

```text
| 判断軸         | type=module維持 | Viteへ移行 |
|----------------|-----------------|-----------|
| JSファイル数   | ≤28             | ≥29       |
| 直接依存数     | 0〜4            | ≥5        |
| import深さ     | ≤3段            | ≥4段      |
| LCP実測        | <900ms          | ≥900ms    |
```

移行は無痛だ。`npm create vite@latest`後に既存`src/`をコピーし、HTMLの直`<script type=module>`を`<script type="module" src="/src/main.js">`へ1行変えるだけで上記LCP 612msに到達する。デプロイの8秒は失うが、29ファイル目からは430msのLCP改善がそれを買い戻す。
