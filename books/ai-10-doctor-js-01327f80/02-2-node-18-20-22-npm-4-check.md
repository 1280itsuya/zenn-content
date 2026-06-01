---
title: "第2章 Node 18/20/22とnpmのバージョン地雷4種をcheckで弾く"
free: false
---

## ERR_REQUIRE_ESM / Unexpected token / engines警告——3エラーの実物ログ

結論から言うと、AI副業自動化スクリプトが起動しない原因の最頻出はNodeランタイム不一致で、症状は次の3つに収束する。Playwright投稿botやdotenv読み込みが動かない時、まずこのログと照合する。

```bash
# 症状A: Node 16残存でESM依存を読むと出る
Error [ERR_REQUIRE_ESM]: require() of ES Module node-fetch/src/index.js
# 症状B: 古いランタイムでtop-level awaitやoptional chaining
SyntaxError: Unexpected token '?'
# 症状C: package.json engines制約に届かない
npm warn EBADENGINE Unsupported engine { required: { node: '>=18' }, current: { node: 'v16.20.2' } }
```

3つとも「Nodeが古い／切替漏れ」が9割で、残り1割がpackage.json設定漏れだ。

## process.version.match()でメジャー番号を抽出するcheckNode()

doctor.jsの中核。`process.version`（例 `v16.20.2`）からメジャー番号だけ抜き、18未満なら赤、CIで使う20と差があれば黄を返す。

```javascript
// doctor.js — checkNode(): nodejs診断ブロック
function checkNode() {
  const major = Number(process.version.match(/^v(\d+)\./)[1]);
  const CI_TARGET = 20; // GitHub Actionsと揃える基準値
  if (major < 18) {
    return { tag: "nodejs", color: "🔴",
      msg: `Node ${major} は古い。nvm install 20.11.1 を実行` };
  }
  if (major !== CI_TARGET) {
    return { tag: "nodejs", color: "🟡",
      msg: `Node ${major}。CIは${CI_TARGET}。nvm use ${CI_TARGET} 推奨` };
  }
  return { tag: "nodejs", color: "🟢", msg: `Node ${major} OK` };
}
console.log(checkNode());
```

赤・黄の時点で「次に叩くコマンド」を文字列で同梱するのが、コピペ即実行型doctorの肝だ。

## nvm-windows install 20.11.1 → nvm use の正しい順序

windows環境でやりがちな失敗が、`install`せずに`use`を叩いて切替漏れになるパターン。順序を固定する。

```powershell
nvm install 20.11.1   # 先にバイナリ取得（数十秒）
nvm use 20.11.1       # その後で有効化
node -v               # v20.11.1 を必ず目視確認
```

`nvm use`後に`node -v`が変わらなければPATH問題（後述）。automationを毎朝7時にタスク実行する場合、タスク起動シェルでも`node -v`を1行ログに残すと夜間の沈黙failを潰せる。

## npm ls -g --depth=0 でグローバル重複を1行検出

npmグローバル版ズレは、旧Node時代の`zenn-cli`や`playwright`が別パスに残ると起きる。`--depth=0`で一覧化し重複を発見する。

```bash
npm ls -g --depth=0
# /Users/.../v16/... と v20/... の両方に zenn-cli が出たら重複
where.exe npm   # windowsでnpm本体の解決順を確認（複数行=黄信号）
npm root -g     # 今アクティブなグローバルルートを確定
```

doctor.jsからは`execSync("npm ls -g --depth=0")`で取り込み、`playwright`が2系統あれば赤を返す判定を足す。

## type:module / commonjs の選択基準とwindows PATH優先順位修正

`Unexpected token`の最後の1割はpackage.jsonの`type`未指定。`import`を使うなら`module`、`require`主体なら省略（commonjs）に倒す。

```jsonc
// dotenvをimportで読むなら module 一択
{ "type": "module", "engines": { "node": ">=18" } }
```

それでも`nvm use`が効かないwindowsでは、システムPATHに残る`C:\Program Files\nodejs`が`%NVM_SYMLINK%`より上に来ているのが原因。優先順位を入れ替える。

```powershell
# ユーザーPATHのnvm symlinkを最優先へ
$nvm = "$env:NVM_SYMLINK"
$old = [Environment]::GetEnvironmentVariable("Path","User")
[Environment]::SetEnvironmentVariable("Path", "$nvm;$old", "User")
# 再起動後 where.exe node の1行目がnvm配下なら修正成功
```

これでcheckNode()が🟢を返せば、第5章のチェックリスト（`nodejs / playwright / dotenv / windows / automation`）のうちランタイム項目を確実に通過できる。
