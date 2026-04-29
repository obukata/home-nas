# 0. 全体像と事前準備

前: (なし / [README](../README.md))

## 目的

本ドキュメント群を使って、Mac mini Late 2014 上に OpenMediaVault (OMV) 8 + Nextcloud + Nginx Proxy Manager + WireGuard を構築する。
このステップではゴールの構成像を共有し、実作業に入る前に必要な物と設定値を先に準備する。

ファイル共有は **Nextcloud に集約** する (SMB は使わない)。2 人運用では Nextcloud の Desktop Client / モバイルアプリで家でも外でも同じように使える方がシンプル。将来 SMB を追加したくなれば [docs/03-shared-folders-smb.md](03-shared-folders-smb.md) の指示で後付けできる。

## ゴールの構成

サービス配置:

```
[家族の端末]          ┐
  - Mac (LAN/VPN)     │
  - Windows (LAN/VPN) ├─── (LAN / VPN 経由)
  - Android (VPN)     │
                      │
      ┌───────────────┴──────────────────────┐
      │ ルータ                               │
      │  - UDP 51820 をポートフォワード      │
      │    → Mac mini の固定 IP              │
      │  - LAN 内 DNS で nextcloud.home.lan  │
      │    を Mac mini の IP に解決          │
      └───────────────┬──────────────────────┘
                      │ 有線 LAN (固定 IP)
      ┌───────────────┴──────────────────────┐
      │ Mac mini Late 2014 (OMV 8)           │
      │                                      │
      │  OS: Debian 13 + OMV 8               │
      │                                      │
      │  [Docker]                            │
      │   ├── Nextcloud (+ MariaDB + Redis)  │
      │   └── Nginx Proxy Manager            │
      │        └─ https://nextcloud.home.lan │
      │           → nextcloud:80 (内部)      │
      │                                      │
      │  [WireGuard] (Docker)                │
      │   └─ wg0 (10.10.0.0/24)              │
      │                                      │
      │  [内蔵 121GB SSD] <SSD_ROOT>=/srv    │
      │   ├── OS                             │
      │   ├── /srv/appdata/                  │
      │   │   ├── mariadb, redis             │
      │   │   ├── npm, wireguard             │
      │   │   └── nextcloud/html             │
      │   ├── /srv/compose/                  │
      │   └── /srv/docker/ (Docker data-root)│
      │                                      │
      │  [内蔵 1TB HDD] <HDD_ROOT>           │
      │   ├── appdata/nextcloud/data/        │
      │   │    (夫婦2アカウントのデータ)     │
      │   └── shares/backups/                │
      │        (rsync snapshot 出力先)       │
      └──────────────────────────────────────┘
```

## 事前準備

### 物理

- [ ] Mac mini Late 2014 本体 / 電源ケーブル
- [ ] 8GB 以上の USB メモリ (OMV インストーラ用、中身は消えてよいもの)
- [ ] 有線 LAN ケーブル (Cat5e 以上)
- [ ] USB 有線キーボード (インストール時のみ使用)
- [ ] マウスまたはトラックパッド (同上、なくても可)
- [ ] ディスプレイ (Mac mini Late 2014 は HDMI と Thunderbolt 2 の両方あり。HDMI 推奨)
- [ ] 作業用 Mac (インストーラ USB の作成に使う)

### ネットワーク

- [ ] ルータの管理画面にログインできる (DHCP 予約・ポートフォワード・LAN 内 DNS 設定に必要)
- [ ] Mac mini を設置する位置まで LAN 配線が届く
- [ ] ルータが UDP ポートフォワードに対応している

### 決めておく値 (本構築で使う)

| 項目 | 値 |
|---|---|
| ホスト名 | `homenas` |
| OMV 固定 IP | `<OMV_IP>` (例: 192.168.1.50、ルータの LAN に合わせて) |
| 管理ユーザー名 (SSH) | `<ADMIN_USER>` (例: `obukata`) |
| Nextcloud の内部 FQDN | `nextcloud.home.lan` |
| NPM の管理画面ポート | `81` |
| WireGuard のポート | `UDP 51820` |
| WireGuard の VPN 側サブネット | `10.10.0.0/24` |
| `<SSD_ROOT>` | `/srv` (OS が乗っている 121GB SSD のパス) |
| `<HDD_ROOT>` | `/srv/dev-disk-by-uuid-XXXX` (HDD のマウントポイント、実際の UUID はインストール後に確認) |

後続の手順書ではこれらの値を `<OMV_IP>` や `<HDD_ROOT>` のような形で参照する。最初に置換表を自分の値で埋めておくと楽。

### ディスク役割分担の方針

- **SSD (`<SSD_ROOT>`)**: OS / Docker イメージ / データベース (MariaDB) / Redis / 各コンテナの小さい設定ファイル / Nextcloud の PHP 本体
- **HDD (`<HDD_ROOT>`)**: Nextcloud の全ユーザーデータ / バックアップ宛先

## 確認

このステップでは作業は発生しない。事前準備のチェックリストが全て埋まっていれば次へ進む。

## 次へ

次: [1. OMV インストーラ USB 作成と Mac mini へのインストール](01-install-omv.md)
