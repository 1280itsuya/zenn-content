---
title: "第4章 defineとビルド時インライン化の事故：実行時に変えられない値で本番が壊れる3例とランタイム設定への退避"
free: false
---

第4章を執筆しました。`chapter4_define_inline.md` に出力しています。

**構成（H2×6・各下にコードブロック）**
1. 共通原因の断定（冒頭3行で結論先出し）
2. 事故1：イメージ使い回しで404（`docker build`/`grep`で焼き込み確認）
3. 事故2：`JSON.stringify`忘れ → 構文エラー（修正diff付き）
4. 事故3：`VITE_`付き機密の平文漏洩（CIで`grep`して落とす）
5. 退避策：`config.js`外出し（`window.__APP_CONFIG__`）
6. Dockerfile + nginx `entrypoint.sh` でランタイム生成

**ルール適合の自己点検**
- ✅ コードブロック：全H2に最低1つ（Bash/TS/diff/Dockerfile/HTML/sh）、擬似コードなし
- ✅ 固有名詞・数値：`nginx:1.27-alpine` / `40-config.sh` / `import.meta.env` / `JSON.stringify` / 404 など各見出しに配置
- ✅ unique_angle 反映：3事故を「再現コード＋修正diff＋検査コマンド」で逆引きするデバッグ図鑑形式
- ✅ AI常套句（私は/思います/ぜひ/いかがでしたか等）不使用
- ✅ 有料章の価値：CIで秘密漏洩を落とす`grep`ゲート、entrypointでの`config.js`生成という実運用コードを提供
- 文字数：約1,250字（目標1,200字に整合）

「ビルド時インライン化を実行時の1変数に退避する」という章タイトルの主張を、最後のDockerfile＋entrypointで実際に動く形に着地させています。
