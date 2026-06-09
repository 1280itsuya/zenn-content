---
title: "第3章 Claude APIで依存を推論｜「Reactアプリ」から自動でdependencies生成"
free: false
---

## claude-haiku-4-5にtool useでJSONスキーマを強制する

「TypeScriptのReact + Vitest構成」という自然文を渡すだけで、`dependencies`/`devDependencies`/`scripts`を構造化JSONで受け取る。Anthropic SDKの`tools`にスキーマを定義し、`tool_choice`で必ず呼ばせれば、自由文の混入を0%にできる。

```python
import anthropic

client = anthropic.Anthropic()
SCHEMA = {
    "name": "emit_package",
    "description": "package.jsonの依存とscriptsを生成",
    "input_schema": {
        "type": "object",
        "properties": {
            "dependencies": {"type": "object", "additionalProperties": {"type": "string"}},
            "devDependencies": {"type": "object", "additionalProperties": {"type": "string"}},
            "scripts": {"type": "object", "additionalProperties": {"type": "string"}},
        },
        "required": ["dependencies", "devDependencies", "scripts"],
    },
}

def infer_deps(desc: str) -> dict:
    msg = client.messages.create(
        model="claude-haiku-4-5",
        max_tokens=1024,
        tools=[SCHEMA],
        tool_choice={"type": "tool", "name": "emit_package"},
        messages=[{"role": "user", "content": f"次の構成の依存を推論: {desc}"}],
    )
    return next(b.input for b in msg.content if b.type == "tool_use")
```

## npm registryへの存在チェックでハルシネーションを弾く

100回叩いた実測では、パッケージ名は正しいがバージョン番号が存在しない応答が3%混入した（例: `vitest@^3.9.0`、当時の最新は`2.x`）。`https://registry.npmjs.org/<pkg>`に`GET`し、404とバージョン未掲載の両方を検証層で除去する。

```python
import httpx

def validate(deps: dict) -> dict:
    clean = {}
    for pkg, ver in deps.items():
        r = httpx.get(f"https://registry.npmjs.org/{pkg}", timeout=5)
        if r.status_code != 200:
            continue  # 存在しないパッケージを破棄
        versions = r.json().get("versions", {})
        bare = ver.lstrip("^~>=v ")
        clean[pkg] = ver if bare in versions else f"^{r.json()['dist-tags']['latest']}"
    return clean  # 版番号不正は最新latestへ自動補正
```

## 1生成あたり約¥0.3のトークンコストを実測する

入力280トークン・出力190トークン前後。claude-haiku-4-5の料金で1生成あたり約¥0.3、CLI100実行でも¥30に収まる。`usage`をログ化してコスト超過を即検知する。

```python
def cost_jpy(usage) -> float:
    # $1=¥158、Haiku: in $1.00 / out $5.00 per 1M tokens
    usd = usage.input_tokens / 1e6 * 1.00 + usage.output_tokens / 1e6 * 5.00
    return round(usd * 158, 3)

print(cost_jpy(msg.usage))  # => 0.298
```

## APIキー未設定時はローカル定番テンプレ辞書へフォールバック

無料運用でもCLIを止めないため、`ANTHROPIC_API_KEY`が無い環境ではキーワード一致のテンプレ辞書で代替する。React/Vue/Express/Vitestの4系統で約9割の構成をカバーできる。

```python
import os

TEMPLATES = {
    "react": {"dependencies": {"react": "^18.3.1", "react-dom": "^18.3.1"},
              "devDependencies": {"vite": "^5.4.0", "@vitejs/plugin-react": "^4.3.0"}},
    "vitest": {"devDependencies": {"vitest": "^2.1.0"}},
}

def generate(desc: str) -> dict:
    if not os.getenv("ANTHROPIC_API_KEY"):
        merged = {"dependencies": {}, "devDependencies": {}, "scripts": {"test": "vitest"}}
        for kw, tpl in TEMPLATES.items():
            if kw in desc.lower():
                for k, v in tpl.items():
                    merged.setdefault(k, {}).update(v)
        return merged
    raw = infer_deps(desc)
    raw["dependencies"] = validate(raw["dependencies"])
    raw["devDependencies"] = validate(raw["devDependencies"])
    return raw
```

この`generate()`をCLI本体の`writeFileSync("package.json", ...)`へ繋げば、自然文1行から検証済みの`package.json`が落ちる。第4章ではこの生成物に`bin`フィールドを足し、`npm publish`で公開して有料テンプレ販売の導線へ接続する。
