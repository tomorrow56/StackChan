
## ビルド

### 依存関係の取得

```bash
python3 ./fetch_repos.py
```

### ツールチェーン

[ESP-IDF v5.5.4](https://docs.espressif.com/projects/esp-idf/en/v5.5.4/esp32s3/index.html)

### ビルド

```bash
idf.py build
```

### ホスト側テスト

モーション座標ヘルパーは、ESP-IDFハードウェアなしでテストできます：

```bash
cmake -S tests -B build-host-tests
cmake --build build-host-tests
ctest --test-dir build-host-tests --output-on-failure
```

### フラッシュ書き込み

```bash
idf.py flash
```
