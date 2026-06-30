# StackChan-RemoteControl-ESPNow

このリポジトリは、`ESPNow` プロトコルを使用して StackChan のサーボ動作をリモート制御するために設計されています。

## 対象デバイス

* StickC-Plus + Hat Mini JoyC

## コンパイル環境

* IDF：ESP-IDF v5.4.2
* デバイスタイプ：esp32

## コンパイルとフラッシュ

1. コンパイル前に、プロジェクト全体で `M5GFX` 内の `__has_include(<driver/i2c_master.h>)` をグローバル検索し、すべての出現箇所を `0` に置換してください。
2. フラッシュ時は、ボーレートを `1500000` に指定してください。

## パッケージファームウェア

```
esptool.py --chip esp32 merge_bin -o StackChan-RemoteControl-ESPNow-jyy-20251231_0x0.bin 0x1000 build\bootloader\bootloader.bin 0x8000 build\partition_table\partition-table.bin 0x10000 build\StackChan-RemoteControl-ESPNow.bin
```
