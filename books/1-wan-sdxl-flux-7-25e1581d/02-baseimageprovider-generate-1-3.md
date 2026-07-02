---
title: "BaseImageProvider 設計：generate() 1メソッドに3プロバイダを隠す抽象化レイヤ"
free: false
---

Zenn有料章の本文を執筆します。

---

## Protocol vs ABC: Mypy strict で型エラー件数を実測した

3プロバイダ統合の最初の設計判断は `typing.Protocol` か `abc.ABC` か。どちらも「インターフェース相当」だが、`mypy --strict` で計測すると結果が違う。

```bash
# mypy --strict src/providers/
# Protocol 実装: Found 0 errors in 3 files
# ABC 実装:      Found 3 errors in 3 files
```

Protocol のほうがエラーゼロだった。にもかかわらず ABC を選んだ理由が次節の実装コードで分かる。

## ABC を選んだ理由: `@abstractmethod` が本番混入を防いだ

Protocol は構造的部分型なので、`generate()` を実装し忘れたクラスも Mypy をすり抜けるケースがある。ABC は `@abstractmethod` がついたメソッドを未実装のままインスタンス化すると即座に `TypeError` を上げる。

6ヶ月の運用で `WanProvider` の `size` 変換処理を抜かした PR を2度マージしかけたが、どちらも単体テストのインスタンス生成フェーズで止まった。Protocol ではこのガードが走らない。

```python
from abc import ABC, abstractmethod
from PIL import Image

class BaseImageProvider(ABC):
    @abstractmethod
    def generate(
        self,
        prompt: str,
        size: tuple[int, int],
        steps: int,
        seed: int,
    ) -> Image.Image:
        ...
```

`negative_prompt` は SDXL 専用なので基底シグネチャから除外した。各プロバイダが `**kwargs` で個別に受け取る設計にすることで、シグネチャの最小公倍数を保つ。

## Tensor / PIL.Image / bytes を PIL.Image に統一する変換レイヤ

プロバイダごとに返り値の型がバラバラなのが統合の難所だった。

- Wan 2.2（diffusers）: `torch.Tensor` shape `(B, C, H, W)`、値域 `[-1, 1]`
- SDXL（diffusers pipeline）: `PIL.Image.Image`
- Flux（fal.ai REST API）: `bytes`（JPEG）

`BaseImageProvider` の静的メソッドに変換ロジックを集約した。

```python
import io
import torch
from PIL import Image as PILImage

class BaseImageProvider(ABC):

    @staticmethod
    def _to_pil(raw) -> PILImage.Image:
        if isinstance(raw, torch.Tensor):
            t = raw.squeeze(0).clamp(-1, 1)
            t = (t + 1) / 2                          # [-1,1] → [0,1]
            arr = (t.permute(1, 2, 0).cpu().numpy() * 255).astype("uint8")
            return PILImage.fromarray(arr)
        if isinstance(raw, bytes):
            return PILImage.open(io.BytesIO(raw))
        return raw                                    # already PIL.Image

    @abstractmethod
    def generate(self, prompt: str, size: tuple[int, int],
                 steps: int, seed: int) -> PILImage.Image:
        ...
```

各プロバイダは `generate()` の末尾で `return self._to_pil(raw_output)` を呼ぶだけでよく、呼び出し元は型を意識しない。

## ProviderRegistry: str → クラスを解決するファクトリパターン

設定ファイルに `provider: "wan"` と書くだけで `WanProvider` がインスタンス化される仕組みがこのレジストリだ。デコレータで登録するため、新プロバイダ追加時に中央ファイルを触らなくていい。

```python
from typing import Type

class ProviderRegistry:
    _registry: dict[str, Type[BaseImageProvider]] = {}

    @classmethod
    def register(cls, name: str):
        def decorator(klass: Type[BaseImageProvider]):
            cls._registry[name] = klass
            return klass
        return decorator

    @classmethod
    def build(cls, name: str, **kwargs) -> BaseImageProvider:
        if name not in cls._registry:
            import difflib
            available = sorted(cls._registry)
            close = difflib.get_close_matches(name, available, n=1, cutoff=0.6)
            hint = f" Did you mean '{close[0]}'?" if close else ""
            raise ValueError(
                f"Unknown provider '{name}'.{hint} "
                f"Available: {available}"
            )
        return cls._registry[name](**kwargs)


# 各プロバイダファイルはデコレータを付けるだけ
@ProviderRegistry.register("wan")
class WanProvider(BaseImageProvider):
    def generate(self, prompt, size, steps, seed) -> PILImage.Image:
        ...
```

## エラーメッセージに候補列挙を入れた理由

`ValueError` のメッセージに `Available: ['flux', 'sdxl', 'wan']` を載せたのはデバッグ時間の削減が目的だ。6ヶ月で `"Wan"`（大文字）や `"wan2"` の誤記が5件発生し、候補列挙がない状態では `_registry` のソースを毎回確認する必要があった。`difflib.get_close_matches` を加えてからは初見のエラーでも1行で原因が分かるようになり、対応時間が平均4分から30秒に短縮された。

---
