# pomera-bluez-patches

BlueZ patches for Linux 3.10, to make Bluetooth LE HID devices (HID over GATT) work on Pomera DM250 / DM200 running Debian.

ポメラ DM250 / DM200 の Debian（カーネル 3.10）で、Bluetooth LE のマウスなど（HID over GATT）を使えるようにするための BlueZ のパッチです。DM200 より前の機種は Linux を動かせないので、対象外です。

新しい BlueZ は、カーネル 3.10 にはない機能を前提にしているところがあり、そのままではマウスが繋がっても動きません。原因の詳しい説明は、こちらの記事に書きました。

- [ポメラDM250環境の変化(2) 周辺機器編](https://zenn.dev/kay1974/articles/aa3143bed8f17d)

## パッチ

| パッチ | 直すこと | 症状 |
|---|---|---|
| `0001-shared-att-treat-fixed-ATT-CID-as-LE.patch` | LE の接続を BR/EDR と誤判定しないようにする | Report Map が 22 バイトで切れ、`hid-generic: unbalanced collection at end of report description` |
| `0002-hog-lib-use-legacy-UHID_CREATE.patch` | `UHID_CREATE2`（3.19 以降）でなく旧来の `UHID_CREATE` を使う | `bt_uhid_send: Operation not supported` |
| `0003-hog-lib-numbered-reports-without-dev_flags.patch` | `UHID_START` に `dev_flags`（3.11 以降）が無いとき、Report ID から番号付きレポートを判断する | 通知は届くのに `/dev/input/eventN` に何も出ない |

どのパッチも単独で当てられます。

### BlueZ の版によって、要るパッチが違います

| BlueZ | Debian | 要るパッチ | 確認方法 |
|---|---|---|---|
| 5.66（`5.66-1+deb12u2`） | bookworm | 0001・0002・0003 | DM250 の実機で動作確認 |
| 5.55（`5.55-3.1+deb11u1`） | bullseye | 0001 のみ | 下記 |

BlueZ 5.55 の `hog-lib.c` は、もともと旧来の `UHID_CREATE` を使っていて、`dev_flags` の処理もありません。そのため 0002 と 0003 は不要で、当てようとしても当たりません（`patch --dry-run` で確認）。

DM200 の bullseye では、tam さんが 0001 に相当する修正だけで BT マウスを動かしています。

- [DM200 の bullseye で Bluetooth](https://zenn.dev/tam/articles/article20260927-dm200-bluetooth)（tam さん）

## 動作確認した環境

- ポメラ DM250
- カーネル 3.10.0（[ichinomoto さんの配布物](https://www.ekesete.net/log/?p=9504)）
- Debian 12 bookworm、BlueZ `5.66-1+deb12u2`
- マウス: Logicool M750（Bluetooth LE で接続）

## パッチ以外に必要なこと

パッチだけでは足りません。次の3つも必要でした。

1. **`/etc/bluetooth/main.conf` の `[GATT]` に `Channels = 1` を書く**
   EATT（Bluetooth 5.2 の機能）の listen に失敗して、`Failed to create GATT database` でアダプタの登録が止まるのを避けます。
2. **ペアリングの前に `bluetoothctl pairable on`**
   これが無いと、ペアリングに成功しても `Bonded: no` になり、切断するたびに鍵が消えます。`bluetoothctl info <アドレス>` で `Bonded: yes` になっているか確かめてください。
3. **自動再接続はできません**
   カーネル 3.10 の管理インターフェース（mgmt 1.3）には、自動接続に必要なコマンドがありません。ログに `hci0 Load Connection Parameters failed: Unknown Command (0x01)` が出ます。マウスを動かしながら `bluetoothctl connect <アドレス>` でこちらから繋いでください。

## ビルドと導入（bookworm の場合）

DM250 の上でビルドします。ソースの取得には、`/etc/apt/sources.list` に `deb-src` の行が必要です。

### 1. ソースを取ってきて展開する

```sh
mkdir -p ~/build && cd ~/build
apt-get source --download-only bluez
dpkg-source --no-check -x bluez_5.66-1+deb12u2.dsc
sudo apt-get build-dep bluez
```

`--no-check` は、展開の前の署名検証を飛ばすオプションです。DM250 では検証に使う `gpgv` が異常終了（signal 6）して、`dpkg-source -x` がそのままでは失敗しました。`apt-get source` も展開の段階で同じ検証をするので、取得（`--download-only`）と展開を分けています。

### 2. パッチを置く

```sh
cd bluez-5.66
cp /path/to/pomera-bluez-patches/debian/patches/*.patch debian/patches/
ls /path/to/pomera-bluez-patches/debian/patches/ >> debian/patches/series
```

### 3. 版番号に `local1` を付ける

`debian/changelog` の先頭に、次のような項目を足します。版番号を元より大きくしておかないと、apt が次の更新のときに黙って Debian の版へ戻してしまいます。

```
bluez (5.66-1+deb12u2local1) bookworm; urgency=medium

  * Local build for kernel 3.10 (Pomera DM250).

 -- Your Name <you@example.com>  Mon, 28 Sep 2026 12:00:00 +0900
```

`devscripts` を入れているなら、`dch --local local` でも同じことができます。

### 4. ビルドして入れる

```sh
dpkg-buildpackage -us -uc -b -j2
cd ..
sudo dpkg -i bluez_*local1_armhf.deb libbluetooth3_*local1_armhf.deb bluez-obexd_*local1_armhf.deb
sudo apt-mark hold bluez libbluetooth3 bluez-obexd
```

DM250 では、差分ビルド（2回目以降）で約11分かかりました。初回はもっとかかります。

`libbluetooth-dev` を入れている場合は、同じビルドでできた `libbluetooth-dev_*local1_armhf.deb` も一緒に入れて、hold してください。`libbluetooth-dev` は `libbluetooth3` と完全に同じ版を要求するので、版がそろっていないと依存関係のエラーで `apt` の更新が全部止まります。

```
libbluetooth-dev : Depends: libbluetooth3 (= 5.66-1+deb12u2) but 5.66-1+deb12u2local1 is installed
E: Unmet dependencies. Try 'apt --fix-broken install' with no packages (or specify a solution).
```

入っているかどうかは `dpkg -l libbluetooth-dev` で確かめられます。このエラーが出ても `apt --fix-broken install` は打たないでください。`libbluetooth-dev` を消すか、`libbluetooth3` をパッチの無い版に戻そうとします。

`apt-mark hold` をしておかないと、Debian のセキュリティ更新（`deb12u3` など）が来たときに、パッチの無い版で上書きされます。更新が出たら、新しい版のソースで同じ手順をやり直してください。

## ライセンス

各パッチは、それが変更する BlueZ のファイルのライセンスに従います。

- `0001`（`src/shared/att.c`）: LGPL-2.1-or-later
- `0002`・`0003`（`profiles/input/hog-lib.c`）: GPL-2.0-or-later

## 注意

無保証です。動作を確認したのは、上に書いた環境だけです。Issue はありがたく読みますが、対応はお約束できません。
