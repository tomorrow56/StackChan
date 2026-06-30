# StackChan オープンソース

<img src="https://m5stack-doc.oss-cn-shenzhen.aliyuncs.com/1205/K151_stack_chan_main_pictures_01.webp" width="60%">

ここでは、StackChan 関連のオープンソースリソースを公開しています。StackChan のファームウェア、リモートコントローラーのファームウェア、モバイルアプリ（iOS / Android）、サーバーのソースコードを含みます。

このリポジトリの更新は、リリース済みのファームウェアやモバイルアプリより遅れる場合があります。

----

<img src="https://cdn.shopify.com/s/files/1/0056/7689/2250/files/5a589623895f65487717894d9240f6b8.png" width="60%">

**StackChan は、M5Stack とユーザーコミュニティが共創した、とってもかわいい AI デスクトップロボットです。** メインコントローラーには M5Stack の**フラッグシップ IoT 開発キット [CoreS3](https://docs.m5stack.com/en/core/CoreS3)** を採用。ESP32-S3 SoC を搭載し、240 MHz デュアルコアプロセッサ、16 MB Flash、8 MB PSRAM、Wi-Fi / BLE をサポートします。本体には 2.0 インチ静電容量式タッチディスプレイ（高強度ガラスカバー）、0.3 MP カメラ、近接・環境光センサー、9 軸 IMU（加速度・ジャイロ・地磁気）、microSD カードスロット、1 W スピーカー、デュアルマイク、電源/リセットボタンを搭載しています。

メインコントローラーに接続された**ロボットボディ**には、給電とデータ通信用の USB-C ポート、550 mAh バッテリー、2 基のフィードバックサーボ（水平軸は 360 度連続回転、垂直軸は 90 度可動）、計 12 個の RGB LED（2 列）、赤外線送信機/受信機、3 ゾーンタッチパネル、多機能 NFC モジュールを装備しています。

**工場出荷時ファームウェア**は、AI Agent、豊かな表情のアニメーション、ESP-NOW ワイヤレスリモートコントロール、オンラインアプリダウンロードなど、多くの機能を備えています。モバイルアプリと連携して映像の確認やアバターのリモート操作ができ、OTA によるオンライン更新にも対応します。また、Arduino や UiFlow2 などでのプログラミングも可能で、M5Stack エコシステムの各種拡張ユニットと接続できるため、さまざまなカスタム機能を簡単に実現できます。

> ⚠️ モーターに接続された可動部が、電源投入・制御下にあるか不明な場合は、無理に手で回さないでください。ハードウェアの破損の原因となる可能性があります。

- 購入リンク: [M5Stack Official Store](https://shop.m5stack.com/products/stackchan-kawaii-co-created-open-source-ai-desktop-robot) | [淘宝 Taobao](https://item.taobao.com/item.htm?id=1042238294510)

- 製品ドキュメントページ: [English](https://docs.m5stack.com/en/StackChan) | [日本語](https://docs.m5stack.com/ja/StackChan) | [中文](https://docs.m5stack.com/zh_CN/StackChan)

- ボードサポートパッケージ: https://github.com/m5stack/StackChan-BSP

StackChan コミュニティの貢献者、特に以下の方々に感謝いたします。

| ![](https://m5stack-doc.oss-cn-shenzhen.aliyuncs.com/1205/avatar_stack_chan.jpg) | ![](https://m5stack-doc.oss-cn-shenzhen.aliyuncs.com/1205/avatar_takao.jpg) |
| -------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| [@stack_chan](https://x.com/stack_chan)                                          | [@mongonta555](https://x.com/mongonta555)                                   |
| Shinya Ishikawa                                                                  | Takao Akaki                                                                 |
