# 家庭用ファイルサーバ 仕様書

## 概要

古い Mac mini を再利用して、夫婦のファイルサーバ兼オンラインストレージを構築する。家の中でも外出先でも Nextcloud に集約してアクセスする構成とする (SMB は使わず、必要になれば後付けする)。

## ハードウェア

| 項目 | 内容 |
|---|---|
| 本体 | Mac mini (Late 2014) |
| メモリ | 16GB |
| ストレージ (OS / DB) | 121GB 内蔵 SSD |
| ストレージ (データ) | 1TB 内蔵 HDD (元 Fusion Drive を分離利用) |
| ストレージ (将来) | USB 外付け HDD/SSD を追加予定 |
| ネットワーク | 有線 LAN (必須) |

## ソフトウェア構成

| レイヤ | 採用 | 備考 |
|---|---|---|
| OS | OpenMediaVault (OMV) 8 | Debian 13 ベース |
| ファイル共有・同期 | Nextcloud | Docker コンテナで運用。夫婦 2 アカウントで共有フォルダを構成 |
| 写真・動画管理 | Immich | Docker コンテナで運用。スマートフォン自動バックアップ・顔認識・AI 検索。LAN 内のみ公開 (`immich.home.lan`) |
| コンテナ基盤 | Docker + docker compose | omv-extras の compose プラグイン経由 |
| VPN | WireGuard (Docker, `linuxserver/wireguard`) | Mac mini 上で稼働。ルータで UDP 51820 のみ開放。クライアント側は AGH を DNS として参照 (PEERDNS) |
| DDNS | DuckDNS | グローバル IP が動的なため、`/srv/appdata/duckdns/duck.sh` を `/etc/cron.d/duckdns` で 5 分おきに更新 |
| LAN 内 DNS | AdGuard Home (Docker) | `*.home.lan` を Mac mini に向ける。広告ブロックも兼ねる。ホスト OS の :53 と競合させないため `systemd-resolved` の `DNSStubListener=no` を設定済 |
| リバースプロキシ | Nginx Proxy Manager (Docker) | Nextcloud を https 化 (LAN 内 DNS + 自己署名)。OMV 管理画面は :80 から :8181 に退避済 |
| (将来) SMB/CIFS | 未使用 | 必要になれば後付け可能。`docs/03-shared-folders-smb.md` 参照 |

## 要件

### 機能要件

- 夫婦で共有するフォルダ (プライベート用 / 仕事用) と、個人フォルダ
- 写真・動画のバックアップ保管
- スマートフォン (iOS/Android) からの写真自動アップロード
- 家の中でも外出先でも同じフォルダにアクセスできること
- 外出先からのファイルアクセス (VPN 経由)

### 非機能要件

- データ容量: 当初 100〜200GB、将来的に拡張可能な構成とする
- 可用性: 家庭用途のためベストエフォート (SLA なし)
- セキュリティ: 外部に直接公開しない。VPN 経由のみアクセス許可
- 運用: CUI 操作を許容。GUI で済むものは GUI で

## クライアント

| 端末 | アクセス方式 |
|---|---|
| Mac | Nextcloud デスクトップクライアント (家では直接、外では WireGuard VPN 経由) |
| Windows | Nextcloud デスクトップクライアント (同上) |
| iOS / Android | Nextcloud 公式アプリ (写真自動アップロード / 外出先は VPN 経由) |

## ネットワーク

- LAN 内は固定 IP を割り当て (ルータの DHCP 予約で対応)
- VPN は WireGuard を使用し、外出先端末からのみ接続を許可
- インターネット側へのポート開放は VPN ポート (UDP 51820 等) のみ

## ストレージ構成

### 現時点

- **SSD (121GB)** = `/dev/sdb` (`/` にマウント): OS / Docker / 各コンテナ設定 (MariaDB, Redis, NPM, WireGuard, AGH, DuckDNS) / Nextcloud の PHP 本体。compose は `/srv/compose/<svc>/`、各コンテナの永続データは `/srv/appdata/<svc>/`
- **HDD (1TB)** = `/dev/sda1` (`/srv/dev-disk-by-uuid-3a9a7624-afbb-4993-a1dd-38e2bd1546f5/` にマウント、以後 `<HDD_ROOT>`): Nextcloud の全ユーザーデータ (`<HDD_ROOT>/appdata/nextcloud/data/`)、バックアップ宛先 (`<HDD_ROOT>/shares/backups/`)
- **SSD 側データはクロスドライブ保護**される (SSD → HDD にバックアップされるため、SSD 故障で完全消失しない)
- **HDD 側のユーザーデータは同一ドライブ内バックアップ** (HDD 故障で両方失う)。誤削除・ランサムウェア対策としての位置付け
- **Immich データ (HDD)** = `<HDD_ROOT>/appdata/immich/` (写真・動画アップロード)、**Immich DB (SSD)** = `/srv/appdata/immich-db/`

### 将来の拡張方針

- 外付けストレージ (USB HDD) を追加し、バックアップ宛先を外付けに切り替える
- 宛先パスの差し替えだけで済むよう、現時点のバックアップ設計でパスを変数化しておく
- 重要データ (写真・動画) はさらに別媒体 (別ドライブ / クラウド) への二次バックアップも検討
- RAID はバックアップの代替にならないため、別系統のバックアップは必須とする

## バックアップ方針

- Nextcloud データディレクトリおよび設定ファイル、DB (`mariadb-dump`)、Immich DB (`pg_dump`)、compose 一式を `/usr/local/sbin/homenas-backup.sh` (rsync) で定期コピー
- 当面の宛先は HDD 内 (`<HDD_ROOT>/shares/backups/`)、将来は外付けへ移行
- `--link-dest` によるハードリンク世代管理で日次スナップショットを取得 (世代保持: 14 日)
- 頻度: 日次 03:00 (`/etc/cron.d/homenas-backup`)。ログ: `/var/log/homenas-backup.log`
- 詳細は [docs/09-backup.md](docs/09-backup.md) 参照

## セキュリティ方針

- Nextcloud は原則 VPN 経由でのみアクセス
- OMV / Nextcloud の管理者パスワードは十分な強度のものを設定
- OS およびコンテナは定期的にアップデート
- 不要なサービス・ポートは開放しない

## 構築手順

概略は以下の通り。各ステップの詳細手順は `docs/` 配下にある。

0. [全体像と事前準備](docs/00-overview.md)
1. [OMV インストーラ USB 作成と Mac mini へのインストール](docs/01-install-omv.md)
2. [OMV 初期設定 (ネットワーク / ユーザー / SSH / アップデート)](docs/02-initial-config.md)
3. [HDD マウントとディレクトリ階層の設計](docs/03-shared-folders-smb.md)
4. [omv-extras 導入と Docker / compose プラグイン有効化](docs/04-omv-extras-docker.md)
5. [Nextcloud コンテナの構築](docs/05-nextcloud.md)
6. [AdGuard Home の導入 (LAN 内 DNS + 広告ブロック)](docs/06-adguard-home.md)
7. [Nginx Proxy Manager の導入と Nextcloud の https 化](docs/07-nginx-proxy-manager.md)
8. [WireGuard の導入とクライアント配布](docs/08-wireguard.md)
9. [バックアップ設定](docs/09-backup.md)
10. [各クライアントからの接続確認](docs/10-verification.md)
11. (オプション) [n8n で家庭用自動化ハブを立てる](docs/11-n8n.md)
12. [Immich の導入 (写真・動画管理)](docs/12-immich.md)

構築完了後の定常運用 (端末追加・アカウント追加・障害対応など) は [docs/operations.md](docs/operations.md) にまとめてある。

## 運用上のメモ

- Mac mini 2014 以前は起動時の `Option` キーでブートメディア選択可
- Wi-Fi は使用せず有線 LAN で接続する
- 初期状態の OMV ログインは `admin` / `openmediavault` (構築後ただちに変更)

## 未決事項

- 外付けストレージの追加時期と具体的な機種・容量
- **外部公開方式の方針転換**: 現状は VPN (WireGuard) 経由のみだが、「VPN OFF で家でも外でも同じ FQDN で繋がる」運用に移行するか検討中。候補は Cloudflare Tunnel (推奨) または古典的ポート開放 (Let's Encrypt)
- **AFFiNE (Notion 風セルフホストツール) 本採用判断**: 試用中。常用するなら `docs/13-affine.md` として手順化予定
- バックアップの自動通知 (失敗時に Slack / メール) と日次ヘルスチェックのスクリプト化

## 変更履歴

| 日付 | 内容 |
|---|---|
| 2026-04-19 | 初版作成 |
| 2026-04-19 | 未決事項 (Mac mini モデル / VPN 位置 / バックアップ方式 / リバースプロキシ要否) を確定し、`docs/` にステップ別手順を追加 |
| 2026-04-19 | 実機のストレージが Fusion Drive 由来の SSD 121GB + HDD 1TB 構成と判明。OS/DB を SSD、ユーザーデータを HDD に分ける構成へ変更 |
| 2026-04-19 | ファイル共有を SMB + Nextcloud の併用から Nextcloud 単独に変更 (夫婦 2 人運用の整理)。SMB は将来必要になった場合の後付けに位置付け |
| 2026-04-19 | 自宅ルータ (光BBユニット E-WMTA2.3) に DNS 機能がないため、LAN 内 DNS を AdGuard Home (Docker) で立てる構成に変更。手順を §6 として追加し、以降の章を繰り下げ |
| 2026-04-24 | Mac mini 上で OMV 8 / Nextcloud / NPM / AGH / WireGuard / DuckDNS の構築完了。日次バックアップ (cron 03:00) 稼働開始 |
| 2026-04-24 | WireGuard 章 (§8) に DuckDNS (DDNS) 手順を追加。グローバル IP 変動への追従を確実化。`crontab -e` は OMV の Salt 管理と競合するため `/etc/cron.d/duckdns` 方式に変更 |
| 2026-04-25 | クライアント側の DNS バイパス無効化手順を `docs/operations.md` §4 に追加 (Android プライベート DNS / Chrome セキュア DNS が AGH を素通りする問題への対処) |
| 2026-04-25 | DNS の特定ドメインだけ繋がらない問題の切り分け手順を `docs/operations.md` §8-A に追加 (AGH 上流 DNS の Cloudflare 1.1.1.1 追加対応含む) |
| 2026-05-11 | バックアップ運用の重大バグを 2 件修正: ① `mysqldump` → `mariadb-dump` (MariaDB 11.x で `mysqldump` 廃止対応)、② rsync の `--exclude='shares/backups/'` が anchor 不足で機能せず、バックアップが自身を再帰コピーして HDD を 100% まで埋め尽くす事故が発生。`--exclude='/backups/'` に修正、`docs/09-backup.md` のトラブルシュート節と本 README 双方に再発防止メモを追記 |
| 2026-06-13 | Immich 導入。Docker Compose で immich-server / immich-machine-learning / immich_redis / immich_postgres を追加。写真データを HDD (`<HDD_ROOT>/appdata/immich/`)、DB を SSD (`/srv/appdata/immich-db/`) に配置。NPM で `immich.home.lan` → port 2283 に転送 (LAN 内のみ)。手順は `docs/12-immich.md` |
| 2026-06-13 | SSH (Termius / macOS) 接続時に日本語が入力・表示できない問題に対処。クライアントが送る不正な `LC_CTYPE=UTF-8` を `~/.bashrc` / `~/.inputrc` で UTF-8 ロケールに上書き。手順とトラブルシュートを `docs/operations.md` §9 に追加 |