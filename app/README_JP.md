# StackChan App

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Flutter](https://img.shields.io/badge/Flutter-3.0+-blue.svg)](https://flutter.dev/)
[![Dart](https://img.shields.io/badge/Dart-3.0+-blue.svg)](https://dart.dev/)

StackChan AIロボットコンパニオンを操作・インタラクションするための強力なFlutterアプリケーションです。Bluetooth接続、AI会話機能、顔表情レンダリング、ダンスコレオグラフィーなどの機能を備えています。

## 機能

- 🤖 **Bluetoothデバイス管理** - BLE経由でStackChanロボットに接続・制御
- 💬 **AI会話** - XiaoZhi AIを活用した自然言語インタラクション
- 🎭 **顔表情レンダリング** - Three.jsによるリアルタイム3Dフェイスアニメーション
- 🎵 **音楽とダンス** - 音楽に合わせたダンスコレオグラフィーの作成・再生
- 📷 **カメラ統合** - ARフィーチャーと顔検出
- 🔐 **セキュア通信** - データ送信のRSA暗号化

## システム要件

- **Flutter SDK**: 3.0+
- **Dart SDK**: 3.0+
- **iOS**: 14.0+（iOSデプロイ用）
- **Android**: API 21+（Androidデプロイ用）
- **macOS**: 11.0+（macOSデプロイ用）

## インストール

### 1. Flutterのインストール

使用するOSに応じて、公式のFlutterインストールガイドに従ってください。

#### macOS

```bash
# Flutter SDKをダウンロード
git clone https://github.com/flutter/flutter.git -b stable
export PATH="$PATH:`pwd`/flutter/bin"

# インストールの確認
flutter doctor
```

#### Windows

```bash
# Flutter SDKをhttps://flutter.dev/docs/get-started/install/windowsからダウンロード
# 解凍してPATHに追加

# インストールの確認
flutter doctor
```

#### Linux

```bash
# Flutter SDKをダウンロード
git clone https://github.com/flutter/flutter.git -b stable
export PATH="$PATH:`pwd`/flutter/bin"

# 依存パッケージをインストール
sudo apt-get install clang cmake ninja-build pkg-config libgtk-3-dev liblzma-dev

# インストールの確認
flutter doctor
```

### 2. プロジェクトのセットアップ

```bash
# リポジトリをクローン
git clone <repository-url>
cd StackChan

# 依存パッケージをインストール
flutter pub get
```

### 3. バックエンドサーバーの設定

アプリケーションの全機能を利用するにはバックエンドサーバーが必要です。ビルド前にサーバーエンドポイントを設定してください。

#### オプション A: 環境設定の使用（推奨）

プロジェクトルートに `.env` ファイルを作成するか、コード内の設定を直接変更してください。

#### オプション B: コード直接設定

`lib/network/urls.dart` を変更してバックエンドサーバーのURLを設定します。

```dart
// lib/network/urls.dart
class Urls {
  // Update this to your backend server address
  static const String url = "your-backend-server:port/";

// ... rest of the configuration
}
```

#### オプション C: 定数値の設定

`lib/util/value_constant.dart` を更新して暗号化キーやその他の定数を設定します。

```dart
// lib/util/value_constant.dart
class ValueConstant {
  // Server RSA Public Key for encryption
  static const String serverPublicKey = """
-----BEGIN PUBLIC KEY-----
YOUR_SERVER_PUBLIC_KEY_HERE
-----END PUBLIC KEY-----
""";

  // Client RSA Private Key for decryption
  static const String clientPrivateKey = """
-----BEGIN RSA PRIVATE KEY-----
YOUR_CLIENT_PRIVATE_KEY_HERE
-----END RSA PRIVATE KEY-----
""";
}
```

**重要**: 本番環境へのデプロイでは、キーをハードコーディングせず、環境変数またはセキュアなキー管理システムを使用してください。

## アプリケーションのビルド

### iOS

```bash
# CocoaPods依存パッケージをインストール
cd ios
pod install
cd ..

# iOSシミュレーターで実行
flutter run -d ios

# リリースビルド（iOSデバイス向け）
flutter build ios --release
```

### Android

```bash
# Androidエミュレーターまたは接続デバイスで実行
flutter run -d android

# リリース用APKをビルド
flutter build apk --release

# Google Play用App Bundleをビルド
flutter build appbundle --release
```

### Androidリリース署名（JKS）

リリースビルド（`apk --release` / `appbundle --release`）では、`build.gradle.kts` にパスワードをハードコーディングする代わりにキーストアを設定してください。

#### 1. JKSファイルを生成する

```bash
keytool -genkeypair -v \
  -keystore android/app/release.jks \
  -alias release \
  -keyalg RSA -keysize 2048 -validity 10000
```

#### 2. `android/key.properties` を作成する

```properties
storePassword=YOUR_STORE_PASSWORD
keyPassword=YOUR_KEY_PASSWORD
keyAlias=release
storeFile=../app/release.jks
```

> `android/.gitignore` はすでに `key.properties` と `*.jks` を除外しています。これらのファイルは非公開のまま保管してください。

#### 3. `android/app/build.gradle.kts` でプロパティを読み込む

Kotlin DSLスタイルの設定を使用します。

```kotlin
import java.util.Properties

val keystoreProperties = Properties()
val keystorePropertiesFile = rootProject.file("key.properties")
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(keystorePropertiesFile.inputStream())
}

android {
    signingConfigs {
        create("release") {
            if (keystorePropertiesFile.exists()) {
                storeFile = file(keystoreProperties["storeFile"] as String)
                storePassword = keystoreProperties["storePassword"] as String
                keyAlias = keystoreProperties["keyAlias"] as String
                keyPassword = keystoreProperties["keyPassword"] as String
            }
        }
    }

    buildTypes {
        release {
            signingConfig = signingConfigs.getByName("release")
        }
    }
}
```

#### 4. リリース成果物をビルドする

```bash
flutter build apk --release
flutter build appbundle --release
```

#### 5. CI/CDに関する推奨事項

CIでは、署名情報を環境変数またはセキュアなシークレットストレージ経由で注入してください。キーストアのパスワードや秘密鍵はリポジトリにコミットしないでください。

## プロジェクト構成

```
lib/
├── main.dart                    # Application entry point
├── app_state.dart               # Global state management
├── model/                       # Data models
│   ├── XiaoZhi/                 # AI service models
│   ├── blue_device_info.dart    # Bluetooth device models
│   ├── dance_list.dart          # Dance choreography models
│   └── ...
├── network/                     # Network layer
│   ├── http.dart                # HTTP client with interceptors
│   ├── urls.dart                # API endpoint configurations
│   └── web_socket_util.dart     # WebSocket management
├── util/                        # Utilities
│   ├── value_constant.dart      # App constants and keys
│   ├── rsa_util.dart            # RSA encryption/decryption
│   ├── blue_util.dart           # Bluetooth utilities
│   ├── music_util.dart          # Music and audio processing
│   └── ...
└── view/                        # UI layer
    ├── home/                    # Home screens
    ├── popup/                   # Modal screens
    └── util/                    # UI components and widgets
```

## バックエンドAPI統合

アプリケーションは2つのメインバックエンドサービスと連携しています。

### 1. StackChanバックエンド（`lib/network/urls.dart`）

- デバイスの登録と管理
- ダンスコレオグラフィーの保存
- ユーザー認証
- ファイルアップロードとメディア管理

### 2. XiaoZhi AIサービス（`lib/util/XiaoZhi_util.dart`）

- AI会話・チャット機能
- エージェントの管理と設定
- TTS（テキスト読み上げ）の音声選択
- ライセンスとアクティベーション管理

**ベースURL設定:**

- StackChanバックエンド: `http://<server-ip>:<port>/stackChan/`
- XiaoZhi AI: `https://XiaoZhi.me/`

## 開発

### コードスタイル

このプロジェクトは公式のDartおよびFlutterスタイルガイドラインに従っています。

- 変数と関数には `camelCase` を使用
- クラスと型には `PascalCase` を使用
- JSONキー（API通信）には `snake_case` を使用
- 公開APIにはdocコメントでドキュメントを記載

### テストの実行

```bash
# すべてのテストを実行
flutter test

# 特定のテストファイルを実行
flutter test test/widget_test.dart

# カバレッジ付きで実行
flutter test --coverage
```

### Lint

```bash
# 静的解析を実行
flutter analyze
```

## コントリビューション

コントリビューションを歓迎します！詳細は [コントリビューティングガイド](CONTRIBUTING.md) をご覧ください。

1. リポジトリをフォーク
2. フィーチャーブランチを作成（`git checkout -b feature/amazing-feature`）
3. 変更をコミット（`git commit -m 'Add some amazing feature'`）
4. ブランチにプッシュ（`git push origin feature/amazing-feature`）
5. プルリクエストを作成

## トラブルシューティング

### よくある問題

**1. flutter doctorが依存関係の不足を報告する**

- `flutter doctor` の指示に従って不足しているコンポーネントをインストールしてください
- iOS向け: Xcodeとコマンドラインツールがインストール・選択されていることを確認してください
- Android向け: Android StudioとSDKが正しく設定されていることを確認してください

**2. Bluetoothが動作しない**

- Bluetooth権限が付与されていることを確認してください
- iOS向け: Info.plistの `NSBluetoothAlwaysUsageDescription` を確認してください
- Android向け: `BLUETOOTH_SCAN` と `BLUETOOTH_CONNECT` のパーミッションを確認してください

**3. バックエンド接続が失敗する**

- `lib/network/urls.dart` のサーバーURLが正しいか確認してください
- ネットワーク接続を確認してください
- バックエンドサーバーが起動・アクセス可能であることを確認してください
- HTTPS接続のSSL証明書を確認してください

**4. iOSでビルドが失敗する**

```bash
cd ios
rm -rf Pods Podfile.lock
pod install --repo-update
cd ..
flutter clean
flutter pub get
```

## ライセンス

このプロジェクトはMITライセンスのもとで公開されています。詳細は [LICENSE](LICENSE) ファイルをご覧ください。

## 謝辞

- StackChanハードウェアを提供してくださったM5Stack Technology CO LTD
- 素晴らしいフレームワークを提供してくださったFlutterチーム
- 3Dレンダリング機能を提供するThree.js
- このプロジェクトで使用されているすべてのコントリビューターおよびオープンソースライブラリ

## サポート

サポートが必要な場合は以下をご利用ください。

1. [Issues](../../issues) ページで既知の問題を確認する
2. 問題がまだ掲載されていない場合は新しいIssueを作成する
3. セキュリティ上の問題については、security@m5stack.com に直接お問い合わせください

---

**注意**: このアプリケーションの全機能を利用するには、対応したStackChanハードウェアとバックエンドサービスが必要です。
