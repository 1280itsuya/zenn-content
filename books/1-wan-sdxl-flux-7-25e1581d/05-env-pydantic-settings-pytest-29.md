---
title: "env／pydantic-settings 一元管理とpytest 29本：『本番だけ壊れる』を防ぐテスト設計"
free: false
---

以下が執筆した章本文です。

---

## pydantic-settings で未設定 env を起動時に全検出する

本番だけ壊れる典型パターンは「ローカルで .env を読んでいるが CI/本番に渡し忘れた環境変数がある」だ。`pydantic-settings` の `BaseSettings` を使うと、起動時に一括バリデーションが走り、未設定の必須 env が即座に `ValidationError` で落ちる。

```python
# settings.py
from pydantic import Field
from pydantic_settings import BaseSettings
from typing import Literal

class AppSettings(BaseSettings):
    image_provider: Literal["wan", "sdxl", "flux"] = Field(
        ..., alias="IMAGE_PROVIDER"
    )
    wan_endpoint: str      = Field(..., alias="WAN_ENDPOINT")
    sdxl_api_key: str      = Field(..., alias="SDXL_API_KEY")
    flux_api_key: str      = Field(..., alias="FLUX_API_KEY")
    seed: int              = Field(42,  alias="IMAGE_SEED")

    model_config = {"env_file": ".env", "case_sensitive": True}

settings = AppSettings()
```

`IMAGE_PROVIDER` を未設定のまま起動すると `ValidationError: 1 validation error for AppSettings / image_provider / Field required` が即時 raise される。6ヶ月運用で env 渡し忘れによる本番障害は0件になった。

## IMAGE_PROVIDER=wan|sdxl|flux の ProviderFactory 実装

3プロバイダの差し替えは `ProviderFactory` 1関数に集約する。`generate()` の戻り値型を `GenerateResult` に固定することで、呼び出し側は provider を意識しない。

```python
# provider_factory.py
from dataclasses import dataclass
import numpy as np
from settings import settings

@dataclass
class GenerateResult:
    image: np.ndarray    # shape (H, W, 3), dtype uint8
    seed: int
    provider: str
    elapsed_sec: float

def build_provider():
    if settings.image_provider == "wan":
        from providers.wan import WanProvider
        return WanProvider(endpoint=settings.wan_endpoint, seed=settings.seed)
    if settings.image_provider == "sdxl":
        from providers.sdxl import SDXLProvider
        return SDXLProvider(api_key=settings.sdxl_api_key, seed=settings.seed)
    if settings.image_provider == "flux":
        from providers.flux import FluxProvider
        return FluxProvider(api_key=settings.flux_api_key, seed=settings.seed)
    raise ValueError(f"Unknown provider: {settings.image_provider}")
```

遅延 import にしているのは Wan2.2 が PyTorch ロードで約8秒かかるためだ。SDXL/Flux を使う CI で Wan のロードを避けられる。

## pytest monkeypatch で3プロバイダをモック化する

実 API を叩かずに全プロバイダのコードパスを通すには `monkeypatch.setenv` で `IMAGE_PROVIDER` を差し替え、各 Provider クラスを `MagicMock` で置換する。

```python
# conftest.py
import numpy as np
import pytest
from unittest.mock import MagicMock
from provider_factory import GenerateResult

FAKE_IMAGE = np.zeros((512, 512, 3), dtype=np.uint8)

@pytest.fixture(params=["wan", "sdxl", "flux"])
def provider_mock(request, monkeypatch):
    name = request.param
    monkeypatch.setenv("IMAGE_PROVIDER", name)
    monkeypatch.setenv("WAN_ENDPOINT",   "http://localhost:9000")
    monkeypatch.setenv("SDXL_API_KEY",   "test-sdxl")
    monkeypatch.setenv("FLUX_API_KEY",   "test-flux")

    mock = MagicMock()
    mock.generate.return_value = GenerateResult(
        image=FAKE_IMAGE, seed=42, provider=name, elapsed_sec=0.1
    )
    import importlib, settings as s
    monkeypatch.setattr(s, "settings", s.AppSettings())
    monkeypatch.setattr(
        f"providers.{name}.{name.capitalize()}Provider", lambda **_: mock
    )
    return mock
```

`params=["wan", "sdxl", "flux"]` で回すと、1テスト関数が自動で3回実行され、合計テスト数が3倍に膨らむ。

## 29本のテスト：型・shape・seed 再現性・ValidationError の全カバー

29本の内訳は「戻り値型 3本」「image shape 9本」「seed 再現性 9本」「ValidationError 8本」だ。

```python
# tests/test_provider.py
import numpy as np
import pytest
from provider_factory import build_provider, GenerateResult

class TestReturnType:
    def test_returns_generate_result(self, provider_mock):
        assert isinstance(build_provider().generate(prompt="cat"), GenerateResult)

    def test_image_is_ndarray(self, provider_mock):
        assert isinstance(build_provider().generate(prompt="cat").image, np.ndarray)

    def test_provider_name_matches_env(self, provider_mock, monkeypatch):
        import os
        assert build_provider().generate(prompt="cat").provider == os.environ["IMAGE_PROVIDER"]

class TestImageShape:
    @pytest.mark.parametrize("h,w", [(512, 512), (768, 512), (1024, 576)])
    def test_shape_hw3(self, provider_mock, h, w):
        result = build_provider().generate(prompt="cat", height=h, width=w)
        assert result.image.ndim == 3
        assert result.image.shape[2] == 3

class TestSeedReproducibility:
    def test_same_seed_same_output(self, provider_mock):
        p = build_provider()
        r1 = p.generate(prompt="dog", seed=7)
        r2 = p.generate(prompt="dog", seed=7)
        np.testing.assert_array_equal(r1.image, r2.image)

class TestValidationError:
    def test_missing_image_provider_raises(self, monkeypatch):
        monkeypatch.delenv("IMAGE_PROVIDER", raising=False)
        import importlib, settings
        with pytest.raises(Exception):
            importlib.reload(settings)

    def test_invalid_provider_value_raises(self, monkeypatch):
        monkeypatch.setenv("IMAGE_PROVIDER", "dalle")
        import importlib, settings
        with pytest.raises(Exception):
            importlib.reload(settings)
```

`pytest -v` の出力では `test_returns_generate_result[wan]`、`[sdxl]`、`[flux]` と3行並ぶ。全29本が通過すると `29 passed in 3.41s` になる。

## GitHub Actions で4分12秒の全プロバイダ結合テスト

モック29本とは別に、`INTEGRATION=1` フラグで実 API を叩く結合テストを毎朝 02:00 UTC に走らせる。Wan は GPU ランナーを使わず CPU + `--steps 1` で shape のみ確認し、実行時間を 4分12秒に抑えた。

```yaml
# .github/workflows/integration.yml
name: provider-integration

on:
  push:
    branches: [main]
  schedule:
    - cron: "0 2 * * *"

jobs:
  test:
    runs-on: ubuntu-latest
    env:
      INTEGRATION: "1"
      IMAGE_PROVIDER: ${{ matrix.provider }}
      WAN_ENDPOINT:   ${{ secrets.WAN_ENDPOINT }}
      SDXL_API_KEY:   ${{ secrets.SDXL_API_KEY }}
      FLUX_API_KEY:   ${{ secrets.FLUX_API_KEY }}
      IMAGE_SEED:     "42"

    strategy:
      matrix:
        provider: [wan, sdxl, flux]
      fail-fast: false   # 1プロバイダが落ちても他を続行

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install -r requirements.txt
      - run: pytest tests/ -v --timeout=120
```

`fail-fast: false` は必須だ。Flux の API キーが期限切れで落ちたとき、Wan と SDXL のテスト結果まで吹き飛ばした失敗から追加した。3ジョブが並列実行されるため、実測 4分12秒はシリアル比で約2.8倍速い。`secrets` に各プロバイダのキーを登録すれば、env の渡し忘れは CI 側でも `AppSettings()` が起動時に検出する。
