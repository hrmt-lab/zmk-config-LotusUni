# lotus_uni_ble — なぜビルド対象から外したか（2026-09-20 調査）

ドングルを使わない BLE 単体構成で Keylink Studio (Host Link v2) を動かそうとしたが、
**バッテリー駆動での Host Link 接続を安定させられなかった**ため、`build.yaml` から外した。
シールド定義 (`boards/shields/lotus_uni/lotus_uni_ble.conf` / `.overlay`、
`Kconfig.shield` / `Kconfig.defconfig` の該当ブロック) は残してある。

## 症状

- BLE 接続でキー入力はできる（ただし引っかかり、同じ文字の連続入力あり）
- Keylink Studio が Host Link デバイスとして認識しない
- ZMK Studio (BLE) は「編集可能」になることがある

## 切り分けで確定したこと

**ソフトウェア側はすべて正常。** 実機ログで確認済み。

- 列挙: Windows は HID サービスを 2 つとも列挙し、Raw HID 側に usage page `0xFF60` /
  usage `0x61` のノードを作る。hidapi からも見えている
- ATT MTU: `mtu=69`（64 byte の書き込みに必要な 67 以上）
- Long Write への転落なし (`offset=0`)
- Windows の CCC 購読あり、`bt_gatt_notify_cb()` は `returned 0`
- 届いたパケットは常に正常にパースされ、DEVICE_HELLO を返している

**USB 接続時は 7/7 で完全動作。バッテリー／充電器駆動ではほぼ成功しない。**

## 根本原因

BLE リンクの電波的な余裕が不足している。パケットサイズで通る／通らないが分かれる。

| 通信 | 1回あたりのサイズ | 結果 |
| --- | --- | --- |
| キー入力 (HID レポート) | 8 byte | 通る（取りこぼしあり） |
| ZMK Studio (BLE) | 30 byte 程度に小分け | たまに通る |
| **Host Link** | **64 byte 固定** (ATT PDU 67 byte) | **通らない** |
| split (ドングル構成) | キー位置のみの小パケット | 問題なし |

長いパケットほど空中時間が長く、1 ビットの誤りで全体が捨てられる。
さらに Windows は HID 出力レポートを **Write Without Response** (`flags=0x02`) で送るため、
落ちても ATT レベルの再送がなく無言で消える。

補強する観測:

- アルミケースを閉じると悪化し、開けると改善する（ただし開けても不足）
- USB ケーブル接続時に安定するのは、ケーブルが放射素子として働くためと推定
- PC 側 Bluetooth は MediaTek MT7925 の WiFi/BT 複合モジュール。
  2.4GHz を共有する coexistence の影響を受けている可能性がある

## 効果がなかった設定（再挑戦時に繰り返さないこと）

`BT_CTLR_RX_BUFFERS=3` / `BT_CTLR_DATA_LENGTH_MAX` 変更 / `BT_CTLR_XTAL_ADVANCED=n` /
`BT_GATT_ENFORCE_SUBSCRIPTION=n` / `BT_BUF_ACL_TX_SIZE`・`TX_COUNT` 変更 /
`KEY_PRESS`・`KEY_STATS` の無効化 / `BT_PERIPHERAL_PREF_LATENCY=0`

**悪化した設定**: `BT_CTLR_PHY_2M=n`
（2M PHY は空中時間が半分で干渉に強い。切ってはいけない）

## 再挑戦する場合に必ず必要な設定

```ini
# ZMK 既定は ATT MTU 65。64 byte の書き込みには 1+2+64 = 67 byte 必要で原理的に不可能。
# ATT MTU = MIN(BT_BUF_ACL_RX_SIZE - 4, BT_L2CAP_TX_MTU)
CONFIG_BT_BUF_ACL_RX_SIZE=73
CONFIG_BT_L2CAP_TX_MTU=69

# nRF52840 の送信出力上限。既定は 0 dBm
CONFIG_BT_CTLR_TX_PWR_PLUS_8=y
```

先に潰すべきは電波環境（ケースのアンテナ開口、PC 側 BT アダプタの位置、WiFi coexistence）。
firmware 設定で埋められる範囲は出し切っている。

## 調査中に見つかった zmk-raw-hid の実装上の不備（今回の原因ではない）

いずれも今回の症状の原因ではなかったが、切り分け中に判明した実装差。
**3件とも修正済み**（`hrmt-lab/zmk-raw-hid` の `custom/raw-hid-custom` に push 済み）。

- `7ed7fab fix: answer GATT reads on the BLE Raw HID report characteristics`
- `8ce8f99 fix: stop the USB Raw HID send path from spinning without an endpoint`

### 1. `src/hog.c` — Report characteristic の read コールバックが NULL

送受信 2 本の Report characteristic が `BT_GATT_CHRC_READ` を宣言しているのに
read コールバックを持たない。Zephyr の `bt_gatt_check_perm()` は `attr->read` が
NULL なら `BT_ATT_ERR_READ_NOT_PERMITTED` を返す（`gatt.c`）。
ZMK 本体の `zmk/app/src/hog.c` は全 report characteristic に実 read コールバック
(`read_hids_input_report` 等) を持っており、そちらに揃えるのが正しい。

対処（実施済み）: 直近の送受信内容をキャッシュして返す read コールバックを 2 本追加した。
属性の並びは変わらないので `notify_params.attr = &raw_hog_svc.attrs[5]` はそのまま。

### 2. `src/usb_hid.c` — `HID_1` 不在時の NULL 参照

`raw_hid_init()` は `device_get_binding(CONFIG_RAW_HID_DEVICE)` が NULL なら
`-EINVAL` を返して終わるが、`raw_usb_send_work_handler()` は `raw_hid_dev` を
検査せず `hid_int_ep_write()` に渡す。`raw_hid_adapter` シールドを付けずに
`CONFIG_RAW_HID=y` だけ有効にしたビルド（`USB_HID_DEVICE_COUNT=1`）で
USB HID が ready になると NULL 参照になる。

対処（実施済み）: work handler の先頭で `raw_hid_dev == NULL` ならキューを捨てて return する。

### 3. `src/usb_hid.c` — USB 未接続時に破棄パスがない

`zmk_usb_is_hid_ready()` が false の間、同じ packet を 10ms 間隔で無期限に
再スケジュールし続ける（`report_pending` が下りない）。BLE 運用で USB 未接続だと
system workqueue 上に 100Hz の work が残り続ける。実害は確認できなかったが、
一定回数で諦めて捨てる方が素直。

対処（実施済み）: 1 秒でリトライを打ち切って破棄する。加えて `send_report()` の入口で
`zmk_usb_is_hid_ready()` を見て、送り先が無いときはキューに積まない。
この判定は ZMK 本体が `endpoints.c` の `is_usb_ready()` で使っているものと同じ。
