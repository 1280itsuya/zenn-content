---
title: "tsconfig strictNullChecksで大量エラーが出る"
emoji: "⚙️"
type: "tech"
topics: ["JavaScript", "TypeScript", "設定"]
published: true
---

## 発生条件

- `tsconfig.json` に `"strict": true` を追加、または `"strictNullChecks": true` を明示したとき
- 既存のプロジェクトに TypeScript を後から導入し、JavaScript から移行した `.ts` / `.tsx` ファイルが混在しているとき
- `null` や `undefined` を返す可能性のある変数を、チェックなしで使用しているコードが多数存在するとき

## 原因

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true
  }
}
```

`"strict": true` を有効にすると、内部で `"strictNullChecks": true` が自動的にオンになる。この設定が有効な状態では、`null` と `undefined` は他の型と互換性がなくなる。

たとえば以下のコードはエラーになる。

```typescript
function getUsername(user: { name: string } | null): string {
  return user.name; // Error: Object is possibly 'null'
}

const input = document.getElementById("email");
console.log(input.value); // Error: Object is possibly 'null'
```

`strictNullChecks` が無効のとき、TypeScript は `null` と `undefined` をすべての型に代入可能として扱うため、上記のコードはエラーにならない。有効化すると、`null` や `undefined` になり得る値へのアクセスはすべて型エラーとして検出される。既存コードが「`null` チェックなしで使う」前提で書かれていた場合、一度に大量のエラーが表面化する。

## 修正方法

### 段階的に有効化する（推奨）

既存プロジェクトへの導入時は、`strict` を一括でオンにせず個別フラグで制御する。

```json
// tsconfig.json
{
  "compilerOptions": {
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true
    // "strictPropertyInitialization": true  ← 後から追加
  }
}
```

`strictNullChecks` だけ先に有効にして対応し、他のフラグは順番に追加していく。

### コード側の修正パターン

```typescript
// nullチェックを明示する
function getUsername(user: { name: string } | null): string {
  if (user === null) {
    return "anonymous";
  }
  return user.name;
}

// オプショナルチェーン演算子を使う
const input = document.getElementById("email");
console.log(input?.value);

// Non-null アサーション演算子（確実に存在すると保証できる場合のみ）
const form = document.getElementById("login-form")!;
```

Non-null アサーション演算子（`!`）は、型エラーを黙らせるだけで実行時の安全性は担保しない。本当に `null` にならないと確信できる箇所にのみ使う。

### 新規プロジェクトの場合

プロジェクト立ち上げ時から `"strict": true` を設定しておく。後から有効にするより、最初から有効な状態でコードを書くほうが修正コストがかからない。

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "strict": true,
    "moduleResolution": "bundler"
  }
}

---

## JS/TS設定エラーをAIが30秒で根本診断
→ [診断キット](https://zenn.dev/itsuya_auto/books/jsconfig-diag-kit)
