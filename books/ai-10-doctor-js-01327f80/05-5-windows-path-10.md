---
title: "第5章 Windows固有の文字化け・PATH・実行ポリシー10項目を確定"
free: false
---

## checkWindows() で cp932 文字化けを chcp 65001 で潰す

Windows の既定コードページ 932 (cp932) では、Node の `console.log` に渡した日本語が `??` 化する。`chcp` を読んで 65001 (UTF-8) 以外なら赤判定にする。

```js
const { execSync } = require("child_process");
function checkEncoding() {
  const cp = execSync("chcp").toString().match(/\d+/)[0];
  return {
    axis: "windows",
    ok: cp === "65001",
    fix: "chcp 65001  # PowerShell プロファイルに追記して永続化",
  };
}
```

## path.join 未使用の \/ 混在と 260 字超パスを fs で検出

`"src" + "/" + name` 直書きは `\` と `/` が混ざり、260 字の MAX_PATH 制限にも当たる。`process.cwd()` の長さを実測する。

```js
const path = require("path");
function checkPathLimit() {
  const full = path.join(process.cwd(), "node_modules", ".bin", "playwright.cmd");
  return {
    axis: "windows",
    ok: full.length < 260,
    fix: 'New-ItemProperty "HKLM:\\SYSTEM\\CurrentControlSet\\Control\\FileSystem" LongPathsEnabled -Value 1',
  };
}
```

## Set-ExecutionPolicy RemoteSigned 未設定を PowerShell で判定

既定 `Restricted` だと `npm run` が `.ps1` を実行できず無言で死ぬ。現在ポリシーを取得して照合する。

```js
function checkExecPolicy() {
  const p = execSync("powershell -Command Get-ExecutionPolicy").toString().trim();
  return {
    axis: "windows",
    ok: ["RemoteSigned", "Bypass", "Unrestricted"].includes(p),
    fix: "Set-ExecutionPolicy -Scope CurrentUser RemoteSigned",
  };
}
```

## 第2〜4章の check を全結合し exit code を返す

第2章 nodejs・第3章 playwright/dotenv・第4章 automation の check を 1 配列に束ね、赤が 1 件でも `process.exit(1)` で CI を落とす。

```js
const checks = [
  ...require("./ch2"), ...require("./ch3"), ...require("./ch4"),
  checkEncoding, checkPathLimit, checkExecPolicy,
];
const results = checks.map((f) => f());
const red = results.filter((r) => !r.ok);
```

## 緑8/赤2＋修正コマンドを Markdown 出力

結果を `緑 ${ok}/赤 ${ng}` のサマリと修正コマンド表に整形し、`doctor.md` として保存する。

```js
const ok = results.length - red.length;
let md = `# 診断結果: 緑 ${ok} / 赤 ${red.length}\n\n`;
red.forEach((r) => (md += `- [${r.axis}] \`${r.fix}\`\n`));
require("fs").writeFileSync("doctor.md", md);
process.exit(red.length ? 1 : 0);
```

赤 2 件なら終了コード 1、修正コマンド 2 行が `doctor.md` に並ぶ。コピペして実行→再診断で緑 10 になる。

## 10項目チェックリストと5判定軸の対応表

各 check の `axis` は下表の 5 スラッグに必ず一致する。doctor.js の出力と本書のトピックを 1:1 で突き合わせられる。

| # | チェック項目 | 判定軸 (axis) |
|---|---|---|
| 1 | Node 18+ / npm 解決 | `nodejs` |
| 2 | npx 実行パス | `nodejs` |
| 3 | Playwright ブラウザ導入 | `playwright` |
| 4 | headless 起動 | `playwright` |
| 5 | .env 読み込み | `dotenv` |
| 6 | APIキー空欄検出 | `dotenv` |
| 7 | chcp 65001 (cp932回避) | `windows` |
| 8 | MAX_PATH 260字 | `windows` |
| 9 | ExecutionPolicy | `windows` |
| 10 | cron/タスク登録 | `automation` |

topics: `nodejs` / `playwright` / `dotenv` / `windows` / `automation`
