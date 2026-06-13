# このリポジトリについて

家庭用ファイルサーバ (Mac mini Late 2014 + OpenMediaVault 8 / Debian 13) の仕様書と運用手順。Docker で Nextcloud / MariaDB / Redis / Nginx Proxy Manager / WireGuard / AdGuard Home / DuckDNS を回している。夫婦 2 人運用。

## まず読むファイル

1. **`README.md`** — ハード構成・採用技術・ネットワーク・ストレージ・バックアップ・未決事項・変更履歴
2. **`docs/operations.md`** — 構築完了後の運用ノウハウ (端末追加 / DNS バイパス対処 / コンテナ更新 / バックアップ確認 / 特定サイト疎通切り分け / トラブル対応)

`docs/00-` 〜 `docs/11-` は構築時のステップ手順。すでに構築済みなので、特定の章を見直す時だけ参照。

## サーバ実機上で押さえるべき不文律

- **`/srv/compose/` は root:root 所有のまま使う**。OMV Compose プラグインが Apply 時に root:root を再適用するので chown しても無駄。compose 操作は `sudo -i` でルートシェルに入るか、OMV Web UI の Services → Compose 経由で行う
- **OMV 管理画面のポートは :8181** (:80 / :443 は NPM に明け渡し済み)
- **ホスト OS の `systemd-resolved` は `DNSStubListener=no`** に設定済 (AGH コンテナと :53 を競合させないため)。`/etc/resolv.conf` は `/run/systemd/resolve/resolv.conf` にシンボリックリンクされている
- **DuckDNS の cron は `/etc/cron.d/duckdns`** (OMV は Salt が root の crontab を管理しているため、`crontab -e` は使わず `/etc/cron.d/` に独立ファイルで置く慣習)

## 重要なパス

| 用途 | パス |
|---|---|
| compose ファイル群 | `/srv/compose/<svc>/` (root:root) |
| 各コンテナの永続データ (SSD 側) | `/srv/appdata/<svc>/` |
| Nextcloud のユーザーデータ (HDD) | `/srv/dev-disk-by-uuid-3a9a7624-afbb-4993-a1dd-38e2bd1546f5/appdata/nextcloud/data/` |
| バックアップ宛先 (HDD) | `/srv/dev-disk-by-uuid-3a9a7624-afbb-4993-a1dd-38e2bd1546f5/shares/backups/` |
| バックアップスクリプト | `/usr/local/sbin/homenas-backup.sh` (毎日 03:00 / cron) |
| バックアップログ | `/var/log/homenas-backup.log` |
| DuckDNS 更新スクリプト | `/srv/appdata/duckdns/duck.sh` (`/etc/cron.d/duckdns` で 5 分おき) |

HDD マウントパスは長いので、シェルで作業する時は `HDD_ROOT=/srv/dev-disk-by-uuid-3a9a7624-afbb-4993-a1dd-38e2bd1546f5` を環境変数に入れると楽。

## よく使うコマンド

```bash
# 全コンテナ状態
sudo docker ps --format 'table {{.Names}}\t{{.Status}}'

# あるコンテナのログ
sudo docker logs -f <container名>

# Nextcloud occ コマンド
sudo docker exec -u www-data nextcloud-app php occ <subcommand>

# バックアップを手動で 1 回回す
sudo /usr/local/sbin/homenas-backup.sh 2>&1 | tee /tmp/backup.log

# AGH の最新クエリ・ログを見る
sudo docker exec adguardhome tail -n 50 /opt/adguardhome/work/data/querylog.json
```

## 過去の事故から学んだ注意点

- **rsync の `--exclude` はソースルート基準で書く**。`--exclude='shares/backups/'` のような prefix 付きパターンはソースが末尾 `/` 付きの場合一致せず、バックアップが自身を再帰コピーして HDD を満杯にする。正しくは `--exclude='/backups/'` (anchor 付き)
- **MariaDB 11.x 以降は `mysqldump` 廃止**。`mariadb-dump` を使う
- **Android のプライベート DNS と Chrome のセキュア DNS は AGH を素通りする**。Wi-Fi の DNS を AGH に向けても効かないことがあるので両方 OFF を確認 (`docs/operations.md` §4)

## 編集する時の方針

- ドキュメントは Markdown。修正は最小差分で
- 手順を増やす時は既存の章立て (§0〜§11 / operations.md) のどこに属するかを先に決める
- 設定変更で `docs/` と現状が乖離したら README 変更履歴に 1 行残す
