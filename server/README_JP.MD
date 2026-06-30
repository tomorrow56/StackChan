# StackChan Server

StackChan Server は、StackChan エコシステムのバックエンドサービスです。StackChan デバイス、モバイルアプリ、ウェブベースの管理コンソール、および選定されたサードパーティ連携を単一の Go サービスから提供します。

このリポジトリには以下が含まれています:

- Go バックエンドのソースコード
- MySQL スキーマ初期化 SQL
- ビルド済みの Flutter Web 管理画面アセット
- Docker および Kustomize デプロイテンプレート
- StackChan デバイスとアプリクライアント間の WebSocket 転送
- XiaoZhi 連携ロジック

セキュリティに関する注記: この README では、実際のホスト名、本番アドレス、鍵、またはトークンの代わりに、`<your-server-host>`、`<admin-token>`、`<your-secret>` などのプレースホルダーを意図的に使用しています。

## 目次

- [1. 概要](#1-概要)
- [2. アーキテクチャとモジュール](#2-アーキテクチャとモジュール)
- [3. 要件](#3-要件)
- [4. ゼロからのクイックスタート](#4-ゼロからのクイックスタート)
- [5. 設定ガイド](#5-設定ガイド)
- [6. 管理コンソールガイド](#6-管理コンソールガイド)
- [7. アプリワークフロー](#7-アプリワークフロー)
- [8. デバイスワークフロー](#8-デバイスワークフロー)
- [9. 認証とレスポンス形式](#9-認証とレスポンス形式)
- [10. モジュール別 API リファレンス](#10-モジュール別-api-リファレンス)
- [11. WebSocket プロトコル](#11-websocket-プロトコル)
- [12. データベース概要](#12-データベース概要)
- [13. プロジェクト構成](#13-プロジェクト構成)
- [14. 開発・ビルド・デプロイ](#14-開発ビルドデプロイ)
- [15. 運用上の注意事項](#15-運用上の注意事項)
- [16. ライセンス](#16-ライセンス)

## 1. 概要

StackChan Server は主に4つの役割を担っています:

1. ユーザー認証
   - アプリは登録とログインをリモートの M5Stack 互換ユーザーシステムに委任します。
   - ログイン成功後、本サービスはユーザープロフィールをローカルに保存し、独自の JWT を発行します。

2. デバイスとコンテンツの管理
   - デバイスの自動作成、ユーザーへのバインド、アンバインド、名前変更、クエリが可能です。
   - ダンス、パノラマ画像、投稿、コメント、フレンド関係は MySQL に保存されます。

3. 管理コンソール機能
   - 管理者ログイン
   - App Store アイテム管理
   - アイコン、ファームウェア、メディア、その他アセットのファイルアップロード

4. リアルタイム通信
   - アプリクライアントと StackChan デバイスは `/stackChan/ws` を通じて接続します。
   - サーバーは音声、画像、モーション制御、通話状態、ステータスメッセージを転送します。

## 2. アーキテクチャとモジュール

### 2.1 アクター

システムには4種類の主要なクライアントタイプがあります:

- 管理者ユーザー: ウェブコンソールにログインし、App Store コンテンツを管理する
- アプリユーザー: 登録、ログイン、デバイスのバインド、ダンスの管理、デバイスデータの閲覧を行う
- StackChan デバイス: データのアップロード、コンテンツの取得、WebSocket セッションの確立を行う
- XiaoZhi プラットフォーム: トークン取得、デバイスアンバインドフロー、エージェントリセット操作に使用される

### 2.2 ルートグループ

| ルートプレフィックス | クライアント | 目的 |
| --- | --- | --- |
| `/admin/stackChan` | 管理コンソール | 管理者ログイン、アップロード、App Store 管理 |
| `/stackChan/v2` | モバイルアプリ | ユーザーログイン、登録、デバイスバインド、ダンス管理 |
| `/stackChan` | デバイス / レガシークライアント | デバイス API、投稿、パノラマ、アップロード、公開アプリ一覧、XiaoZhi API |
| `/stackChan/ws` | アプリ / デバイス | WebSocket リアルタイムチャネル |
| `/file/*` | 全クライアント | アップロードファイルおよびビルトインメディアの静的ファイルアクセス |
| `/` | ブラウザ | Flutter Web 管理コンソール |

### 2.3 コアモジュール

- User モジュール: リモートログイン/登録およびローカル JWT 発行
- Device モジュール: デバイスのブートストラップ、バインド/アンバインド、デバイス情報クエリ
- Dance モジュール: モーション JSON、ダンス名、音楽 URL の管理
- Community モジュール: デバイスからの投稿とコメント
- Pano モジュール: デバイスごとのパノラマ画像レコード
- App Store モジュール: 管理者側の CRUD、アプリ/デバイスクライアント向けの公開読み取りアクセス
- File モジュール: `/file` 配下でのファイルアップロードと配信
- WebSocket モジュール: 接続プール、メッセージ転送、ハートビート、古い接続のクリーンアップ
- XiaoZhi モジュール: トークン取得、強制リフレッシュ、デバイス側リセットのサポート

## 3. 要件

### 3.1 ランタイム依存関係

- Go: `go.mod` で宣言されている `go 1.26.3`
- MySQL: `8.0+` を推奨
- OS: 開発環境では macOS、Linux、または Windows、本番環境では Linux を推奨

### 3.2 オプションツール

- GoFrame CLI (`gf`): `make build`、`make dao`、`make ctrl` 用
- Docker: イメージパッケージング用
- `kubectl` および `kustomize`: Kubernetes デプロイ用

## 4. ゼロからのクイックスタート

### 4.1 リポジトリのクローン

```bash
git clone <your-repo-url>
cd stackchan-server
```

### 4.2 データベースの作成

SQL ファイルは `stackChan.` スキーマプレフィックスを使用しているため、先にデータベースを作成してください:

```sql
CREATE DATABASE IF NOT EXISTS stackChan
  DEFAULT CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;
```

次にスキーマをインポートします:

```bash
mysql -u <db_user> -p < check_list/create_mysql_database.sql
```

別のデータベース名を使用する場合は、以下のすべてを更新してください:

- `check_list/create_mysql_database.sql`
- `manifest/config/config.yaml`
- `hack/config.yaml`

### 4.3 サービスの設定

主要設定ファイル:

- `manifest/config/config.yaml`

最低限、以下のセクションを確認してください:

- `server.address`
- `database.default.link`
- `jwt.secret`
- `admin.users`
- `m5stack.*`
- `rsa.server.*`
- `xiaozhi.*`

プレースホルダーのみを含む最小構成例:

```yaml
server:
  address: ":12800"
  openapiPath: "/api.json"

logger:
  path: "./logs"
  file: "{Y-m-d}.log"
  level: "all"
  stdout: true

database:
  default:
    link: "mysql:<db_user>:<db_password>@tcp(<db_host>:3306)/stackChan?charset=utf8mb4&collation=utf8mb4_0900_ai_ci"

jwt:
  secret: "<long-random-jwt-secret>"

admin:
  users:
    - username: "<admin-username>"
      password: "<admin-password>"

m5stack:
  loginUrl: ""
  registrationUrl: ""
  registrationToken: "Bearer <registration-token>"
  issuer: ""
  audience: ""

xiaozhi:
  secret_key: "<xiaozhi-secret-key>"
  generate_license_token: "<xiaozhi-license-token>"
```

JWT シークレットの生成例:

```bash
openssl rand -base64 32
```

### 4.4 サービスの起動

開発モード:

```bash
go run .
```

または先にビルドする場合:

```bash
go build -o stackChan .
./stackChan
```

起動後のローカルエンドポイント例:

- `http://<your-server-host>:12800/` - 管理コンソール
- `http://<your-server-host>:12800/api.json` - OpenAPI JSON
- `http://<your-server-host>:12800/file/...` - 静的ファイル

### 4.5 初期動作確認

起動後の推奨チェック項目:

1. ブラウザで `http://<your-server-host>:12800/` を開く。
2. `http://<your-server-host>:12800/api.json` を開く。
3. 管理者ログイン API を呼び出し、トークンが返却されることを確認する。
4. ファイルを1つアップロードし、`/file/...` から取得できることを確認する。

## 5. 設定ガイド

### 5.1 `server`

- `address`: GoFrame のリッスンアドレス（例: `:12800`）
- `openapiPath`: OpenAPI JSON の出力パス（現在は `/api.json`）
- `swaggerPath`: デフォルトではコメントアウト。Swagger UI を公開する場合のみ有効化してください

注意: サービスは `internal/cmd/cmd.go` 内で `s.SetPort(12800)` も呼び出しています。ポート設定を変更する場合は、設定ファイルとコードパスの両方を確認してください。

### 5.2 `logger`

ログはデフォルトで `./logs` に書き込まれ、stdout にも出力されます。

便利な設定項目:

- `level`
- `stdout`
- `rotateExpire`
- `rotateBackup`
- `rotateSize`

### 5.3 `database`

本プロジェクトは現在 MySQL のみを対象としています。

テーブル定義を変更した場合、通常は GoFrame モデルの再生成も必要です:

```bash
make dao
```

実行前に、`hack/config.yaml` が正しいデータベースインスタンスを指していることを確認してください。

### 5.4 `jwt`

- ユーザー JWT と管理者 JWT の両方が `jwt.secret` を使用します
- ユーザートークンは長い有効期限で生成されます
- 管理者トークンは 24 時間の有効期限で生成されます

テスト以外の環境では強力なシークレットを使用してください。

### 5.5 `admin.users`

このセクションは管理コンソールの認証情報を定義します。

例:

```yaml
admin:
  users:
    - username: "<admin-username>"
      password: "<admin-password>"
```

### 5.6 `m5stack`

このセクションはアプリ側の登録とログインに使用されます。

- `loginUrl`: リモートログインエンドポイント
- `registrationUrl`: リモート登録エンドポイント
- `registrationToken`: リモート登録の Authorization 値
- `issuer` / `audience`: ローカル発行トークンに書き込まれる JWT クレーム値

### 5.7 `rsa`

デバイス API グループと WebSocket エントリはどちらも RSA 暗号化 MAC トークンに依存しています。

- サーバーは `rsa.server.private` で受信ヘッダーを復号化します
- アプリ/デバイス側は `rsa.server.public` で暗号化する必要があります

推奨される鍵生成コマンド:

```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out server_private.pem
openssl rsa -in server_private.pem -pubout -out server_public.pem
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out client_private.pem
openssl rsa -in client_private.pem -pubout -out client_public.pem
```

### 5.8 `xiaozhi`

XiaoZhi 連携に使用されます。

- `secret_key`: 開発者トークンと交換するためのキー
- `generate_license_token`: ライセンストークンエンドポイントから返却される値

サービスは XiaoZhi トークンをメモリにキャッシュし、リフレッシュロジックにより 24 時間周期でリセットします。

## 6. 管理コンソールガイド

ビルド済みの管理コンソールは `web/management` に格納されており、Go サーバーから直接配信されます。

エントリ URL:

```text
http://<your-server-host>:12800/
```

以下の例で使用するシェル変数を定義できます:

```bash
BASE_URL="http://<your-server-host>:12800"
```

### 6.1 管理者ログイン

```bash
curl -X POST "$BASE_URL/admin/stackChan/login" \
  -H 'Content-Type: application/json' \
  -d '{
    "user_name": "<admin-username>",
    "pass_word": "<admin-password>"
  }'
```

成功時のレスポンス形式:

```json
{
  "code": 0,
  "message": "",
  "data": {
    "token": "<admin-token>"
  }
}
```

重要な詳細:

- `/admin/stackChan/login` を除くすべての管理者 API は `Authorization` を必要とします
- 現在のミドルウェアは `Bearer <token>` ではなく、生のトークン値を期待しています

### 6.2 アプリアイコン、ファームウェア、またはアセットファイルのアップロード

```bash
curl -X POST "$BASE_URL/admin/stackChan/uploadFile" \
  -H "Authorization: <admin-token>" \
  -F "file=@./icon.png" \
  -F "name=icon.png" \
  -F "directory=apps"
```

典型的なレスポンス:

```json
{
  "code": 0,
  "message": "",
  "data": {
    "path": "file/apps/icon.png"
  }
}
```

公開 URL は以下の通りです:

```text
$BASE_URL/file/apps/icon.png
```

### 6.3 App Store アイテムの追加

```bash
curl -X POST "$BASE_URL/admin/stackChan/app/add" \
  -H "Authorization: <admin-token>" \
  -H 'Content-Type: application/json' \
  -d '{
    "appName": "StackChan Demo",
    "appIconUrl": "'$BASE_URL'/file/apps/icon.png",
    "description": "Demo app for StackChan",
    "firmwareUrl": "'$BASE_URL'/file/apps/demo.bin"
  }'
```

フィールドの概要:

- `appName`: 必須
- `appIconUrl`: 完全な公開 URL を推奨
- `description`: 自由記述の説明文
- `firmwareUrl`: ファームウェアまたはパッケージのダウンロード URL

### 6.4 アプリ一覧の取得

```bash
curl -H "Authorization: <admin-token>" \
  "$BASE_URL/admin/stackChan/apps"
```

### 6.5 アプリの更新

```bash
curl -X PUT "$BASE_URL/admin/stackChan/app/update" \
  -H "Authorization: <admin-token>" \
  -H 'Content-Type: application/json' \
  -d '{
    "id": 1,
    "appName": "StackChan Demo v2",
    "description": "Updated description"
  }'
```

### 6.6 アプリの削除

```bash
curl -X DELETE "$BASE_URL/admin/stackChan/app/delete" \
  -H "Authorization: <admin-token>" \
  -H 'Content-Type: application/json' \
  -d '{"id":1}'
```

削除は論理削除のみです: `app_store.is_deleted` が `1` に設定されます。

## 7. アプリワークフロー

アプリ側の API は `/stackChan/v2` 配下にグループ化されています。

### 7.1 ユーザー登録

```bash
curl -X POST "$BASE_URL/stackChan/v2/user/registration" \
  -H 'Content-Type: application/json' \
  -d '{
    "username": "<app-username>",
    "email": "<user-email>",
    "password": "<user-password>"
  }'
```

このリクエストは設定済みのリモート登録エンドポイントにプロキシされます。

### 7.2 ログイン

```bash
curl -X POST "$BASE_URL/stackChan/v2/user/login" \
  -H 'Content-Type: application/json' \
  -d '{
    "username": "<app-username>",
    "password": "<user-password>"
  }'
```

レスポンスにはローカル発行の JWT が含まれ、以下では `<user-token>` として参照します。

### 7.3 現在のユーザー情報の取得

```bash
curl "$BASE_URL/stackChan/v2/user" \
  -H "token: Bearer <user-token>"
```

### 7.4 デバイスのバインド

```bash
curl -X POST "$BASE_URL/stackChan/v2/device/bind" \
  -H "token: Bearer <user-token>" \
  -H 'Content-Type: application/json' \
  -d '{"mac":"AA:BB:CC:DD:EE:FF"}'
```

成功時、サーバーは以下を実行します:

- デバイスレコードが存在しない場合は作成する
- 現在のユーザー UID を `device.uid` に保存する
- `bind_time` を設定する

### 7.5 現在のユーザーのデバイス一覧

```bash
curl "$BASE_URL/stackChan/v2/devices" \
  -H "token: Bearer <user-token>"
```

### 7.6 デバイスの更新

```bash
curl -X PUT "$BASE_URL/stackChan/v2/device/update" \
  -H "token: Bearer <user-token>" \
  -H 'Content-Type: application/json' \
  -d '{
    "mac": "AA:BB:CC:DD:EE:FF",
    "name": "Desk StackChan"
  }'
```

注意: リクエスト構造体には `longitude` と `latitude` が含まれていますが、デフォルトの SQL スキーマでは現在 `device` テーブルにこれらのカラムは作成されません。使用する予定がある場合は、先にカラムを追加してください。

### 7.7 デバイスのアンバインド

```bash
curl -X POST "$BASE_URL/stackChan/v2/device/unbind" \
  -H "token: Bearer <user-token>" \
  -H 'Content-Type: application/json' \
  -d '{"mac":"AA:BB:CC:DD:EE:FF"}'
```

このフローは以下を試行します:

- デフォルトの XiaoZhi エージェント設定を復元する
- XiaoZhi 側でデバイスをアンバインドする
- ローカルのバインド関係をクリアする

### 7.8 デフォルトのエージェント設定を復元する

```bash
curl -X POST "$BASE_URL/stackChan/v2/device/agent/restore" \
  -H "token: Bearer <user-token>" \
  -H 'Content-Type: application/json' \
  -d '{"mac":"AA:BB:CC:DD:EE:FF"}'
```

### 7.9 ダンスの管理

特定デバイスのダンス一覧を取得:

```bash
curl -G "$BASE_URL/stackChan/v2/dance" \
  -H "token: Bearer <user-token>" \
  --data-urlencode "mac=AA:BB:CC:DD:EE:FF"
```

ダンスの作成:

```bash
curl -X POST "$BASE_URL/stackChan/v2/dance" \
  -H "token: Bearer <user-token>" \
  -H 'Content-Type: application/json' \
  -d '{
    "mac": "AA:BB:CC:DD:EE:FF",
    "danceName": "wave-demo",
    "musicUrl": "'$BASE_URL'/file/music/stackchan_music.mp3",
    "danceData": [
      {
        "leftEye": {"x":0,"y":0,"rotation":0,"weight":100,"size":0},
        "rightEye": {"x":0,"y":0,"rotation":0,"weight":100,"size":0},
        "mouth": {"x":0,"y":0,"rotation":0,"weight":0,"size":0},
        "pitchServo": {"angle":450,"speed":900},
        "yawServo": {"angle":0,"speed":900},
        "leftRgbColor": "#AAAAAA",
        "rightRgbColor": "#AAAAAA",
        "durationMs": 1000
      }
    ]
  }'
```

ダンスの詳細を取得:

```bash
curl -G "$BASE_URL/stackChan/v2/danceData" \
  -H "token: Bearer <user-token>" \
  --data-urlencode "id=1"
```

ダンスの更新:

```bash
curl -X PUT "$BASE_URL/stackChan/v2/dance" \
  -H "token: Bearer <user-token>" \
  -H 'Content-Type: application/json' \
  -d '{"id":1,"danceName":"wave-demo-v2"}'
```

ダンスの削除:

```bash
curl -X DELETE "$BASE_URL/stackChan/v2/dance" \
  -H "token: Bearer <user-token>" \
  -H 'Content-Type: application/json' \
  -d '{"id":1}'
```

## 8. デバイスワークフロー

デバイス側の API は主に `/stackChan` と `/stackChan/ws` を使用します。

### 8.1 デバイス認証ヘッダーの構築

デバイスの REST API と WebSocket エンドポイントの両方が RSA 暗号化 MAC トークンを期待しています。

プレーンテキスト形式:

```text
<mac>|<nonce>|<unix_timestamp_seconds>
```

生成ロジック:

```text
plainText = "AA:BB:CC:DD:EE:FF|random_nonce|1710000000"
cipher    = RSA-OAEP-SHA256-Encrypt(plainText, rsa.server.public)
header    = Base64(cipher)
```

リクエストヘッダー:

```http
Authorization: <rsa-mac-token>
```

サーバーは以下を実行します:

- ヘッダーを Base64 デコードする
- `rsa.server.private` を使用して復号化する
- ペイロードから MAC を抽出する
- タイムスタンプがサーバー時刻と 10 秒以上異なるトークンを拒否する

### 8.2 デバイスレコードの作成

```bash
curl -X POST "$BASE_URL/stackChan/device" \
  -H "Authorization: <rsa-mac-token>" \
  -H 'Content-Type: application/json' \
  -d '{"name":"My StackChan"}'
```

### 8.3 デバイス情報の取得

```bash
curl "$BASE_URL/stackChan/device/info" \
  -H "Authorization: <rsa-mac-token>"
```

### 8.4 デバイス情報の更新

```bash
curl -X PUT "$BASE_URL/stackChan/device/info" \
  -H "Authorization: <rsa-mac-token>" \
  -H 'Content-Type: application/json' \
  -d '{"name":"Living Room StackChan"}'
```

### 8.5 このデバイスにバインドされたユーザーの取得

```bash
curl "$BASE_URL/stackChan/device/user" \
  -H "Authorization: <rsa-mac-token>"
```

### 8.6 デバイス側からのアンバインド

```bash
curl -X POST "$BASE_URL/stackChan/device/unbind" \
  -H "Authorization: <rsa-mac-token>"
```

### 8.7 ランダムなオンラインデバイスの取得

```bash
curl -G "$BASE_URL/stackChan/device/randomList" \
  -H "Authorization: <rsa-mac-token>" \
  --data-urlencode "pageSize=6"
```

このリストは現在接続中の StackChan WebSocket クライアントから抽出され、呼び出し元自身の MAC は除外されます。

### 8.8 フレンド、投稿、コメント、パノラマ API

フレンドデバイスの追加:

```bash
curl -X POST "$BASE_URL/stackChan/friend" \
  -H "Authorization: <rsa-mac-token>" \
  -H 'Content-Type: application/json' \
  -d '{"friendMac":"11:22:33:44:55:66"}'
```

投稿の作成:

```bash
curl -X POST "$BASE_URL/stackChan/post/add" \
  -H "Authorization: <rsa-mac-token>" \
  -H 'Content-Type: application/json' \
  -d '{
    "content_text": "hello stackchan",
    "content_image": "'$BASE_URL'/file/posts/a.jpg"
  }'
```

投稿の一覧:

```bash
curl -G "$BASE_URL/stackChan/post/get" \
  -H "Authorization: <rsa-mac-token>" \
  --data-urlencode "page=1" \
  --data-urlencode "pageSize=10"
```

コメントの作成:

```bash
curl -X POST "$BASE_URL/stackChan/post/comment/create" \
  -H "Authorization: <rsa-mac-token>" \
  -H 'Content-Type: application/json' \
  -d '{"postId":1,"content":"nice!"}'
```

コメントの一覧:

```bash
curl -G "$BASE_URL/stackChan/post/comment/get" \
  -H "Authorization: <rsa-mac-token>" \
  --data-urlencode "postId=1" \
  --data-urlencode "page=1" \
  --data-urlencode "pageSize=10"
```

パノラマ画像レコードの追加:

```bash
curl -X POST "$BASE_URL/stackChan/pano" \
  -H "Authorization: <rsa-mac-token>" \
  -H 'Content-Type: application/json' \
  -d '{"url":"'$BASE_URL'/file/pano/p1.jpg"}'
```

パノラマ画像の一覧:

```bash
curl "$BASE_URL/stackChan/pano" \
  -H "Authorization: <rsa-mac-token>"
```

### 8.9 デバイス側からのファイルアップロード

```bash
curl -X POST "$BASE_URL/stackChan/uploadFile" \
  -H "Authorization: <rsa-mac-token>" \
  -F "file=@./photo.jpg" \
  -F "name=photo.jpg" \
  -F "directory=posts"
```

### 8.10 公開アプリ一覧と XiaoZhi API

公開アプリ一覧の取得:

```bash
curl "$BASE_URL/stackChan/apps"
```

現在の XiaoZhi 開発者トークンの取得:

```bash
curl "$BASE_URL/stackChan/xiaozhi/token"
```

トークンの強制リフレッシュ:

```bash
curl "$BASE_URL/stackChan/xiaozhi/token/refresh"
```

ライセンストークンの取得:

```bash
curl "$BASE_URL/stackChan/xiaozhi/generateLicenseToken"
```

## 9. 認証とレスポンス形式

### 9.1 認証ルール

| グループ | プレフィックス | ヘッダー | 備考 |
| --- | --- | --- | --- |
| Admin | `/admin/stackChan` | `Authorization: <admin-token>` | `/login` は公開 |
| App | `/stackChan/v2` | `token: Bearer <user-token>` | `/user/login` と `/user/registration` は公開 |
| Device | `/stackChan` | `Authorization: <rsa-mac-token>` | ほとんどの API は MAC 抽出に依存 |
| WebSocket | `/stackChan/ws` | `Authorization: <rsa-mac-token>` | クエリパラメータ `deviceType` も必要 |
| 静的ファイル | `/file/*` | なし | 直接ファイルアクセス |

追加の注意事項:

- アプリのミドルウェアは `Bearer ` プレフィックスを自動的に除去します
- 管理者のミドルウェアは生のトークン値を期待しています
- 一部の `/stackChan` ルートは MAC が存在しなくてもハードフェイルしませんが、クライアントは一貫して標準ヘッダー形式に従うべきです

### 9.2 統一レスポンス形式

GoFrame は通常の API レスポンスを以下の形式でラップします:

```json
{
  "code": 0,
  "message": "",
  "data": {}
}
```

失敗時:

- `code` にはエラーコードが含まれます
- `message` にはエラーの説明が含まれます
- `data` は通常 `null` です

## 10. モジュール別 API リファレンス

### 10.1 管理者 API: `/admin/stackChan`

| Method | Path | 目的 | 主要パラメータ |
| --- | --- | --- | --- |
| POST | `/login` | 管理者ログイン | `user_name`, `pass_word` |
| POST | `/uploadFile` | アイコン、ファームウェア、またはアセットファイルのアップロード | form-data: `file`, `name`, `directory` |
| POST | `/app/add` | App Store アイテムの作成 | `appName`, `appIconUrl`, `description`, `firmwareUrl` |
| GET | `/apps` | アプリ一覧 | なし |
| PUT | `/app/update` | アプリの更新 | `id` 必須、その他はオプション |
| DELETE | `/app/delete` | アプリの論理削除 | `id` |

### 10.2 アプリ API: `/stackChan/v2`

| Method | Path | 目的 | 主要パラメータ |
| --- | --- | --- | --- |
| POST | `/user/registration` | ユーザー登録 | `username`, `email`, `password` |
| POST | `/user/login` | ログインして JWT を取得 | `username`, `password` |
| GET | `/user` | 現在のユーザーを取得 | ヘッダー `token` |
| GET | `/devices` | 現在のユーザーのデバイス一覧 | ヘッダー `token` |
| POST | `/device/bind` | デバイスのバインド | `mac` |
| POST | `/device/unbind` | デバイスのアンバインド | `mac` |
| PUT | `/device/update` | バインド済みデバイスの更新 | `mac`、オプション `name`, `longitude`, `latitude` |
| POST | `/device/agent/restore` | デフォルトのエージェント設定を復元 | `mac` |
| GET | `/dance` | デバイスのダンス一覧 | クエリ `mac` |
| POST | `/dance` | デバイスのダンスを作成 | `mac`, `danceName`, `danceData`, `musicUrl` |
| GET | `/danceData` | ダンスの詳細を取得 | クエリ `id` |
| PUT | `/dance` | ダンスの更新 | `id` と オプションフィールド |
| DELETE | `/dance` | ダンスの削除 | `id` |

### 10.3 デバイス API: `/stackChan`

| Method | Path | 目的 | 主要パラメータ |
| --- | --- | --- | --- |
| POST | `/device` | デバイスレコードの作成 | `name` |
| PUT | `/device` | デバイス名の更新 | `name` |
| GET | `/device/info` | 現在のデバイス情報を取得 | なし |
| PUT | `/device/info` | 現在のデバイス情報を更新 | `name` |
| GET | `/device/randomList` | ランダムなオンラインデバイス一覧 | クエリ `pageSize` |
| GET | `/device/user` | 現在のデバイスのバインド済みユーザーを取得 | なし |
| POST | `/device/unbind` | デバイス側からのアンバインド | なし |
| POST | `/friend` | フレンドデバイスの追加 | `friendMac` |
| GET | `/dance` | 現在のデバイスのダンス一覧 | なし |
| POST | `/dance` | 現在のデバイスのダンスを作成 | `danceName`, `danceData`, `musicUrl` |
| GET | `/danceData` | ダンスの詳細を取得 | クエリ `id` |
| PUT | `/dance` | ダンスの更新 | `id` と オプションフィールド |
| DELETE | `/dance` | ダンスの削除 | `id` |
| GET | `/musicList` | ビルトイン音楽ファイル一覧 | なし |
| POST | `/pano` | パノラマレコードの追加 | `url` |
| GET | `/pano` | パノラマレコード一覧 | なし |
| POST | `/uploadFile` | ファイルのアップロード | form-data: `file`, `name`, `directory` |
| GET | `/apps` | 公開 App Store 一覧の取得 | なし |
| GET | `/xiaozhi/token` | XiaoZhi トークンの取得 | なし |
| GET | `/xiaozhi/token/refresh` | XiaoZhi トークンの強制リフレッシュ | なし |
| GET | `/xiaozhi/generateLicenseToken` | ライセンストークンの取得 | なし |

### 10.4 コミュニティ API: `/stackChan`

| Method | Path | 目的 | 主要パラメータ |
| --- | --- | --- | --- |
| POST | `/post/add` | 投稿の作成 | `content_text`、オプション `content_image` |
| GET | `/post/get` | ページネーション付き投稿一覧 | `page`, `pageSize` |
| DELETE | `/post/delete` | 投稿の削除 | `id` |
| POST | `/post/comment/create` | コメントの作成 | `postId`, `content` |
| GET | `/post/comment/get` | ページネーション付きコメント一覧 | `postId`, `page`, `pageSize` |
| DELETE | `/post/comment/delete` | コメントの削除 | `postId`, `commentId` |

### 10.5 公開 App Store レスポンス

公開リストエンドポイント:

```bash
curl "$BASE_URL/stackChan/apps"
```

返却されるフィールドは通常以下を含みます:

- `id`
- `appName`
- `appIconUrl`
- `description`
- `firmwareUrl`
- `createAt`
- `updateAt`

## 11. WebSocket プロトコル

エンドポイント URL の例:

```text
ws://<your-server-host>:12800/stackChan/ws?deviceType=StackChan
ws://<your-server-host>:12800/stackChan/ws?deviceType=App&deviceId=<app-device-id>
```

要件:

- `Authorization: <rsa-mac-token>` を送信する必要があります
- `deviceType=StackChan` はハードウェアクライアントを意味します
- `deviceType=App` はアプリクライアントを意味し、`deviceId` も必要です

### 11.1 バイナリフレーム形式

```text
1 byte  : msgType
4 bytes : payload length (big endian)
N bytes : payload
```

### 11.2 メッセージタイプ

| 値 | 名前 | 意味 |
| --- | --- | --- |
| `0x01` | `Opus` | 音声フレーム |
| `0x02` | `Jpeg` | 画像フレーム |
| `0x03` | `ControlAvatar` | アバター / 表情制御 |
| `0x04` | `ControlMotion` | モーション制御 |
| `0x05` | `OnCamera` | カメラをオンにする |
| `0x06` | `OffCamera` | カメラをオフにする |
| `0x07` | `TextMessage` | テキストメッセージ |
| `0x09` | `RequestCall` | 通話リクエスト |
| `0x0A` | `RefuseCall` | 通話を拒否 |
| `0x0B` | `AgreeCall` | 通話を承諾 |
| `0x0C` | `HangupCall` | 通話を切断 |
| `0x0D` | `UpdateDeviceName` | デバイス名の更新 |
| `0x0E` | `GetDeviceName` | デバイス名の照会 |
| `0x10` | `ping` | サーバーハートビート |
| `0x11` | `pong` | クライアントハートビート応答 |
| `0x12` | `OnPhoneScreen` | 電話画面投影の開始 |
| `0x13` | `OffPhoneScreen` | 電話画面投影の停止 |
| `0x14` | `Dance` | ダンス再生メッセージ |
| `0x15` | `GetAvatarPosture` | アバター姿勢の照会 |
| `0x16` | `DeviceOffline` | デバイスオフライン通知 |
| `0x17` | `DeviceOnline` | デバイスオンライン通知 |
| `0x18` | `OnAudio` | 音声サブスクリプションの開始 |
| `0x19` | `OffAudio` | 音声サブスクリプションの停止 |
| `0x1A` | `AimedTakePhoto` | 指定写真撮影 |

### 11.3 タイマー

起動時に、サーバーは2つの定期タスクを開始します:

- 5 秒ごと: WebSocket ハートビートメッセージの送信
- 15 秒ごと: 期限切れ接続のクリーンアップ

## 12. データベース概要

スキーマファイル:

- `check_list/create_mysql_database.sql`

主要テーブル:

| テーブル | 目的 |
| --- | --- |
| `user` | リモート UID をキーとするキャッシュ済みローカルユーザープロフィール |
| `device` | `mac` をキーとするデバイステーブル |
| `device_dance` | ダンスモーションデータ |
| `device_friend` | デバイスフレンドシップリンク |
| `device_pano` | パノラマ画像レコード |
| `device_post` | デバイス作成の投稿 |
| `device_post_comment` | 投稿コメント |
| `app_store` | 管理者が管理するアプリ一覧 |

追加の注意事項:

- `app_store` は `is_deleted` による論理削除を使用します
- `device.bind_time` は現在 datetime 型ではなく文字列として保存されています
- SQL スクリプトは `stackChan` データベース自体を作成しません。先に作成してください

## 13. プロジェクト構成

```text
.
├── api/                         # GoFrame API definitions
│   ├── admin/                   # Admin APIs
│   ├── appstore/                # Public App Store APIs
│   ├── dance/                   # Dance APIs
│   ├── device/                  # Device and app-device APIs
│   ├── file/                    # Upload APIs
│   ├── friend/                  # Friend APIs
│   ├── pano/                    # Pano APIs
│   ├── post/                    # Post and comment APIs
│   ├── stackchandevice/         # Device-bound-user and device-unbind APIs
│   ├── user/                    # User login and registration APIs
│   └── xiaozhi/                 # XiaoZhi APIs
├── check_list/                  # MySQL bootstrap SQL
├── file/                        # Uploaded files and built-in music
├── hack/                        # GoFrame CLI and generation config
├── internal/
│   ├── boot/                    # Heartbeat and cleanup schedulers
│   ├── cmd/                     # HTTP server startup and route binding
│   ├── controller/              # Controllers
│   ├── dao/                     # GoFrame DAO layer
│   ├── middleware/              # Auth and CORS middleware
│   ├── model/                   # Business models and entities
│   ├── packed/                  # Embedded/static resource linkage
│   ├── service/                 # Service layer
│   ├── web_socket/              # WebSocket pools and forwarding
│   └── xiaozhi/                 # XiaoZhi client logic
├── manifest/
│   ├── config/                  # Runtime config
│   ├── deploy/                  # Kustomize templates
│   └── docker/                  # Docker build files
├── resource/                    # Template/resource directory
├── utility/                     # Helpers such as RSA decryption
├── web/management/              # Built Flutter Web admin assets
├── LICENSE
├── Makefile
├── README.MD
└── main.go
```

## 14. 開発・ビルド・デプロイ

### 14.1 よく使うコマンド

```bash
# Run directly
go run .

# Build a binary
go build -o stackChan .

# Build via GoFrame CLI
make build

# Generate controller/sdk code
make ctrl

# Generate DAO/DO/Entity from the database
make dao

# Build Docker image
make image
```

注意事項:

- `make build`、`make dao`、`make ctrl` は `gf` CLI に依存しています
- `gf` がインストールされていない場合、Makefile は自動的にダウンロードとインストールを試みます

### 14.2 Docker

Dockerfile のパス:

- `manifest/docker/Dockerfile`

必要な入力:

- `stackChan` という名前のコンパイル済みバイナリ
- ビルドコンテキストに配置された `config.yaml` ファイル

典型的なフロー:

```bash
go build -o stackChan .
cp manifest/config/config.yaml ./config.yaml
docker build -f manifest/docker/Dockerfile -t stackchan-server:local .
```

### 14.3 Kubernetes

リポジトリには `manifest/deploy/kustomize` が含まれていますが、本番環境向けのデプロイではなくスターターテンプレートとして扱ってください。

使用前に、少なくとも以下を確認してください:

- サービス名がまだ `template-single` ベースであること
- ターゲットポートがまだ `8000` ベースであること
- プレースホルダーのデプロイメタデータ
- 実際のサービスポートとイメージ名に対する環境固有の設定の整合性

## 15. 運用上の注意事項

1. 管理コンソールは静的アセットとして提供されています
   - リポジトリには `web/management` 配下にビルド済み出力が含まれています
   - そのフロントエンドの Flutter ソースプロジェクトは含まれていません

2. v2 ダンスのデフォルトパスにハードコードされた外部音楽 URL があります
   - `internal/controller/dance/dance_v2_get_list.go` を確認してください
   - 本番使用前に、ご自身の公開ファイル URL またはローカルデプロイ URL に置き換えてください

3. デバイス更新 API は `longitude` と `latitude` を公開しています
   - デフォルトのスキーマではこれらのカラムは作成されません
   - この機能を正しく有効にするには、先にカラムを追加してください

4. 環境固有の設定値をそのまま公開しないでください
   - デプロイ前に管理者認証情報、JWT シークレット、登録トークン、RSA 鍵、XiaoZhi シークレットを置き換えてください
   - リポジトリ全体をオープンソース化する予定がある場合は、この README とは別にランタイム設定ファイルをサニタイズしてください

5. 管理者とアプリのトークンヘッダーは意図的に異なっています
   - Admin: `Authorization: <admin-token>`
   - App: `token: Bearer <user-token>`

## 16. ライセンス

本プロジェクトは MIT ライセンスの下でリリースされています。

ルートファイルを参照してください:

- `LICENSE`

```text
MIT License
Copyright (c) 2026 M5Stack Technology CO LTD
```
