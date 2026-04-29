# 1. OMV インストーラ USB 作成と Mac mini へのインストール

前: [0. 全体像と事前準備](00-overview.md)

## 目的

OpenMediaVault 8 (Debian 13 ベース) の ISO を USB に焼き、Mac mini Late 2014 の **内蔵 121GB SSD** に OMV をクリーンインストールする。1TB HDD はデータ用としてそのまま残し、後のステップでマウントする。

## 前提

- 事前準備 ([00-overview.md](00-overview.md)) が完了している
- 作業用 Mac で SSD 相当の USB が作業中に全消去されることを理解している

## 手順

### 1-1. OMV 8 の ISO をダウンロード

OMV 公式 (SourceForge) から最新の OMV 8 インストーライメージを取得する。

- <https://www.openmediavault.org/> → Download ページ
- `openmediavault_8.x.x-amd64.iso` を選ぶ

ダウンロード後、SHA256 を併記のチェックサムと照合する (任意だが推奨)。

```bash
shasum -a 256 ~/Downloads/openmediavault_8.*.iso
```

### 1-2. USB に書き込み

作業用 Mac に USB を挿し、デバイス名を確認する。

```bash
diskutil list
```

対象 USB の識別子 (例: `disk4`) を確認したら、アンマウントして `dd` で書き込む。`/dev/rdiskN` (`r` 付き raw デバイス) を使うと速い。

**書き込み先を間違えるとシステムドライブを破壊するので、識別子を二度確認する。**

```bash
# 例: disk4 が USB の場合
diskutil unmountDisk /dev/disk9

# 書き込み (iso のパスと rdisk 番号は自分の値に差し替え)
sudo dd if=/Users/obukata/Downloads/openmediavault_8.0.7-amd64.iso of=/dev/rdisk9 bs=1m status=progress
```

GUI で済ませたい場合は [balenaEtcher](https://etcher.balena.io/) を使う。

完了したら取り出す。

```bash
diskutil eject /dev/disk9
```

### 1-3. Mac mini を USB ブート

1. Mac mini に USB・キーボード・ディスプレイ・LAN ケーブルを接続し、電源を入れる
2. 起動音直後から `Option (⌥)` を押し続けてブートメニューを表示
3. USB (多くは黄色いアイコンで "EFI Boot" と表示される) を選択
4. Debian/OMV のインストーラが起動する

### 1-4. インストーラ設定

Debian 標準のテキストインストーラが立ち上がる。以下を設定。

| 項目 | 設定値 |
|---|---|
| 言語 | English (ログ・エラーメッセージが英語になり、検索しやすい) |
| 場所 | Other → Asia → Japan |
| ロケール | `en_US.UTF-8` (TTY で文字化けせず、ログ検索にも有利。日本語ファイル名は UTF-8 で問題なく扱える) |
| キーボード | Japanese (必要に応じて) |
| ホスト名 | `homenas` |
| ドメイン名 | 空欄または `home.lan` |
| root パスワード | 強固なもの (忘れないように) |
| ディスク | **内蔵 121GB SSD** (`APPLE SSD` 等と表示される小さい方) を選択し **全消去**。1TB の HDD は **選択しない** (次ステップでデータ用として個別マウント) |
| パーティション | ガイドで SSD 全体使用、LVM なしの標準で良い |
| ネットワーク | 有線 LAN (DHCP) でまず取得しておく (固定 IP は次ステップ) |

インストール完了後、USB を抜いて再起動する。

### 1-5. 起動確認

- Mac mini のコンソールにログインプロンプトが出る
- 別の端末から `ping homenas.local` または `ping <DHCP で割り当てられた IP>` が通る

## 確認

- [ ] コンソールにログインプロンプトが出て、root でログインできる
- [ ] 同じ LAN の Mac/PC から Mac mini に ping が通る
- [ ] ブラウザで `http://<DHCP で割り当てられた IP>/` を開くと OMV のログイン画面が表示される

## トラブルシュート

- **`Option` キーで USB が出ない**: USB が UEFI 非対応の焼き方になっている可能性。`dd` または balenaEtcher で焼き直す。Mac mini Late 2014 は UEFI 起動なので、Legacy BIOS 専用のイメージだと出ない
- **Wi-Fi を使ってしまった**: OMV では Wi-Fi を使わない。有線 LAN で接続し直す
- **インストール後に画面が出ない**: Mac mini 特有の EDID 問題で、HDMI → DP 変換や別モニタで解決する場合あり。以降は SSH 運用でコンソール不要

## 次へ

次: [2. OMV 初期設定](02-initial-config.md)
