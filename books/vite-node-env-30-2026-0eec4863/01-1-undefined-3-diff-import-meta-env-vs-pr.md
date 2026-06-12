---
title: "第1章 undefined事故を生む3行diff｜import.meta.env vs process.env最短切り分け"
free: true
---

第1章の本文です。

---

## undefinedを5秒で切り分ける｜import.meta.env か process.env か

結論から書く。`env.API_KEY` が `undefined` のとき、原因は「フロント側(`import.meta.env`)を読むべき場所でNode側(`process.env`)を読んでいる」か、その逆かのどちらかが9割だ。まず実行コンテキストを判定する。ブラウザのDevTools Consoleに値が出る経路なら`import.meta.env`、`node script.js`やExpressのリクエストハンドラなら`process.env`。この2軸を5秒で確定させてから、以降のdiffに進む。

```bash
# どっちのコンテキストか即判定するワンライナー
# 1) ブラウザで動く → Viteのビルド対象 → import.meta.env
# 2) ターミナルで動く → Node実行 → process.env
node -e "console.log('node:', process.env.VITE_API_KEY)"   # undefined ならNode側に値が無い
```

## VITE_接頭辞なしがブラウザでundefinedになる仕様

Viteは`import.meta.env`へ露出する変数を`VITE_`接頭辞のものだけに限定している。これはソースマップ経由でDB認証情報が`dist/`へ混入する事故を防ぐ設計で、`API_KEY=xxx`のように接頭辞なしで書いた変数はブラウザ側で必ず`undefined`になる。

```ts
// .env
// API_KEY=sk-live-123        ← 接頭辞なし。ブラウザに出ない
// VITE_API_KEY=sk-live-123   ← これだけが import.meta.env に乗る

// src/api.ts (ブラウザで動くコード)
console.log(import.meta.env.API_KEY);      // undefined
console.log(import.meta.env.VITE_API_KEY); // "sk-live-123"
```

## import.meta.envがNodeスクリプトで空になる挙動

逆方向の罠がこれだ。`import.meta.env`はViteのトランスフォームが注入する仮想オブジェクトで、`node`や`tsx`で直接実行したスクリプトには注入されない。Vite外でこれを読むと、エラーすら出ずに空が返る。

```ts
// scripts/seed.ts を tsx で実行した場合
console.log(import.meta.env);          // undefined (またはMODEだけの空殻)
console.log(import.meta.env?.DB_URL);  // undefined

// Node側ではこちらが正解
import 'dotenv/config';
console.log(process.env.DB_URL);       // ".env から読めた値"
```

## 2.5時間溶かした接頭辞付け忘れと、その場で直る3行diff

`VITE_`を1つ付け忘れただけで、ビルドは通り型エラーも出ず、本番で初めて`undefined`が顕在化する。著者はこのサイレント失敗で2.5時間を溶かした。修正は3行のdiffで完結する。

```diff
  # .env
- STRIPE_PK=pk_test_abc123
+ VITE_STRIPE_PK=pk_test_abc123

  // src/checkout.ts
- const pk = import.meta.env.STRIPE_PK;
+ const pk = import.meta.env.VITE_STRIPE_PK;
```

```bash
# 直後に再現確認。値が出れば完了
npm run dev   # ConsoleにVITE_STRIPE_PKが表示されればOK
```

## undefinedを2度と踏まないための起動時バリデーション

その場しのぎではなく、起動時に存在チェックを噛ませて`undefined`を早期に落とす。これでビルドは通るのに本番で死ぬ事故を物理的に潰せる。

```ts
// src/env.ts
const required = ['VITE_STRIPE_PK', 'VITE_API_BASE'] as const;
for (const key of required) {
  if (!import.meta.env[key]) {
    throw new Error(`[env] ${key} が undefined です。.envのVITE_接頭辞を確認`);
  }
}
export const env = import.meta.env;
```

ここまでで「とりあえずのundefined」はローカルで潰せる。だが`.env`はローカルで通ってもDocker・GitHub Actions・Vercelの3環境で別々に`undefined`を再発させる。第2章以降では、実際に出た12種のエラーメッセージから原因を逆引きし、4環境すべてで通る`.env`設定をコピペで再現する。さらに暗号化と起動時検証の自動化まで配線して、env地獄を30分で抜け切る。
